# 4. Solution Strategy

## 4.1 Technology Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Desktop GUI** | Java Swing | Existing Allegro desktop application is Swing-based; the PoC mirrors this technology to validate modernization paths. |
| **Web frontend** | Vue.js 2 (SPA) | Lightweight framework suitable for a PoC web client with minimal setup. |
| **Relay server** | Node.js + WebSocket | Minimal effort to implement a broadcast relay; no persistence or business logic required. |
| **HTTP echo backend** | HTTPBin (Docker) | Provides a ready-made echo endpoint for testing HTTP submissions without building a custom service. |
| **Architecture pattern** | Model–View–Presenter (MVP) | Decouples the Swing UI from business logic, making it easier to swap the backend or modify the view independently. |
| **Event propagation** | Observer pattern (EventEmitter) | Avoids tight coupling between the HTTP service and the presenter; model can notify multiple observers. |
| **Data format** | JSON | Universally supported; both Java (javax.json) and JavaScript handle JSON natively. |

## 4.2 Architectural Approaches

### Approach 1 – HTTP-Based Integration

The Swing HTTP client follows the MVP pattern:

```
PocView ──► PocPresenter ──► PocModel ──► HttpBinService ──► HTTPBin (Docker)
  ▲                │                            │
  └────────────────┘◄──── EventEmitter ◄────────┘
       (form clear)       (response event)
```

- The view is a passive renderer; all logic is in the presenter.
- `PocModel` acts as the aggregate root, collecting all form field values and delegating HTTP submission to `HttpBinService`.
- `EventEmitter` decouples the response notification from the caller.

### Approach 2 – WebSocket-Based Integration

```
Vue.js Search Client
       │  sendMessage() via WebSocket
       ▼
Node.js WebSocket Server (relay)
       │  broadcast to all clients
       ▼
Java Swing WebSocket Client
       │  onMessage() parses JSON, routes by "target"
       ▼
Form fields updated / textarea updated
```

- The Node.js server has no business logic; it only relays messages.
- Messages carry a `target` field (`"textfield"` or `"textarea"`) to route content to the correct GUI component.
- The Vue.js client contains a hard-coded in-memory customer database for the PoC.

## 4.3 Key Design Principles Applied

| Principle | How it is Applied |
|---|---|
| **Separation of Concerns** | MVP pattern keeps view, presenter, and model in separate classes. |
| **Loose Coupling** | `EventEmitter` / `EventListener` interfaces decouple model from presenter. |
| **Single Responsibility** | `HttpBinService` is solely responsible for HTTP communication; `PocModel` manages state. |
| **Convention over Configuration** | `ModelProperties` enum drives all field names, preventing magic strings across the codebase. |
