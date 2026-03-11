# arc42 Architecture Documentation

**websocket_swing — Allegro Modernization PoC**

*Version 1.0 | March 2026*

---

> **About arc42**
>
> arc42 is a template for architecture communication and documentation.
> This document follows the [arc42 template](https://arc42.org) structure to describe the architecture of the websocket_swing project.

---

## Table of Contents

1. [Introduction and Goals](#1-introduction-and-goals)
2. [Architecture Constraints](#2-architecture-constraints)
3. [System Scope and Context](#3-system-scope-and-context)
4. [Solution Strategy](#4-solution-strategy)
5. [Building Block View](#5-building-block-view)
6. [Runtime View](#6-runtime-view)
7. [Deployment View](#7-deployment-view)
8. [Crosscutting Concepts](#8-crosscutting-concepts)
9. [Architecture Decisions](#9-architecture-decisions)
10. [Quality Requirements](#10-quality-requirements)
11. [Risks and Technical Debts](#11-risks-and-technical-debts)
12. [Glossary](#12-glossary)

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

The **websocket_swing** project is a Proof of Concept (PoC) for the Allegro modernization initiative. It demonstrates real-time, bi-directional communication between a legacy Java Swing desktop client and a modern Vue.js web interface, connected through a Node.js WebSocket server.

The primary use case is:

> A case worker in a web browser searches for a person in a mock database and transfers the selected person's data (including payment information) directly into the legacy Allegro desktop application — in real time, without copy-paste.

**Key functional requirements:**

| ID  | Requirement |
|-----|-------------|
| F01 | A Vue.js web client must allow searching for persons by name, zip code, and city. |
| F02 | The selected person's data must be transferred to the Java Swing application in real time. |
| F03 | The Java Swing application must receive the data and populate the form fields automatically. |
| F04 | The Java Swing application must be able to submit form data via HTTP POST to a backend service. |
| F05 | A textarea in both clients must be kept synchronized in real time via WebSocket. |

### 1.2 Quality Goals

| Priority | Quality Goal | Motivation |
|----------|-------------|------------|
| 1 | **Real-time communication** | Data transfer between clients must be immediate (sub-second latency) to support a seamless user workflow. |
| 2 | **Multi-client support** | Both a Java Swing desktop client and a Vue.js web client must connect and communicate simultaneously. |
| 3 | **Extensibility** | The PoC must serve as a foundation that can be extended with additional clients or back-end systems. |
| 4 | **Simplicity** | As a PoC, the architecture must be easy to understand and demonstrate to stakeholders. |

### 1.3 Stakeholders

| Role | Expectations |
|------|-------------|
| **Allegro Product Team** | Wants to validate feasibility of modernizing the Allegro desktop workflow with a web interface. |
| **Developers** | Need a clean, understandable PoC to extend or integrate into the production system. |
| **Project Manager** | Needs a working demo to present to stakeholders as part of the Allegro modernization roadmap. |

---

## 2. Architecture Constraints

### 2.1 Technical Constraints

| Constraint | Background |
|-----------|------------|
| **Java SDK >= 22** | The Swing client is compiled against Java 22 features. All JVM-related tooling must use Java 22 or newer. |
| **Docker required** | The HTTPBin service must run in a Docker container (`kennethreitz/httpbin`) on port 8080. |
| **Node.js runtime** | The WebSocket server is implemented in Node.js and must be started separately. |
| **Local network** | All components communicate over localhost; no remote endpoints are configured. |
| **Vue 2.x** | The web client is implemented in Vue 2.6. Migration to Vue 3 is not in scope for the PoC. |

### 2.2 Organizational Constraints

| Constraint | Background |
|-----------|------------|
| **PoC scope** | This is a Proof of Concept only. Production concerns such as security, persistence, and scalability are deliberately out of scope. |
| **IntelliJ IDEA** | The Java Swing modules are configured for IntelliJ IDEA (`.iml` project file, `.launch` run configurations). |
| **No database** | All search data is mocked in the Vue client. No persistence layer exists. |

---

## 3. System Scope and Context

### 3.1 Business Context

The system connects a web-based search interface (Vue.js) with a legacy desktop application (Java Swing) through a real-time message broker (Node.js WebSocket server).

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          websocket_swing System                             │
│                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    Node.js WebSocket Server (:1337)                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│         ▲                                             ▲                      │
│         │ ws://localhost:1337                         │ ws://localhost:1337  │
│         │                                             │                      │
│  ┌──────┴───────────┐                    ┌────────────┴──────────────┐      │
│  │ Java Swing Client │                    │    Vue.js Web Client      │      │
│  │  (websocket/Main) │                    │   (node-vue-client)       │      │
│  └──────────────────┘                    └───────────────────────────┘      │
│         │                                                                    │
│         │ HTTP POST                                                           │
│         ▼                                                                    │
│  ┌──────────────────┐                                                        │
│  │   HTTPBin        │                                                        │
│  │ (Docker :8080)   │                                                        │
│  └──────────────────┘                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

**External actors:**

| Actor | Role |
|-------|------|
| **Case Worker (Web)** | Uses the Vue.js client in a browser to search for persons and transfer data to ALLEGRO. |
| **Case Worker (Desktop)** | Uses the Java Swing application to manage and submit person/payment data. |
| **HTTPBin** | Third-party HTTP echo service (Docker container) used to simulate a backend POST endpoint. |

### 3.2 Technical Context

| Interface | Technology | Direction | Description |
|-----------|-----------|-----------|-------------|
| WebSocket (:1337) | WebSocket (RFC 6455) | bidirectional | Real-time message exchange between Vue client, Node server, and Swing client |
| HTTP POST (:8080/post) | HTTP 1.1 / JSON | Swing → HTTPBin | Swing client submits form data to HTTPBin; response is echoed back |
| Browser → Vue | HTTP (:8081) | User → Browser | Vue.js is served via webpack dev server |

---

## 4. Solution Strategy

### 4.1 Core Technology Decisions

| Decision | Rationale |
|---------|-----------|
| **WebSocket for real-time sync** | WebSocket provides low-latency, full-duplex communication, avoiding polling. This makes data transfer between clients feel instantaneous. |
| **Node.js as message broker** | Node.js has native, lightweight WebSocket support. It acts as a simple broadcast relay without complex routing logic. |
| **Vue.js for the web client** | Vue.js provides reactive data binding, making UI updates driven by WebSocket messages straightforward. |
| **Java Swing desktop client** | Represents the existing ALLEGRO desktop client (legacy system). The PoC simulates this client receiving data from the web interface. |
| **MVP pattern for Swing** | The Model-View-Presenter pattern separates UI, business logic, and data binding in the Swing application, making the code maintainable and testable. |
| **JSON as message format** | JSON is human-readable, widely supported across Java, JavaScript, and Vue.js, and avoids the need for binary protocol implementation. |

### 4.2 Architecture Strategy

The architecture uses a **star topology** with the Node.js server at the center:

- The Node.js server is a pure **broadcast relay**: every message received from any client is forwarded to all connected clients.
- Clients use a **"target" field** in the JSON message to identify what type of content is being sent (`"textfield"` or `"textarea"`), allowing clients to route incoming messages internally.
- The Java Swing application has **two independent implementations** in this PoC: one using a direct MVP/HTTP approach (`com.poc.*`) and one using WebSocket (`websocket.Main`), demonstrating two integration pathways.

---

## 5. Building Block View

### 5.1 Level 1 — System Overview

```
┌───────────────────────────────────────────────────────────┐
│                  websocket_swing System                    │
│                                                           │
│   ┌────────────────┐   ┌────────────────┐   ┌─────────┐  │
│   │  node-server   │   │ node-vue-client│   │  swing  │  │
│   │ (WebSocket Srv)│   │  (Web Client)  │   │ (Swing) │  │
│   └────────────────┘   └────────────────┘   └─────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

| Building Block | Responsibility |
|----------------|---------------|
| **node-server** | Accepts WebSocket connections from all clients. Broadcasts every received message to all connected clients. |
| **node-vue-client** | Web-based search interface. Allows searching and selecting persons from mock data. Sends selected persons to ALLEGRO via WebSocket. |
| **swing** | Java desktop client with two implementations: MVP-based HTTP form submission and WebSocket-based real-time data receiver. |

### 5.2 Level 2 — node-server

**File:** `node-server/src/WebsocketServer.js`

```
┌───────────────────────────────────────────┐
│           WebsocketServer.js              │
│                                           │
│  ┌──────────────┐   ┌──────────────────┐  │
│  │  HTTP Server │──►│ WebSocket Server │  │
│  │  (base)      │   │  (port 1337)     │  │
│  └──────────────┘   └───────┬──────────┘  │
│                             │             │
│                   ┌─────────▼──────────┐  │
│                   │  clients[]  array  │  │
│                   │  (connection pool) │  │
│                   └─────────┬──────────┘  │
│                             │ broadcast   │
│                             ▼             │
│                   for each client:        │
│                     client.sendUTF(msg)   │
└───────────────────────────────────────────┘
```

| Component | Responsibility |
|-----------|---------------|
| HTTP Server | Underlying HTTP server required by the WebSocket library. Does not handle HTTP requests. |
| WebSocket Server | Wraps the HTTP server; manages WebSocket connections. |
| clients[] array | Stores all active WebSocket connections. |
| Broadcast | On every incoming message, sends the message to all clients in the array. |

### 5.3 Level 2 — node-vue-client

**Directory:** `node-vue-client/src/`

```
┌──────────────────────────────────────────────┐
│                 Vue.js App                    │
│                                              │
│  ┌──────────┐    ┌────────────────────────┐  │
│  │  App.vue │───►│      Search.vue        │  │
│  │ (root)   │    │                        │  │
│  └──────────┘    │  ┌──────────────────┐  │  │
│                  │  │  Search Form     │  │  │
│                  │  │  (14 inputs)     │  │  │
│                  │  └──────────────────┘  │  │
│                  │  ┌──────────────────┐  │  │
│                  │  │  Results Table   │  │  │
│                  │  │  (5 mock persons)│  │  │
│                  │  └──────────────────┘  │  │
│                  │  ┌──────────────────┐  │  │
│                  │  │  Payment Table   │  │  │
│                  │  └──────────────────┘  │  │
│                  │  ┌──────────────────┐  │  │
│                  │  │  WebSocket       │  │  │
│                  │  │  Client          │  │  │
│                  │  └──────────────────┘  │  │
│                  └────────────────────────┘  │
└──────────────────────────────────────────────┘
```

| Component | Responsibility |
|-----------|---------------|
| `App.vue` | Root component; renders header with branding and mounts the Search component. |
| `Search.vue` | Main application logic: person search, result display, selection, payment info, and WebSocket integration. |
| Search Form | 14 input fields for search criteria. Client-side filtering against mock data. |
| Results Table | Displays matching persons. Row click selects a person and loads their payment data. |
| Payment Table | Displays payment entries for the selected person. |
| WebSocket Client | Connects to `ws://localhost:1337`. Sends selected person data and syncs textarea. |

### 5.4 Level 2 — swing

**Directory:** `swing/src/main/java/`

```
┌──────────────────────────────────────────────────────────────┐
│                        swing module                          │
│                                                              │
│  ┌─────────────────────────────────┐                         │
│  │   com.poc (MVP Implementation)  │                         │
│  │                                 │                         │
│  │  ┌──────────┐  ┌────────────┐   │                         │
│  │  │ PocView  │  │ PocModel   │   │                         │
│  │  │ (Swing   │  │ (Business  │   │                         │
│  │  │  UI)     │  │  Logic)    │   │                         │
│  │  └────┬─────┘  └──────┬─────┘   │                         │
│  │       │               │         │                         │
│  │  ┌────▼───────────────▼─────┐   │                         │
│  │  │      PocPresenter        │   │                         │
│  │  │  (Binding & Event Hdlg)  │   │                         │
│  │  └──────────────────────────┘   │                         │
│  │                                 │                         │
│  │  ┌─────────────┐  ┌──────────┐  │                         │
│  │  │HttpBinSvc   │  │EventEmit │  │                         │
│  │  │(HTTP POST)  │  │(Pub-Sub) │  │                         │
│  │  └─────────────┘  └──────────┘  │                         │
│  └─────────────────────────────────┘                         │
│                                                              │
│  ┌─────────────────────────────────┐                         │
│  │  websocket (WS Implementation)  │                         │
│  │                                 │                         │
│  │  ┌──────────────────────────┐   │                         │
│  │  │ WebsocketClientEndpoint  │   │                         │
│  │  │  (@ClientEndpoint JSR-356│   │                         │
│  │  │   connects :1337)        │   │                         │
│  │  └──────────────────────────┘   │                         │
│  └─────────────────────────────────┘                         │
└──────────────────────────────────────────────────────────────┘
```

#### com.poc — MVP Implementation

| Class | Responsibility |
|-------|---------------|
| `com.Main` | Application entry point. Instantiates PocView, PocModel, and PocPresenter. |
| `PocView` | Pure Swing UI. GridBagLayout form with 13 text fields, 3 radio buttons, 1 button, 1 text area. No business logic. |
| `PocPresenter` | Binds PocView UI elements to PocModel ValueModel instances via DocumentListeners. Handles button actions. Listens to EventEmitter for server responses. |
| `PocModel` | Business logic layer. Holds an EnumMap of ValueModel instances. On `action()`: collects field values and calls HttpBinService. |
| `HttpBinService` | HTTP client. Posts form data as JSON to `http://localhost:8080/post`. Returns echoed response. |
| `EventEmitter` | Publisher-Subscriber implementation. Notifies registered EventListeners of new events (response strings). |
| `ValueModel<T>` | Generic typed wrapper for a single form field value. Enables type-safe data binding. |
| `ModelProperties` | Enum defining all form field keys (e.g., `FIRST_NAME`, `LAST_NAME`, `IBAN`). |

#### websocket — WebSocket Implementation

| Class | Responsibility |
|-------|---------------|
| `websocket.Main` | Entry point for the WebSocket-based Swing client. Connects to Node.js server at `ws://localhost:1337`. Uses CountDownLatch to keep thread alive. |
| `WebsocketClientEndpoint` | JSR-356 `@ClientEndpoint`. Handles `@OnOpen`, `@OnMessage`, `@OnClose`. On message: parses JSON and updates Swing UI fields. |
| `SearchResult` | Data Transfer Object. Holds person fields received from the WebSocket message. |

---

## 6. Runtime View

### 6.1 Person Data Transfer: Vue → Swing

This scenario shows the primary use case: a case worker selects a person in the web client and transfers data to the Swing client.

```
Vue Client          Node.js Server       Swing Client
    │                      │                   │
    │  connect()            │                   │
    │──────────────────────►│                   │
    │                       │  connectToServer() │
    │                       │◄──────────────────│
    │                       │  @OnOpen          │
    │                       │──────────────────►│
    │                       │                   │
    │  [user selects row]   │                   │
    │  [clicks "Nach ALLEGRO│                   │
    │   übernehmen"]        │                   │
    │                       │                   │
    │  sendMessage(person,  │                   │
    │  'textfield')         │                   │
    │──────────────────────►│                   │
    │                       │  broadcast(msg)   │
    │◄──────────────────────│──────────────────►│
    │  (echo back)          │                   │  @OnMessage
    │                       │                   │  parseJSON(msg)
    │                       │                   │  target="textfield"
    │                       │                   │  update UI fields:
    │                       │                   │  tf_name, tf_first,
    │                       │                   │  tf_iban, tf_bic...
    │                       │                   │
```

**JSON message format:**

```json
{
  "target": "textfield",
  "content": {
    "name": "Mayer",
    "first": "Hans",
    "knr": "79423984",
    "dob": "1985-03-15",
    "zip": "95183",
    "ort": "Musterstadt",
    "street": "Hauptstraße",
    "hausnr": "12",
    "zahlungsempfaenger": {
      "iban": "DE27100777770209299700",
      "bic": "ERFBDE8E759",
      "valid_from": "2020-01-04"
    }
  }
}
```

### 6.2 Textarea Synchronization

Both clients keep a shared textarea in sync:

```
Vue Client          Node.js Server       Swing Client
    │                      │                   │
    │  [user types in       │                   │
    │   textarea]          │                   │
    │                       │                   │
    │  watch: textarea →    │                   │
    │  sendMessage(text,    │                   │
    │  'textarea')          │                   │
    │──────────────────────►│                   │
    │                       │  broadcast(msg)   │
    │                       │──────────────────►│
    │                       │                   │  @OnMessage
    │                       │                   │  target="textarea"
    │                       │                   │  textArea.setText(...)
```

### 6.3 Form Submission: Swing → HTTPBin

After the case worker reviews the form data in the Swing client:

```
Swing Client (com.poc)       HTTPBin (:8080)
        │                           │
        │  [user clicks "Anordnen"] │
        │                           │
        │  PocPresenter.action()    │
        │  PocModel.action()        │
        │  HttpBinService.post(data)│
        │  HTTP POST /post (JSON)   │
        │──────────────────────────►│
        │                           │  echo response
        │◄──────────────────────────│
        │                           │
        │  EventEmitter.emit(resp)  │
        │  PocPresenter updates UI  │
        │  textArea shows response  │
        │  form fields cleared      │
```

---

## 7. Deployment View

### 7.1 Infrastructure

All components run on the same local machine (localhost) in this PoC setup:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Developer Workstation                              │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  JVM Process (Java 22)                                              │   │
│  │  websocket_swing/swing — Main.java                                  │   │
│  │  Port: none (client-only, connects out to :1337 and :8080)          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Node.js Process                                                    │   │
│  │  websocket_swing/node-server — WebsocketServer.js                   │   │
│  │  Port: 1337 (WebSocket)                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Node.js Process (Vue CLI Dev Server)                               │   │
│  │  websocket_swing/node-vue-client                                    │   │
│  │  Port: 8081 (HTTP, webpack dev server)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Docker Container                                                   │   │
│  │  Image: kennethreitz/httpbin                                        │   │
│  │  Port: 8080 (HTTP, mapped from container port 80)                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Port Assignments

| Port | Protocol | Component | Direction |
|------|----------|-----------|-----------|
| 1337 | WebSocket | node-server | inbound from Vue client and Swing WS client |
| 8080 | HTTP | HTTPBin (Docker) | inbound from Swing MVP client |
| 8081 | HTTP | Vue CLI dev server | inbound from browser |

### 7.3 Startup Order

The following startup order must be observed:

```
1. Start Docker (Docker Desktop or Rancher Desktop)
2. docker run -p 8080:80 kennethreitz/httpbin     # HTTPBin
3. cd node-server && node src/WebsocketServer.js  # WebSocket server
4. cd node-vue-client && npm run serve             # Vue web client
5. Run Swing client via IntelliJ (websocket/Main or com/Main)
```

---

## 8. Crosscutting Concepts

### 8.1 Message Protocol

All WebSocket messages use JSON with an envelope format:

```json
{
  "target": "<message-type>",
  "content": { ... }
}
```

The `"target"` field acts as a message discriminator:

| Target value | Content schema | Handled by |
|-------------|---------------|------------|
| `"textfield"` | SearchResult object (person + payment data) | Swing `@OnMessage` → updates text fields |
| `"textarea"` | Plain string | Swing `@OnMessage` → updates JTextArea; Vue watcher → updates textarea |

### 8.2 Error Handling

The PoC uses minimal error handling:

- Java HTTP calls wrap exceptions in `RuntimeException` (fail-fast strategy).
- WebSocket `@OnClose` uses a `CountDownLatch` to signal the main thread to exit cleanly.
- The Node.js server does not validate message content; malformed messages are broadcast as-is.

### 8.3 JSON Processing

| Component | Library | Approach |
|-----------|---------|---------|
| Java Swing (sending) | `javax.json` (JSON-P) | `JsonGenerator` for building JSON output |
| Java Swing (receiving) | `javax.json.stream.JsonParser` | Event-based streaming parser; manual field extraction |
| Node.js server | Native JavaScript | No parsing; messages forwarded as raw strings |
| Vue.js client | `JSON.stringify` / `JSON.parse` | Native browser JSON API |

### 8.4 Concurrency

- **Java Swing client**: Swing UI updates from WebSocket `@OnMessage` callbacks must run on the Event Dispatch Thread (EDT). The current implementation updates UI fields directly in the handler — this should be wrapped in `SwingUtilities.invokeLater()` for production use.
- **CountDownLatch**: Used in `websocket.Main` to keep the main thread alive while the WebSocket connection is active.
- **Node.js server**: Single-threaded event loop; no concurrency issues.

### 8.5 Data Binding (MVP Pattern)

In the `com.poc` implementation, data binding is implemented as follows:

```
PocView (Swing UI)
    │
    │ DocumentListener
    ▼
PocPresenter
    │
    │ reads/writes
    ▼
ValueModel<T>  ──────────► PocModel (EnumMap<ModelProperties, ValueModel>)
```

Each form field is backed by a `ValueModel<String>` instance in the `PocModel`'s `EnumMap`. The presenter attaches a `DocumentListener` to each `JTextField`, which updates the corresponding `ValueModel` on every keystroke.

---

## 9. Architecture Decisions

### ADR-001: WebSocket as Communication Protocol

**Context:** We need real-time, bi-directional communication between a desktop client (Java) and a web client (Vue.js).

**Decision:** Use WebSocket (RFC 6455) via a Node.js relay server.

**Consequences:**
- ✅ Sub-second latency for data transfer.
- ✅ Supported by all modern browsers and by JSR-356 in Java.
- ✅ Simple to implement for a PoC.
- ❌ Requires a running WebSocket server; no offline mode.
- ❌ No message persistence; messages lost if client is not connected.

---

### ADR-002: Node.js as Broadcast Relay

**Context:** We need a server to forward WebSocket messages between clients.

**Decision:** Use a minimal Node.js WebSocket server that broadcasts all messages to all connected clients.

**Consequences:**
- ✅ Extremely simple implementation (< 40 lines of code).
- ✅ No message routing logic required.
- ❌ No message filtering; all clients receive all messages.
- ❌ Does not scale beyond a small number of connections.

---

### ADR-003: Two Swing Client Implementations

**Context:** The PoC needs to demonstrate both an HTTP-based workflow and a WebSocket-based workflow.

**Decision:** Provide two separate Swing client entry points:
- `com.Main` — MVP-based client using HTTP POST to HTTPBin.
- `websocket.Main` — WebSocket-based client that receives data from the Vue client.

**Consequences:**
- ✅ Demonstrates two integration pathways independently.
- ❌ Some code duplication (two independent UI implementations).

---

### ADR-004: Mock Data in Vue Client

**Context:** A backend search service is not available for this PoC.

**Decision:** Embed 5 hardcoded mock persons directly in `Search.vue`.

**Consequences:**
- ✅ No backend dependencies; demo can run fully offline.
- ❌ Data is static and cannot be updated without code changes.

---

### ADR-005: JSON-P for Java JSON Handling

**Context:** Java Swing client needs to produce and consume JSON.

**Decision:** Use `javax.json` (JSON-P) API with event-based streaming parser.

**Consequences:**
- ✅ No third-party libraries like Jackson or Gson required.
- ✅ Explicit, predictable parsing behavior.
- ❌ Verbose; no automatic POJO serialization/deserialization.

---

## 10. Quality Requirements

### 10.1 Quality Tree

```
Quality
├── Real-time Performance
│   └── WebSocket message delivery < 100ms on LAN
├── Reliability
│   └── Server broadcasts to all connected clients without loss
├── Usability
│   ├── Vue search form filters results immediately (client-side)
│   └── Swing form fields auto-populate on WebSocket message receipt
├── Maintainability
│   ├── MVP pattern separates Swing UI, presenter, and model
│   └── Clear module boundaries (node-server, node-vue-client, swing)
└── Portability
    └── Runs on Windows, macOS, Linux with Java 22, Node.js, and Docker
```

### 10.2 Quality Scenarios

| Scenario | Stimulus | Response | Measurement |
|---------|---------|---------|------------|
| Data transfer speed | Case worker clicks "Nach ALLEGRO übernehmen" | Swing form fields populated | < 200ms end-to-end on localhost |
| Multi-client | Vue and Swing both connected | Both receive broadcasts | No messages lost |
| Startup | Developer follows README steps | All components running | < 5 minutes from checkout to running demo |

---

## 11. Risks and Technical Debts

### 11.1 Risks

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|-----------|
| **No authentication** | High (given current state) | High (in production) | For PoC only; add OAuth/JWT before production use |
| **Unencrypted WebSocket (ws://)** | High | High (in production) | Use WSS (WebSocket Secure) with TLS in production |
| **Broadcast to all clients** | High | Medium | In production, implement targeted message routing (e.g., room IDs or user tokens) |
| **Swing UI updates outside EDT** | Medium | Medium | Wrap `@OnMessage` UI updates in `SwingUtilities.invokeLater()` |
| **No server-side input validation** | High | Medium | Add message schema validation in Node.js server |

### 11.2 Technical Debts

| Item | Area | Description | Effort to Fix |
|------|------|-------------|--------------|
| **No unit tests** | All modules | No JUnit, Jest, or other test framework present | High |
| **Vue 2.x** | node-vue-client | Vue 2 is in maintenance mode; Vue 3 is the current standard | Medium |
| **Hardcoded URLs** | All modules | `ws://localhost:1337`, `http://localhost:8080` are hardcoded | Low |
| **No logging framework** | Java + Node.js | Uses `System.out.println` and `console.log`; no structured logging | Low |
| **EDT safety** | swing/websocket | WebSocket `@OnMessage` updates Swing components outside the EDT | Low |
| **Empty `ViewData.java`** | swing/com.poc | Placeholder class with no content | Low |
| **No CORS policy** | node-server | Server accepts connections from any origin | Low |

---

## 12. Glossary

| Term | Definition |
|------|-----------|
| **ALLEGRO** | The legacy desktop application being modernized in the Allegro modernization project. In this PoC, represented by the Java Swing client. |
| **Arc42** | A software architecture documentation template with 12 standardized sections. See [arc42.org](https://arc42.org). |
| **Broadcast** | Sending a WebSocket message to all connected clients simultaneously. |
| **EDT (Event Dispatch Thread)** | The single thread in Java Swing responsible for all UI updates. Non-EDT UI updates can cause race conditions or visual glitches. |
| **HTTPBin** | An open-source HTTP request/response service that echoes POST request data as JSON. Used as a mock backend. |
| **JSR-356** | Java API for WebSocket — the Java standard for WebSocket communication (`@ClientEndpoint`, `@OnMessage`, etc.). |
| **JSON-P** | Java API for JSON Processing (`javax.json`). Part of the Jakarta EE platform. |
| **MVP (Model-View-Presenter)** | A software architectural pattern separating the application into three components: the data model, the UI view, and the presenter that mediates between them. |
| **Nach ALLEGRO übernehmen** | German for "Transfer to ALLEGRO." The button in the Vue client that sends the selected person to the Swing client via WebSocket. |
| **Node.js** | A JavaScript runtime environment for server-side applications. Used here to run the WebSocket relay server. |
| **PoC (Proof of Concept)** | A preliminary implementation to validate a technical approach before committing to a full production implementation. |
| **Tyrus** | The reference implementation of JSR-356 (Java WebSocket API), used in the Swing client. |
| **ValueModel\<T\>** | A generic wrapper class in `com.poc` that holds a single typed form field value and supports data binding. |
| **Vue.js** | A progressive JavaScript framework for building web user interfaces. |
| **WebSocket** | A communication protocol providing full-duplex communication channels over a single TCP connection (RFC 6455). |
| **ws://** | The URI scheme for unencrypted WebSocket connections. The secure variant is `wss://`. |
| **Zahlungsempfänger** | German for "payment recipient." Refers to the payment information (IBAN, BIC, valid_from) associated with a person in the mock data. |
