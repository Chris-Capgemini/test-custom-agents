# 5. Building Block View

## 5.1 Level 1 – System Overview

The system consists of four independently runnable components:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Allegro Modernization PoC                         │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐ │
│  │ [1] Swing HTTP   │  │ [2] Node.js WS   │  │ [3] Swing WebSocket  │ │
│  │ Client           │  │ Relay Server     │  │ Client               │ │
│  └──────────────────┘  └──────────────────┘  └──────────────────────┘ │
│                                  ▲                                     │
│                         ┌────────┘                                     │
│                         │                                              │
│  ┌──────────────────────┐                                              │
│  │ [4] Vue.js Web       │                                              │
│  │ Client               │                                              │
│  └──────────────────────┘                                              │
└────────────────────────────────────────────────────────────────────────┘
```

| Block | Technology | Source Location | Responsibility |
|---|---|---|---|
| **[1] Swing HTTP Client** | Java 22, Swing, javax.json | `websocket_swing/swing/` | Desktop GUI for HTTP-based form submission to HTTPBin |
| **[2] Node.js WS Relay Server** | Node.js, ws npm package | `websocket_swing/node-server/` | Relays WebSocket messages between all connected clients |
| **[3] Swing WebSocket Client** | Java 22, Swing, Tyrus WebSocket | `websocket_swing/swing/src/main/java/websocket/` | Desktop GUI receiving form data via WebSocket from Vue client |
| **[4] Vue.js Web Client** | Vue 2, JavaScript | `websocket_swing/node-vue-client/` | Browser-based search UI; sends selected customer data via WebSocket |

---

## 5.2 Level 2 – Swing HTTP Client Internals

```
┌───────────────────────────────────────────────────────┐
│                 Swing HTTP Client                      │
│                                                       │
│  ┌───────────┐    ┌───────────────┐   ┌───────────┐  │
│  │ PocView   │◄──►│ PocPresenter  │──►│ PocModel  │  │
│  │ (View)    │    │ (Controller)  │   │ (Model)   │  │
│  └───────────┘    └───────────────┘   └─────┬─────┘  │
│                          ▲                  │        │
│                          │           ┌──────▼──────┐ │
│              EventEmitter│           │HttpBinSvc   │ │
│                          │           └─────────────┘ │
│  ┌────────────────────────────────────────────────┐  │
│  │  ValueModel<T>  ·  ModelProperties (enum)       │  │
│  └────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
```

| Class | Package | Responsibility |
|---|---|---|
| `Main` | `com` | Entry point; wires `PocView`, `PocModel`, `PocPresenter`, `EventEmitter` |
| `PocView` | `com.poc.presentation` | Renders Swing JFrame with all form fields (text fields, radio buttons, button) |
| `PocPresenter` | `com.poc.presentation` | Binds `DocumentListener` / `ChangeListener` on view fields to `ValueModel`; triggers `PocModel.action()` on submit |
| `PocModel` | `com.poc.model` | Holds `EnumMap<ModelProperties, ValueModel>` as the form state; calls `HttpBinService.post()` and emits result |
| `HttpBinService` | `com.poc.model` | Sends HTTP POST to `http://localhost:8080/post` with JSON payload; returns response body |
| `EventEmitter` | `com.poc.model` | Maintains list of `EventListener` callbacks; calls each listener with the response string |
| `EventListener` | `com.poc.model` | Functional interface defining the `onEvent(String response)` callback |
| `ValueModel<T>` | `com.poc` | Generic container wrapping a single mutable value |
| `ModelProperties` | `com.poc.model` | Enum with 13 constants defining all form field keys |

---

## 5.3 Level 2 – Node.js WebSocket Relay Server Internals

```
┌──────────────────────────────────────────────┐
│             Node.js Relay Server             │
│                                              │
│  HTTP Server (port 1337)                     │
│        │                                    │
│        ▼                                    │
│  WebSocket Server                           │
│        │                                    │
│   ┌────┴─────────────────────┐              │
│   │  clients[] (connection   │              │
│   │  registry)               │              │
│   └──────────────────────────┘              │
│                                              │
│  On message: broadcast to all clients        │
└──────────────────────────────────────────────┘
```

| Component | Responsibility |
|---|---|
| HTTP Server | Transport layer for WebSocket upgrade |
| WebSocket Server | Manages client connections; broadcasts any received message to all connected clients |
| `clients[]` array | In-memory registry of all active WebSocket connections |

---

## 5.4 Level 2 – Vue.js Web Client Internals

```
┌─────────────────────────────────────────────────────┐
│                 Vue.js Web Client                    │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  App.vue (root)                             │   │
│  │    └── Search.vue (main component)          │   │
│  │          ├── #person (search form)          │   │
│  │          ├── #search_result (results table) │   │
│  │          ├── #zahlungsempfaenger (accounts) │   │
│  │          ├── #send_to_allegro (submit btn)  │   │
│  │          └── #textarea (response display)  │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  Data: formdata, search_space, selected_result,     │
│        zahlungsempfaenger_selected                  │
│                                                     │
│  Methods: connect(), searchPerson(),                │
│           selectResult(), sendMessage()             │
└─────────────────────────────────────────────────────┘
```

| Component | Responsibility |
|---|---|
| `App.vue` | Root component; renders application header and mounts `Search.vue` |
| `Search.vue` | All UI logic: search form, results table, account selection, WebSocket send |

---

## 5.5 Level 2 – Swing WebSocket Client Internals

| Class / Inner Class | Responsibility |
|---|---|
| `websocket.Main` | Swing frame with same form layout; manages WebSocket lifecycle via `CountDownLatch` |
| `WebsocketClientEndpoint` | `@ClientEndpoint`; connects to `ws://localhost:1337/`; parses incoming JSON and populates form fields |
| `Message` | Inner DTO: `{ target: String, content: String }` |
| `SearchResult` | Inner DTO mapping JSON fields to `ModelProperties` names for form population |
