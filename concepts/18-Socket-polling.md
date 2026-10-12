# Architectural Pattern: Asynchronous Task Notification Using Socket.IO Rooms

## 1. Problem Statement & Context
When a user submits a long-running request to a backend server (e.g., heavy data export, AI model inference, video encoding, complex reports):
*   **The Problem:** Traditional HTTP request-response cycles fail because keeping the connection open risks timeouts. 
*   **The Anti-Pattern (Polling):** Having the client periodically check an endpoint (`GET /api/task/status`) wastes network bandwidth, spikes CPU load with infinite request spam, and introduces delivery lag.
*   **The Solution:** An event-driven architecture using **Socket.IO rooms** to push updates immediately upon completion over a single persistent WebSocket connection.

---

## 2. Architecture & System Flow
1. **Initiate Task (HTTP POST):** The client triggers the long-running task via a REST endpoint. The backend assigns a unique `taskId` and responds immediately with a `202 Accepted` status code.
2. **Subscribe (Socket.IO Room):** The frontend receives the `taskId` and immediately instructs the Socket.IO server to place its connection into a specific room named after that `taskId`.
3. **Async Processing:** The backend processes the job out-of-band (or offloads it to a background worker queue like BullMQ or Celery).
4. **Push Notification (Room Emit):** As soon as the task completes, the backend emits a completion event targeted directly to the isolated `taskId` room. The client receives the payload instantly, updates the UI, and leaves the room.

---

## 3. Production Code Implementation

### A. Backend: Node.js, Express, & Socket.IO Server (`server.js`)
```javascript
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

// 1. Socket.IO Room Lifecycle Connection
io.on('connection', (socket) => {
  console.log(`Client connected: ${socket.id}`);

  // Client subscribes exclusively to updates for a specific task
  socket.on('monitor_task', (taskId) => {
    socket.join(taskId);
    console.log(`Socket [${socket.id}] isolated into Room [${taskId}]`);
  });
});

// Mock Asynchronous Processing Pipeline
function processHeavyJobInBackground(taskId) {
  console.log(`[Worker] Executing long-running task for ID: ${taskId}...`);
  
  // Simulating a 5-second asynchronous processing workload
  setTimeout(() => {
    const resultData = {
      status: "COMPLETED",
      timestamp: new Date().toISOString(),
      payload: { 
        reportId: 9876, 
        downloadUrl: `/secure/exports/${taskId}.csv` 
      }
    };

    // 4. Targeted push notification ONLY to sockets waiting inside this room
    io.to(taskId).emit('task_completed', resultData);
    console.log(`[Notifier] Dispatched payload to Room [${taskId}]`);
  }, 5000); 
}

// 2. HTTP Endpoint to initiate the background work
app.post('/api/start-task', express.json(), (req, res) => {
  // Generate a distinct tracking identifier for the task
  const taskId = `task_${Date.now()}_${Math.random().toString(36).substr(2, 5)}`;
  
  // Hand off execution immediately without 'awaiting' it
  processHeavyJobInBackground(taskId);

  // Inform client the request is accepted and provide the tracking token
  res.status(202).json({ 
    success: true,
    taskId, 
    message: "Background processing initiated." 
  });
});

const PORT = 3000;
server.listen(PORT, () => console.log(`In-Memory Gateway running on port ${PORT}`));
```

### B. Frontend: JavaScript Client Application (`client.js`)
```javascript
import { io } from "socket.io-client";
const socket = io("http://localhost:3000");

async function handleLongRunningRequest() {
  try {
    uiUpdateState("Processing started... please wait.");

    // 1. Trigger the job on the backend via standard HTTP
    const response = await fetch("http://localhost:3000/api/start-task", {
      method: "POST"
    });
    const { taskId } = await response.json();
    console.log(`Acquired Tracking Token: ${taskId}`);

    // 2. Join the unique Socket.IO room for this specific task
    socket.emit("monitor_task", taskId);

    // 3. Listen for the completion push notification
    socket.on("task_completed", (result) => {
      console.log("Real-time notification payload received:", result);
      
      // Update application UI state with the final result
      renderTaskSuccess(result.payload.downloadUrl);
      
      // Memory cleanup: Tear down the listener when the lifecycle ends
      socket.off("task_completed");
    });

  } catch (error) {
    console.error("Task initiation pipeline failed:", error);
    uiUpdateState("An error occurred. Please try again.");
  }
}

function uiUpdateState(message) {
  document.getElementById("statusBox").innerText = message;
}

function renderTaskSuccess(url) {
  document.getElementById("statusBox").innerHTML = `
    <span style="color: green; font-weight: bold;">Task Complete!</span>
    <a href="${url}" download>Click here to download your report</a>
  `;
}

document.getElementById("submitBtn").addEventListener("click", handleLongRunningRequest);
```

---

## 4. Key Architectural Interview Talking Points
*   **Security & Data Isolation:** Using unique room names based on UUIDs or `taskId` tokens ensures that data payloads are only broadcasted to authorized sockets. Even if multiple users trigger tasks simultaneously, they are digitally isolated into separate rooms.
*   **Automatic Resource Cleanup:** When a user closes the browser or disconnects, Socket.IO cleans up empty rooms automatically. Once the room has 0 connected clients, it is destroyed from memory, preventing memory leaks.
*   **Decoupled Multi-Protocol Structure:** We combine **HTTP for Command Submission** (easy to authenticate, handles REST standards cleanly) with **WebSockets for State Notification** (ideal for low-latency, real-time messaging).
*   **Production Scaling Note:** In a clustered server environment (e.g., multiple Node.js instances behind a Load Balancer), standard memory rooms fail because the client might be connected to Server A while the background task completes on Server B. To scale this setup, mention that you would hook up the **Socket.IO Redis Adapter**, allowing rooms and events to seamlessly sync across all instances.
