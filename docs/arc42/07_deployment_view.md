# 7. Deployment View

## 7.1 Infrastructure Overview

All components run on a single developer machine (localhost). There is no staging or production environment for this PoC.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Developer Machine                            │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  JVM Process (Java >= 22)                                    │   │
│  │  ┌─────────────────────────┐  ┌──────────────────────────┐  │   │
│  │  │  Swing HTTP Client      │  │  Swing WebSocket Client  │  │   │
│  │  │  com.Main               │  │  websocket.Main          │  │   │
│  │  └─────────────────────────┘  └──────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌───────────────────────────────────────┐                         │
│  │  Node.js Process                      │                         │
│  │  WebSocket Relay Server               │                         │
│  │  Port: 1337                           │                         │
│  └───────────────────────────────────────┘                         │
│                                                                     │
│  ┌───────────────────────────────────────┐                         │
│  │  Node.js Process (Vue CLI dev server) │                         │
│  │  Vue.js Web Client                    │                         │
│  │  Port: 8081 (default Vue CLI port)    │                         │
│  └───────────────────────────────────────┘                         │
│                                                                     │
│  ┌───────────────────────────────────────┐                         │
│  │  Docker Container                     │                         │
│  │  kennethreitz/httpbin                 │                         │
│  │  Port: 8080 (mapped from 80)          │                         │
│  └───────────────────────────────────────┘                         │
│                                                                     │
│  Browser (any)                                                      │
│  http://localhost:8081                                              │
└─────────────────────────────────────────────────────────────────────┘
```

## 7.2 Process Inventory

| Process | Runtime | Port | Start Command |
|---|---|---|---|
| **Swing HTTP Client** | Java >= 22 | — | Run `com.Main` from IntelliJ |
| **Swing WebSocket Client** | Java >= 22 | — | Run `websocket.Main` from IntelliJ |
| **Node.js WS Relay Server** | Node.js | 1337 | `node src/WebsocketServer.js` (in `node-server/`) |
| **Vue.js Dev Server** | Node.js | 8081 | `yarn serve` (in `node-vue-client/`) |
| **HTTPBin Echo Service** | Docker | 8080 | `docker run -p 8080:80 kennethreitz/httpbin` |

## 7.3 Dependencies Between Processes

```
Swing HTTP Client ──────────────────────────────► HTTPBin (port 8080)

Vue.js Web Client ─────────────────────────────► Node.js WS Server (port 1337)
Node.js WS Server ─────────────────────────────► Swing WS Client (receives broadcast)
Swing WS Client ───────────────────────────────► Node.js WS Server (port 1337)
```

**Startup Order for WebSocket Scenario:**
1. Start Node.js WebSocket relay server.
2. Start Swing WebSocket client (connects to WS server on startup).
3. Start Vue.js dev server.
4. Open browser and connect to `http://localhost:8081`.

**Startup Order for HTTP Scenario:**
1. Start Docker container with HTTPBin.
2. Start Swing HTTP client.

## 7.4 Network Ports Summary

| Port | Protocol | Used By | Direction |
|---|---|---|---|
| `8080` | HTTP | HTTPBin echo service | Inbound from Swing HTTP client |
| `1337` | WebSocket (ws://) | Node.js relay server | Inbound from Vue.js and Swing WS clients |
| `8081` | HTTP | Vue.js dev server | Inbound from browser |
