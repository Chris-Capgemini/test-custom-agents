# 9. Architecture Decisions

## ADR-001 – Use of MVP Pattern for Swing HTTP Client

**Status:** Accepted

**Context:**  
The Swing HTTP client needs to present a form UI and submit data to a backend. Without a clear separation of concerns, UI and business logic would be intermingled, making the code harder to maintain and test.

**Decision:**  
Apply the Model–View–Presenter (MVP) pattern:
- `PocView` renders the UI only.
- `PocPresenter` binds UI events to the model.
- `PocModel` holds state and executes business logic.

**Consequences:**
- (+) Clear separation of concerns.
- (+) `PocModel` and `HttpBinService` can be tested without the Swing UI.
- (−) More classes to manage for a PoC.

---

## ADR-002 – Use of Observer Pattern (EventEmitter) for Response Notification

**Status:** Accepted

**Context:**  
After an HTTP submission completes, the presenter needs to be notified to clear the form. A direct callback would couple `PocModel` to `PocPresenter`.

**Decision:**  
Introduce `EventEmitter` and `EventListener` to decouple the notification mechanism from the subscriber.

**Consequences:**
- (+) `PocModel` does not need to know about `PocPresenter`.
- (+) Multiple listeners can be registered without changing the model.
- (−) Slight indirection that may be overkill for a PoC with a single listener.

---

## ADR-003 – Node.js as WebSocket Relay Server

**Status:** Accepted

**Context:**  
A bridge is needed to allow the Vue.js browser client to communicate with the Java Swing desktop client. Browser-to-desktop communication is not directly possible.

**Decision:**  
Use a simple Node.js WebSocket relay server that broadcasts every received message to all connected clients.

**Consequences:**
- (+) Minimal implementation effort.
- (+) Language-agnostic; any client supporting WebSocket can connect.
- (−) All clients receive all messages (no targeted delivery beyond the `target` field in the payload).
- (−) No persistence; all state is lost on server restart.

---

## ADR-004 – HTTPBin as Mock Backend for HTTP Scenario

**Status:** Accepted

**Context:**  
The actual Allegro backend is not available for the PoC. An echo service is needed to validate that the HTTP submission works end-to-end.

**Decision:**  
Use the `kennethreitz/httpbin` Docker image as the HTTP backend. It echoes the request body as a JSON response.

**Consequences:**
- (+) Zero implementation effort for the backend.
- (+) Docker image is widely available.
- (−) Not representative of a real Allegro API response format.
- (−) Requires Docker to be running on the developer machine.

---

## ADR-005 – Hard-Coded Localhost URLs

**Status:** Accepted (for PoC only)

**Context:**  
All service URLs (HTTPBin, WebSocket server) need to be known at compile/runtime. For a PoC, the overhead of configuration management is not justified.

**Decision:**  
Hard-code all URLs as `localhost` with fixed ports in the source code.

**Consequences:**
- (+) Simple and fast to implement.
- (−) Cannot be deployed to remote environments without code changes.
- (−) Must be refactored before any production use.

---

## ADR-006 – Mock Customer Data in Vue.js Client

**Status:** Accepted (for PoC only)

**Context:**  
No customer database or search API exists in the PoC. The Vue.js client still needs realistic data to demonstrate the search-and-select workflow.

**Decision:**  
Embed a hard-coded array of 5 customer records with associated payment accounts directly in the Vue.js component (`search_space`).

**Consequences:**
- (+) No backend search service required.
- (+) Deterministic, reproducible demo data.
- (−) Data cannot be modified at runtime.
- (−) Must be replaced with a real data source in production.
