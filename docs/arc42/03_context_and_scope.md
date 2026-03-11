# 3. Context and Scope

## 3.1 Business Context

The system assists operators in looking up customer records and forwarding selected customer and payment beneficiary data to the Allegro backend system.

```
┌──────────────────────────────────────────────────────────────┐
│                    Allegro Modernization PoC                  │
│                                                              │
│   ┌────────────┐     ┌─────────────┐    ┌────────────────┐  │
│   │ Java Swing │     │  Node.js    │    │  Vue.js Web    │  │
│   │ HTTP Client│     │  WebSocket  │    │  Client        │  │
│   └─────┬──────┘     │  Server     │    └───────┬────────┘  │
│         │            └──────┬──────┘            │           │
│         │                   │◄──────────────────┘           │
└─────────┼───────────────────┼───────────────────────────────┘
          │ HTTP POST          │ WebSocket broadcast
          ▼                   ▼
  ┌───────────────┐   ┌────────────────┐
  │  HTTPBin      │   │  Java Swing    │
  │  Echo Service │   │  WebSocket     │
  │  (Docker)     │   │  Client        │
  └───────────────┘   └────────────────┘
```

### External Systems

| External System | Direction | Communication | Purpose |
|---|---|---|---|
| **HTTPBin** (Docker) | Outbound | HTTP POST (`/post`) | Echoes submitted form data as JSON for testing purposes. |
| **Allegro Backend** | Outbound | TBD (represented by HTTPBin in PoC) | Target system that receives customer / payment data. |
| **Operator (Desktop)** | Inbound | Java Swing GUI | Inputs search criteria, selects records, triggers submission. |
| **Operator (Browser)** | Inbound | Vue.js SPA via WebSocket | Searches customer records, selects and forwards data to the Swing client. |

## 3.2 Technical Context

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Developer Machine                            │
│                                                                      │
│  ┌────────────────────┐        ┌────────────────────────────────┐   │
│  │  Java Swing        │        │  Node.js WebSocket Server      │   │
│  │  HTTP Client       │        │  ws://localhost:1337/          │   │
│  │  (Java >= 22.0.1)  │        └───────────────┬────────────────┘   │
│  └────────┬───────────┘                        │ WS broadcast       │
│           │ HTTP POST                          │                     │
│           ▼                                   ▼                     │
│  ┌────────────────────┐        ┌────────────────────────────────┐   │
│  │  HTTPBin           │        │  Java Swing WebSocket Client   │   │
│  │  http://localhost: │        │  + Vue.js Web Client           │   │
│  │  8080/post         │        │  http://localhost:8081         │   │
│  │  (Docker)          │        │  (Vue dev server)              │   │
│  └────────────────────┘        └────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────┘
```

### Communication Channels

| Channel | Protocol | Port | Direction | Payload |
|---|---|---|---|---|
| Swing HTTP → HTTPBin | HTTP/1.1 | 8080 | Outbound | JSON body (form data) |
| Vue.js → Node WS Server | WebSocket (`ws://`) | 1337 | Outbound | JSON message `{target, content}` |
| Node WS Server → Swing WS Client | WebSocket (`ws://`) | 1337 | Inbound | JSON message `{target, content}` |
| Swing WS Client → Node WS Server | WebSocket (`ws://`) | 1337 | Outbound | JSON message |
