# Dynamic Versioned Driver Architecture in C#

## Overview

For a C# application that needs to:

- Load drivers dynamically at runtime
- Add a new driver without recompiling the application
- Support multiple versions of the same driver simultaneously
- Configure different clients to use different driver versions
- Isolate driver dependencies

A good architecture is:

```text
                    ┌────────────────────────┐
                    │ Driver.Abstractions    │
                    │                        │
                    │ IDriver                │
                    │ IDevice                │
                    │ DriverInfo             │
                    │ DriverConfiguration    │
                    └───────────┬────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
       AcmeDriver 1.0     AcmeDriver 2.0     SiemensDriver 1.0
              │                 │                 │
              └─────────────────┼─────────────────┘
                                │
                       AssemblyLoadContext
                                │
                                ▼
                    ┌────────────────────────┐
                    │    Driver Registry     │
                    │                        │
                    │ acme    → 1.0         │
                    │ acme    → 2.0         │
                    │ siemens → 1.0         │
                    └───────────┬────────────┘
                                │
                         Device Factory
                                │
                ┌───────────────┴───────────────┐
                │                               │
             Client A                        Client B
                │                               │
             acme 1.0                        acme 2.0
```

The key distinction is:

1. **Dynamic plugin loading** → `AssemblyLoadContext`
2. **Selecting the correct driver for each client** → a registry keyed by `(driver name, version)`

---

# 1. Application-facing interface

Your application should not know about concrete drivers.

```csharp
public interface IDevice
{
    Task<string> GetStatusAsync();
    Task SendAsync(string command);
}
```

Business logic should depend only on `IDevice`.

---

# 2. Driver contract

Create a separate assembly, for example:

```text
MyCompany.Driver.Abstractions.dll
```

Put the plugin contracts in this assembly:

```csharp
public interface IDriver
{
    string Name { get; }

    Version Version { get; }

    IDevice CreateDevice(DriverConfiguration configuration);
}
```

Configuration can be:

```csharp
public sealed class DriverConfiguration
{
    public required string ConnectionString { get; init; }

    public Dictionary<string, string> Options { get; init; } = new();
}
```

A driver implementation could look like:

```csharp
public sealed class AcmeDriverV1 : IDriver
{
    public string Name => "acme";

    public Version Version => new(1, 0);

    public IDevice CreateDevice(DriverConfiguration configuration)
    {
        return new AcmeDeviceV1(configuration);
    }
}
```

A newer version:

```csharp
public sealed class AcmeDriverV2 : IDriver
{
    public string Name => "acme";

    public Version Version => new(2, 0);

    public IDevice CreateDevice(DriverConfiguration configuration)
    {
        return new AcmeDeviceV2(configuration);
    }
}
```

---

# 3. Driver registry

The registry keeps multiple versions of drivers loaded simultaneously.

```csharp
public readonly record struct DriverKey(
    string Name,
    Version Version);
```

Interface:

```csharp
public interface IDriverRegistry
{
    void Register(IDriver driver);

    IDriver Get(string name, Version version);

    IReadOnlyCollection<IDriver> GetAll(string name);
}
```

Implementation:

```csharp
public sealed class DriverRegistry : IDriverRegistry
{
    private readonly ConcurrentDictionary<DriverKey, IDriver> _drivers = new();

    public void Register(IDriver driver)
    {
        var key = new DriverKey(driver.Name, driver.Version);

        if (!_drivers.TryAdd(key, driver))
        {
            throw new InvalidOperationException(
                $"Driver '{driver.Name}' version '{driver.Version}' is already registered.");
        }
    }

    public IDriver Get(string name, Version version)
    {
        var key = new DriverKey(name, version);

        if (!_drivers.TryGetValue(key, out var driver))
        {
            throw new InvalidOperationException(
                $"Driver '{name}' version '{version}' is not loaded.");
        }

        return driver;
    }

    public IReadOnlyCollection<IDriver> GetAll(string name)
    {
        return _drivers
            .Where(x => x.Key.Name.Equals(name, StringComparison.OrdinalIgnoreCase))
            .Select(x => x.Value)
            .ToArray();
    }
}
```

The registry can contain:

```text
acme     1.0
acme     2.0
siemens  1.0
abb      3.2
```

at the same time.

---

# 4. Configure each client with a driver version

Each client can explicitly specify which driver and version it needs:

```csharp
public sealed class ClientConfiguration
{
    public required string ClientId { get; init; }

    public required string DriverName { get; init; }

    public required Version DriverVersion { get; init; }

    public required DriverConfiguration DriverConfiguration { get; init; }
}
```

Example configuration:

```json
{
  "clients": [
    {
      "clientId": "client-a",
      "driverName": "acme",
      "driverVersion": "1.0",
      "driverConfiguration": {
        "connectionString": "..."
      }
    },
    {
      "clientId": "client-b",
      "driverName": "acme",
      "driverVersion": "2.0",
      "driverConfiguration": {
        "connectionString": "..."
      }
    }
  ]
}
```

Therefore:

```text
Client A → Acme 1.0
Client B → Acme 2.0
```

---

# 5. Device factory

Use a factory to hide the registry from the rest of the application:

```csharp
public interface IDeviceFactory
{
    IDevice Create(ClientConfiguration configuration);
}
```

Implementation:

```csharp
public sealed class DeviceFactory : IDeviceFactory
{
    private readonly IDriverRegistry _registry;

    public DeviceFactory(IDriverRegistry registry)
    {
        _registry = registry;
    }

    public IDevice Create(ClientConfiguration configuration)
    {
        var driver = _registry.Get(
            configuration.DriverName,
            configuration.DriverVersion);

        return driver.CreateDevice(
            configuration.DriverConfiguration);
    }
}
```

Application code becomes:

```csharp
var device = deviceFactory.Create(clientConfiguration);

await device.SendAsync("START");
```

The application does not need to know which concrete driver is being used.

---

# 6. Dynamic driver loading

If you want to drop a DLL into a directory while the application is running, use .NET's `AssemblyLoadContext`.

A basic folder structure is:

```text
MyApplication/
    Drivers/
        AcmeDriver/
            1.0.0/
                AcmeDriver.dll
            2.0.0/
                AcmeDriver.dll

        SiemensDriver/
            1.0.0/
                SiemensDriver.dll
```

A simple loader can discover implementations of `IDriver`:

```csharp
public sealed class DriverLoader
{
    private readonly IDriverRegistry _registry;

    public DriverLoader(IDriverRegistry registry)
    {
        _registry = registry;
    }

    public void Load(string path)
    {
        var assembly = AssemblyLoadContext.Default.LoadFromAssemblyPath(
            Path.GetFullPath(path));

        var driverTypes = assembly
            .GetTypes()
            .Where(t =>
                typeof(IDriver).IsAssignableFrom(t) &&
                !t.IsAbstract &&
                !t.IsInterface);

        foreach (var type in driverTypes)
        {
            var driver = (IDriver)Activator.CreateInstance(type)!;

            _registry.Register(driver);
        }
    }
}
```

Then:

```csharp
loader.Load("./Drivers/AcmeDriver/1.0.0/AcmeDriver.dll");
```

However, this default-context approach is **not enough when multiple versions need to coexist with potentially different dependencies**.

---

# 7. Use AssemblyLoadContext per driver version

If you want:

```text
Acme 1.0
Acme 2.0
```

loaded at the same time, create a separate `AssemblyLoadContext` for each plugin/version.

```csharp
public sealed class DriverLoadContext : AssemblyLoadContext
{
    public DriverLoadContext()
        : base(isCollectible: true)
    {
    }

    protected override Assembly? Load(
        AssemblyName assemblyName)
    {
        return null;
    }
}
```

Then load a driver using its own context:

```csharp
var context = new DriverLoadContext();

var assembly = context.LoadFromAssemblyPath(
    Path.GetFullPath(path));
```

Conceptually:

```text
DriverLoadContext #1
    AcmeDriver 1.0
        ↓
    AcmeDriverV1

DriverLoadContext #2
    AcmeDriver 2.0
        ↓
    AcmeDriverV2
```

This is especially useful when different versions have different dependencies.

For example:

```text
Acme 1.0
    Newtonsoft.Json 12.x

Acme 2.0
    Newtonsoft.Json 13.x
```

The separate load contexts can isolate those dependencies.

---

# 8. Recommended folder structure

Yes, it is recommended to keep each driver version in a separate folder:

```text
Drivers/
├── Acme/
│   ├── 1.0.0/
│   │   ├── Acme.Driver.dll
│   │   └── SomeDependency.dll
│   │
│   └── 2.0.0/
│       ├── Acme.Driver.dll
│       └── SomeDependency.dll
│
└── Siemens/
    └── 1.0.0/
        ├── Siemens.Driver.dll
        └── SomeDependency.dll
```

A good convention is:

```text
Drivers/{driver-name}/{version}/
```

The directory layout itself is not a .NET requirement. You could technically use:

```text
Drivers/
    Acme-1.0.dll
    Acme-2.0.dll
```

What matters is that each plugin version gets the appropriate `AssemblyLoadContext` and dependency resolution.

The versioned directory structure makes installation, upgrades, rollback, and unloading much easier.

---

# 9. Share the abstractions assembly

This is very important.

The architecture should look like:

```text
MyApplication
       │
       ▼
Driver.Abstractions
```

and:

```text
AcmeDriver.dll ──────► Driver.Abstractions
SiemensDriver.dll ───► Driver.Abstractions
AbbDriver.dll ───────► Driver.Abstractions
```

The driver should **not** reference the application's main assembly.

Also, when using custom `AssemblyLoadContext`s, make sure `Driver.Abstractions.dll` is shared between the host and plugin contexts.

Otherwise you can accidentally end up with two different runtime versions of `IDriver`:

```text
Host:
    Driver.Abstractions.IDriver

Plugin:
    Driver.Abstractions.IDriver
```

Even though the names are identical, .NET can treat them as different types if they were loaded into different contexts.

The plugin should use the host's shared abstraction assembly.

---

# 10. Stateful drivers

If drivers maintain state such as:

- TCP connections
- sockets
- sessions
- authentication state
- device connections
- caches

do not necessarily treat the registered `IDriver` instance as the actual client/device instance.

Instead, think of the registry as holding the driver implementation/factory:

```text
Registry
   │
   ├── acme 1.0 → driver factory
   ├── acme 2.0 → driver factory
   └── siemens 1.0 → driver factory
```

And create separate device instances:

```text
Client A
   └── Device instance → Acme 1.0

Client B
   └── Device instance → Acme 2.0
```

This prevents Client A and Client B from accidentally sharing state.

You might eventually evolve the contract into:

```csharp
public interface IDriver
{
    DriverInfo Info { get; }

    IDevice CreateDevice(DriverConfiguration configuration);
}
```

with:

```csharp
public sealed record DriverInfo(
    string Name,
    Version Version,
    string DisplayName);
```

---

# 11. Why not just use Microsoft.Extensions.DependencyInjection?

You can and should still use Microsoft's DI system **inside the host and inside driver implementations where appropriate**.

However, I would not make the normal DI container responsible for selecting the driver version.

For example, avoid trying to model everything as:

```text
IDriver → AcmeDriver
```

because you actually have:

```text
IDriver
 ├── Acme 1.0
 ├── Acme 2.0
 ├── Siemens 1.0
 └── ...
```

The application needs to select based on:

```text
(driver name, driver version)
```

So an explicit registry/factory is easier to reason about:

```text
Client configuration
        │
        ▼
   DeviceFactory
        │
        ▼
   DriverRegistry
        │
        ▼
 (name + version)
        │
        ▼
    IDriver
        │
        ▼
     IDevice
```

DI can then be used for the application's infrastructure and for dependencies required by the driver.

---

# 12. Final recommended architecture

For your use case, I would use:

```text
                         MyApplication
                              │
                 ┌────────────┴────────────┐
                 │                         │
          DeviceFactory              DriverManager
                 │                         │
                 │                  Discovers DLLs
                 │                         │
                 ▼                         ▼
          DriverRegistry          AssemblyLoadContext
                 │                         │
       ┌─────────┼─────────┐               │
       │         │         │               │
     acme      acme     siemens            │
     1.0       2.0        1.0               │
       │         │         │               │
       └─────────┴─────────┴───────────────┘
                         │
                         ▼
                Driver.Abstractions
```

With files:

```text
Drivers/
├── Acme/
│   ├── 1.0.0/
│   │   └── Acme.Driver.dll
│   └── 2.0.0/
│       └── Acme.Driver.dll
│
└── Siemens/
    └── 1.0.0/
        └── Siemens.Driver.dll
```

And client configuration:

```text
Client A → acme 1.0.0
Client B → acme 2.0.0
Client C → siemens 1.0.0
```

This gives you:

- Runtime driver installation
- Multiple driver versions loaded simultaneously
- Client-specific driver selection
- Dependency isolation
- Ability to unload a driver version later
- No dependency on concrete drivers in business logic
- Clean separation between plugin loading and driver selection

## One caveat

A production implementation of `AssemblyLoadContext` should go beyond the minimal example above. In particular, you will want a custom dependency resolver using `AssemblyDependencyResolver`, careful sharing of `Driver.Abstractions`, lifecycle management, and a strategy for safely unloading old driver versions.

The core design, however, is:

```text
Versioned folder
      ↓
AssemblyLoadContext
      ↓
IDriver
      ↓
DriverRegistry(name + version)
      ↓
DeviceFactory
      ↓
Client-specific IDevice
```
