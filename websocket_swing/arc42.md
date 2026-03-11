# arc42 Architecture Documentation
## Allegro Modernization PoC – WebSocket Swing

**Version:** 1.0  
**Date:** 2026-03-11  
**Status:** Draft  

---

## Table of Contents

1. [Introduction and Goals](#1-introduction-and-goals)
2. [Architecture Constraints](#2-architecture-constraints)
3. [System Scope and Context](#3-system-scope-and-context)
4. [Solution Strategy](#4-solution-strategy)
5. [Building Block View](#5-building-block-view)
6. [Runtime View](#6-runtime-view)
7. [Deployment View](#7-deployment-view)
8. [Cross-cutting Concepts](#8-cross-cutting-concepts)
9. [Architecture Decisions](#9-architecture-decisions)
10. [Quality Requirements](#10-quality-requirements)
11. [Risks and Technical Debt](#11-risks-and-technical-debt)
12. [Glossary](#12-glossary)

---

## 1. Introduction and Goals

### 1.1 Requirements Overview

This project is a **Proof of Concept (PoC)** demonstrating the technical feasibility of incrementally modernizing the legacy **ALLEGRO** desktop application. The PoC answers the key question: _Can a modern web-based UI communicate bidirectionally with the existing Java Swing desktop client using WebSockets, without replacing the legacy system entirely?_

The main functional requirements demonstrated are:

| ID  | Requirement                                                                                              |
|-----|----------------------------------------------------------------------------------------------------------|
| F1  | A web-based search UI allows users to search for persons by name, address, or other attributes.          |
| F2  | The user can select a search result and a payment recipient (Zahlungsempfänger) in the web UI.           |
| F3  | The selected data can be transferred ("Nach ALLEGRO übernehmen") to the ALLEGRO Swing client in real time via WebSocket. |
| F4  | The ALLEGRO Swing client receives the transferred data and auto-fills its form fields accordingly.       |
| F5  | The Swing client can submit form data to a backend HTTP service (HTTPBin mock).                          |
| F6  | A free-text area in the web UI can also push content to the Swing client's text area in real time.       |

### 1.2 Quality Goals

| Priority | Quality Goal     | Motivation                                                                                       |
|----------|------------------|--------------------------------------------------------------------------------------------------|
| 1        | Interoperability | The modern web UI must be able to communicate seamlessly with the existing Java Swing application.|
| 2        | Simplicity       | As a PoC, the architecture must be easy to set up and understand for demonstration purposes.     |
| 3        | Extensibility    | The chosen patterns (MVP for Swing, component-based for Vue) should support future growth.       |

### 1.3 Stakeholders

| Role              | Name / Group       | Expectation                                                          |
|-------------------|--------------------|----------------------------------------------------------------------|
| Developer / PoC Author | Capgemini Team | Validate the technical approach of the modernization strategy.       |
| Business Sponsor  | ALLEGRO Project    | Confirm that legacy investment can be preserved during modernization. |
| End User          | ALLEGRO Users      | Experience a modern UI while the legacy backend remains operational. |

---

## 2. Architecture Constraints

### 2.1 Technical Constraints

| Constraint | Description |
|---|---|
| Java >= 22.0.1 | The Swing client requires Java SDK version 22.0.1 or higher. |
| Maven | The Swing module is built with Apache Maven. |
| Node.js | The WebSocket relay server and the Vue.js development toolchain require Node.js. |
| Docker | The HTTPBin mock backend is run as a Docker container (`kennethreitz/httpbin` on port 8080). |
| Local Network | All components run on `localhost` in the PoC. No external network access is required. |
| WebSocket Protocol | The real-time communication channel is based on the standard WebSocket protocol (RFC 6455). |

### 2.2 Organizational Constraints

| Constraint | Description |
|---|---|
| PoC Scope | This is a demonstration, not a production system. No authentication, authorization, or data persistence is implemented. |
| Mock Data | Person data (search space) is hard-coded in the Vue frontend. No real database or backend search service is used. |

---

## 3. System Scope and Context

### 3.1 Business Context

The system supports a workflow where a front-office agent uses a **modern web UI** to look up a client's data and then transfers the result into the **legacy ALLEGRO desktop application** with a single click. This eliminates manual re-typing and reduces errors.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               User's Desktop                                │
│                                                                             │
│   ┌─────────────────────┐       ┌──────────────────────────────────────┐   │
│   │  Modern Web Browser │       │  ALLEGRO Java Swing Desktop Client   │   │
│   │  (Vue.js Search UI) │       │  (Legacy System Target)              │   │
│   └─────────────────────┘       └──────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

| Neighbor System     | Type           | Communication           | Description                                     |
|---------------------|----------------|-------------------------|-------------------------------------------------|
| Web Browser         | External Actor | HTTP / WebSocket (WS)   | The user interacts with the Vue.js client here. |
| ALLEGRO Swing App   | Internal System| WebSocket (WS)          | The legacy Java desktop application.            |
| HTTPBin (Mock)      | External System| HTTP POST               | Simulates a real backend REST API.              |

### 3.2 Technical Context

```
┌──────────────────┐    WS (port 1337)    ┌──────────────────────┐    WS (port 1337)    ┌──────────────────────────┐
│  Vue.js Client   │ ──────────────────►  │  Node.js WS Server   │ ──────────────────►  │  Java Swing WS Client    │
│  (Browser)       │ ◄──────────────────  │  (Relay/Broadcast)   │ ◄──────────────────  │  (websocket/Main.java)   │
└──────────────────┘                      └──────────────────────┘                      └──────────────────────────┘
                                                                                                     │
                                                                                         HTTP POST (port 8080)
                                                                                                     │
                                                                                                     ▼
                                                                                         ┌────────────────────────┐
                                                                                         │  HTTPBin (Docker)      │
                                                                                         │  Mock REST Backend     │
                                                                                         └────────────────────────┘
```

| Interface          | Technology            | Port  | Direction                    |
|--------------------|-----------------------|-------|------------------------------|
| Vue → WS Server    | WebSocket (RFC 6455)  | 1337  | Bidirectional                |
| Swing → WS Server  | WebSocket (JSR-356)   | 1337  | Bidirectional (receive only in this PoC) |
| Swing → HTTPBin    | HTTP POST (JSON)      | 8080  | Outbound                     |

---

## 4. Solution Strategy

The core idea is to use a **WebSocket relay server** as a technology bridge:

1. The **legacy Java Swing application** is enhanced with a WebSocket client (using Tyrus/JSR-356) so it can receive messages without modification to its core business logic.
2. A **lightweight Node.js WebSocket server** acts as a message broker. It broadcasts any received message to all connected clients (both the web client and the Swing client).
3. A **modern Vue.js web client** connects to the same WebSocket server. It provides the new search-and-transfer UI and sends structured JSON messages.

This strategy avoids a full rewrite. The Swing application remains operational while a new UI layer is added in front of it. The architectural pattern chosen for the Swing application itself is **Model-View-Presenter (MVP)**, which cleanly separates UI, presentation logic, and data—making future testing and replacement of UI components much easier.

---

## 5. Building Block View

### 5.1 Level 1 – Whitebox: Overall System

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                             websocket_swing (PoC)                                    │
│                                                                                      │
│  ┌──────────────────────┐   ┌──────────────────────┐   ┌──────────────────────────┐ │
│  │   node-vue-client    │   │    node-server       │   │    swing                 │ │
│  │  (Vue.js Web App)    │   │  (WS Relay Server)   │   │  (Java Swing Desktop)    │ │
│  └──────────────────────┘   └──────────────────────┘   └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

| Building Block     | Responsibility                                                                                         |
|--------------------|--------------------------------------------------------------------------------------------------------|
| `node-vue-client`  | Provides a modern web-based search UI. Allows searching for persons and transferring data to ALLEGRO.  |
| `node-server`      | Acts as a WebSocket message relay. Broadcasts messages from any client to all other connected clients. |
| `swing`            | The legacy ALLEGRO Java Swing application enhanced with a WebSocket client and refactored to use MVP.  |

### 5.2 Level 2 – Whitebox: `swing` Module

```
┌─────────────────────────────────────────────────────────────┐
│                        swing module                         │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  package: websocket                                  │  │
│  │  websocket/Main.java (WebSocket Client + Swing UI)   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  package: com (MVP Refactored Version)               │  │
│  │                                                      │  │
│  │  ┌────────────┐  ┌───────────────┐  ┌────────────┐  │  │
│  │  │  PocView   │  │ PocPresenter  │  │  PocModel  │  │  │
│  │  │ (Swing UI) │◄─│  (Bindings,  │──│ (Data,     │  │  │
│  │  │            │  │   Actions)    │  │  HTTP call)│  │  │
│  │  └────────────┘  └───────────────┘  └────────────┘  │  │
│  │                         │                  │         │  │
│  │                   EventEmitter      HttpBinService   │  │
│  │                  (Observer pattern) (HTTP client)    │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Class / Component    | Package                    | Responsibility                                                                                   |
|----------------------|----------------------------|--------------------------------------------------------------------------------------------------|
| `websocket/Main`     | `websocket`                | Entry point for the original PoC. Initializes the Swing UI and WebSocket client (Tyrus). Parses incoming JSON messages and updates UI fields. |
| `com/Main`           | `com`                      | Entry point for the refactored MVP version. Wires together View, Model, and Presenter.           |
| `PocView`            | `com.poc.presentation`     | Declares all Swing UI components (JFrame, JTextField, JRadioButton, JTextArea, JButton).         |
| `PocPresenter`       | `com.poc.presentation`     | Binds UI components to model properties. Handles button action. Subscribes to EventEmitter for UI updates after backend response. |
| `PocModel`           | `com.poc.model`            | Holds the form data as a `Map<ModelProperties, ValueModel<?>>`. Triggers the HTTP POST action.   |
| `HttpBinService`     | `com.poc.model`            | Sends a JSON POST request to the HTTPBin mock service at `http://localhost:8080/post`.           |
| `EventEmitter`       | `com.poc.model`            | Simple observer/callback mechanism to notify the Presenter when the backend response is ready.   |
| `EventListener`      | `com.poc.model`            | Functional interface for the event callback.                                                     |
| `ModelProperties`    | `com.poc.model`            | Enum defining all data fields of the form (e.g., `FIRST_NAME`, `LAST_NAME`, `IBAN`, etc.).      |
| `ValueModel<T>`      | `com.poc`                  | Generic wrapper holding a single typed value for a form field.                                   |

### 5.3 Level 2 – Whitebox: `node-vue-client` Module

| Component         | File                            | Responsibility                                                                                       |
|-------------------|---------------------------------|------------------------------------------------------------------------------------------------------|
| `App`             | `src/App.vue`                   | Root Vue component. Renders the application header and embeds the `Search` component.                |
| `Search`          | `src/components/Search.vue`     | Main UI component. Contains the person search form, results table, payment recipient table, and textarea. Manages WebSocket connection, search logic, and message sending. |

### 5.4 Level 2 – Whitebox: `node-server` Module

| Component         | File                            | Responsibility                                                                                       |
|-------------------|---------------------------------|------------------------------------------------------------------------------------------------------|
| `WebsocketServer` | `src/WebsocketServer.js`        | Creates an HTTP server on port 1337 and attaches a WebSocket server. Accepts connections, tracks clients, and broadcasts any received UTF-8 message to all connected clients. |

---

## 6. Runtime View

### 6.1 Scenario: Transfer Person Data from Web UI to ALLEGRO Swing Client

```
User (Browser)          Vue Search Component         Node.js WS Server         Java Swing Client
      │                         │                            │                          │
      │──── Fill search form ──►│                            │                          │
      │                         │── searchPerson() ─────────X (local search in memory) │
      │◄─── Display results ────│                            │                          │
      │                         │                            │                          │
      │──── Select person ─────►│                            │                          │
      │──── Select IBAN/BIC ───►│                            │                          │
      │                         │                            │                          │
      │─── Click "Nach ALLEGRO ►│                            │                          │
      │      übernehmen"        │                            │                          │
      │                         │── sendMessage(JSON) ──────►│                          │
      │                         │   { target: "textfield",   │                          │
      │                         │     content: { person } }  │                          │
      │                         │                            │── broadcast to all ─────►│
      │                         │                            │   clients                │
      │                         │                            │                          │── onMessage(json)
      │                         │                            │                          │   parse & populate
      │                         │                            │                          │   Swing form fields
```

### 6.2 Scenario: Swing Client Submits Data to HTTPBin

```
User (Swing)              PocPresenter              PocModel               HttpBinService
      │                         │                       │                         │
      │── Click "Anordnen" ────►│                       │                         │
      │                         │── model.action() ────►│                         │
      │                         │                       │── httpBinService.post() ►│
      │                         │                       │                         │── HTTP POST /post
      │                         │                       │                         │   (to localhost:8080)
      │                         │                       │                         │◄── HTTP 200 (JSON)
      │                         │                       │◄── return responseBody ──│
      │                         │                       │── eventEmitter.emit() ──►│
      │                         │◄── event callback ────│                         │
      │                         │── update view ────────►│                         │
      │◄── Reset form fields ───│                       │                         │
      │    + show response      │                       │                         │
```

### 6.3 Scenario: Real-time Textarea Sync (Web → Swing)

```
User (Browser)          Vue Search Component         Node.js WS Server         Java Swing Client
      │                         │                            │                          │
      │── Type in textarea ────►│                            │                          │
      │                         │── watch: internal_content  │                          │
      │                         │   _textarea                │                          │
      │                         │── sendMessage(val, "textarea") ──►│                  │
      │                         │   { target: "textarea",    │                          │
      │                         │     content: "text..." }   │                          │
      │                         │                            │── broadcast ────────────►│
      │                         │                            │                          │── textArea.setText()
```

---

## 7. Deployment View

### 7.1 Infrastructure: Local Development (PoC)

All components run on a single developer machine:

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                            Developer Workstation                                     │
│                                                                                      │
│   ┌──────────────────┐   ┌──────────────────────┐   ┌────────────────────────────┐  │
│   │  Web Browser     │   │  Terminal / Process  │   │  Terminal / Process        │  │
│   │                  │   │                      │   │                            │  │
│   │  node-vue-client │   │  node-server         │   │  swing (Java Application)  │  │
│   │  Vue.js Dev      │   │  node               │   │  java -cp ... Main         │  │
│   │  Server          │   │  WebsocketServer.js  │   │  (websocket/Main.java)     │  │
│   │  (port 8081)     │   │  (port 1337)         │   │                            │  │
│   └──────────────────┘   └──────────────────────┘   └────────────────────────────┘  │
│                                                                                      │
│   ┌──────────────────────────────────────────────────────────────────────────────┐  │
│   │  Docker Container                                                            │  │
│   │  kennethreitz/httpbin (port 8080)                                            │  │
│   └──────────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Start-up Sequence

1. Start Docker Desktop / Rancher Desktop
2. Start HTTPBin: `docker run -p 8080:80 kennethreitz/httpbin`
3. Start WebSocket relay server: `cd node-server && npm install && node src/WebsocketServer.js`
4. Start Vue.js client: `cd node-vue-client && yarn install && yarn serve`
5. Start Java Swing client: Open `websocket_swing` project in IntelliJ and run `websocket.Main`

---

## 8. Cross-cutting Concepts

### 8.1 Communication Protocol: JSON over WebSocket

All messages exchanged via the WebSocket relay follow a common JSON envelope:

```json
{
  "target": "textfield | textarea",
  "content": { /* person object or plain string */ }
}
```

- **`target: "textfield"`**: The `content` is a full person JSON object. The Swing client parses individual fields (`name`, `first`, `dob`, `zip`, `ort`, `street`, `hausnr`, `iban`, `bic`, `valid_from`) and fills the corresponding text fields.
- **`target: "textarea"`**: The `content` is a plain string that is set as the text of the Swing `JTextArea`.

### 8.2 UI Pattern: Model-View-Presenter (MVP)

The refactored Swing client (`com` package) uses the MVP pattern:

- **Model (`PocModel`)**: Holds form data and business logic (HTTP call). Completely independent of any Swing classes.
- **View (`PocView`)**: Contains only Swing UI component declarations and layout. No business logic.
- **Presenter (`PocPresenter`)**: Binds model properties to view components via `DocumentListener` and `ChangeListener`. Handles user actions (button click) by delegating to the model.

### 8.3 Event-Driven Updates

The `EventEmitter` / `EventListener` pattern decouples the Model from the Presenter for asynchronous responses:

- The `PocPresenter` subscribes a callback to the `EventEmitter` during construction.
- After a successful or failed HTTP call, `PocModel` calls `eventEmitter.emit(responseBody)`.
- The Presenter's callback is invoked, which updates the View (e.g., shows the response and resets fields).

### 8.4 Data Binding

Two-way data binding in the Swing client is simulated through `DocumentListener` (for text fields) and `ChangeListener` (for radio buttons). Changes in the UI components immediately update the corresponding `ValueModel<T>` in the `PocModel`.

### 8.5 API Contract

The REST API contract is formally documented in `api.yml` using the **OpenAPI 3.0.1** specification. The `PostObject` schema defines the same fields as `ModelProperties` in the code.

---

## 9. Architecture Decisions

### ADR-001: WebSocket Relay as Integration Mechanism

**Decision**: Use a Node.js WebSocket server as a message relay between the web client and the Java Swing client.

**Context**: The Java Swing application cannot receive HTTP requests directly (no embedded server). A push-based mechanism is needed to transfer data to the desktop application in real time.

**Alternatives considered**:
- **Shared database / file**: Polling-based, not real-time.
- **REST endpoint in Swing**: Would require embedding an HTTP server (e.g., using embedded Jetty), adding complexity.
- **gRPC / named pipes**: More complex, platform-specific.

**Rationale**: WebSocket is a well-supported, bidirectional, low-latency protocol. The relay approach is simple (< 50 lines of code) and keeps both clients symmetric—both connect _out_ to the server, avoiding firewall issues.

**Consequences**: All WebSocket messages are broadcast to all connected clients, meaning the Swing client also receives messages sent by itself (if applicable). For this PoC, this is acceptable.

---

### ADR-002: MVP Pattern for Swing Refactoring

**Decision**: Refactor the Swing application to use the Model-View-Presenter (MVP) pattern.

**Context**: The original `websocket/Main.java` is a monolithic class mixing UI layout, WebSocket logic, JSON parsing, and event handling. This makes it hard to test or extend.

**Rationale**: MVP separates concerns, making each component independently testable. The `PocModel` has no dependency on Swing and could be unit-tested without a display. The `PocView` is a passive container of UI components.

**Consequences**: The code is split across more files, but each class has a single clear responsibility.

---

### ADR-003: Hard-coded Mock Data in Vue Client

**Decision**: Person data for the search is embedded as a JavaScript array in `Search.vue`.

**Context**: A real backend search service is out of scope for this PoC.

**Rationale**: Keeps the PoC self-contained and runnable without any external dependencies beyond the WebSocket server and HTTPBin.

**Consequences**: Data cannot be updated without code changes. This must be replaced by a real API call in production.

---

## 10. Quality Requirements

### 10.1 Quality Tree

```
Quality
├── Interoperability
│   └── Web-to-Swing data transfer works correctly via WebSocket
├── Usability
│   └── Person search with partial match on name, address, city, or ZIP
├── Maintainability
│   ├── MVP pattern separates UI, logic, and data in Swing client
│   └── Vue component-based architecture separates UI concerns
└── Portability
    └── Runs on any OS with Java >= 22, Node.js, Docker
```

### 10.2 Quality Scenarios

| ID  | Quality Attribute  | Stimulus                                             | Response                                                                 | Measure                  |
|-----|--------------------|------------------------------------------------------|--------------------------------------------------------------------------|--------------------------|
| Q1  | Interoperability   | User clicks "Nach ALLEGRO übernehmen" in the browser.| The correct person data appears in all Swing text fields within 1 second.| < 1s end-to-end latency. |
| Q2  | Usability          | User enters a partial last name in the search form.  | All matching persons are shown in the results table.                     | Correct partial-match results. |
| Q3  | Maintainability    | Developer adds a new form field.                     | Only `ModelProperties`, `PocView`, `PocPresenter`, and `api.yml` need changing. | ≤ 4 files changed.      |

---

## 11. Risks and Technical Debt

| ID  | Risk / Debt                                      | Impact  | Probability | Mitigation / Note                                                                                 |
|-----|--------------------------------------------------|---------|-------------|---------------------------------------------------------------------------------------------------|
| R1  | **No authentication or authorization**           | High    | High        | Any client connecting to port 1337 can read all messages. Must be addressed before production use. |
| R2  | **WS server broadcasts to all clients**          | Medium  | High        | In a multi-user environment, data would leak to all connected Swing instances. Requires client identification and targeted routing. |
| R3  | **Hard-coded mock data in Vue client**           | Medium  | High (PoC)  | Must be replaced by a real backend search API call.                                              |
| R4  | **No error handling in Node.js server**          | Medium  | Medium      | Invalid JSON or unexpected disconnections are not gracefully handled.                             |
| R5  | **`websocket/Main.java` is a monolith**          | Low     | Low (PoC)   | The original version mixes UI, WebSocket, and JSON parsing. The MVP refactor (`com/Main`) addresses this; the old file should be removed eventually. |
| R6  | **`PocModel.action()` calls `toString()` on null** | Medium | Medium      | If any model property is `null` and `toString()` is called, a `NullPointerException` could occur. |
| R7  | **Dependency versions are outdated**             | Medium  | Medium      | `tyrus-standalone-client:1.15`, `javax.json:1.1.4` – security patches may exist for newer versions. Check CVE databases before any production use. |

---

## 12. Glossary

| Term                         | Definition                                                                                                                        |
|------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| **ALLEGRO**                  | The legacy Java Swing desktop application being modernized. The name of the target application in the PoC context.               |
| **PoC (Proof of Concept)**   | A prototype implementation to validate a technical or architectural approach before full-scale development.                       |
| **arc42**                    | A template and framework for software architecture documentation. See https://arc42.org.                                          |
| **WebSocket**                | A communication protocol providing full-duplex channels over a single TCP connection. Standardized in RFC 6455.                   |
| **MVP (Model-View-Presenter)**| An architectural pattern for UI applications separating data (Model), layout (View), and logic (Presenter).                      |
| **Zahlungsempfänger**        | German for "payment recipient". Represents banking data (IBAN, BIC, valid-from date) associated with a person.                   |
| **HTTPBin**                  | An open-source HTTP request/response service used as a mock backend. Returns what was sent to it.                                 |
| **Tyrus**                    | The Reference Implementation of JSR-356 (Java WebSocket API) used as the WebSocket client library in the Swing application.      |
| **JSR-356**                  | Java Specification Request 356 – Java API for WebSocket.                                                                         |
| **Vue.js**                   | A progressive JavaScript framework for building web user interfaces.                                                              |
| **Node.js**                  | A JavaScript runtime built on Chrome's V8 engine, used here to run the WebSocket relay server.                                   |
| **EventEmitter**             | A component implementing the Observer pattern to notify the Presenter of asynchronous model events.                               |
| **OpenAPI**                  | A standard specification format for REST APIs (formerly Swagger). The API contract is defined in `api.yml`.                      |
| **Vorname / Name**           | German for "first name" / "last name".                                                                                            |
| **PLZ / Ort / Strasse**      | German for "ZIP code" / "city" / "street".                                                                                        |
| **Geburtsdatum**             | German for "date of birth".                                                                                                       |
| **Anordnen**                 | German for "arrange" / "submit". The label for the action button in the Swing client.                                             |
