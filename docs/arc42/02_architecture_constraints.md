# 2. Architecture Constraints

## 2.1 Technical Constraints

| ID | Constraint | Background / Rationale |
|---|---|---|
| TC-01 | **Java >= 22.0.1** | The Swing clients use modern Java APIs (e.g., Records, enhanced switches). The Maven POM targets Java 22. |
| TC-02 | **Maven as build tool** | Dependency management and compilation of the Java Swing modules are done via Maven (`pom.xml`). |
| TC-03 | **Node.js WebSocket server** | The relay server uses the `websocket` npm package. Node.js must be installed to run the server. |
| TC-04 | **Vue 2.x for the web client** | The Vue.js web client is built with Vue CLI 4 targeting Vue 2.6. Migration to Vue 3 is out of scope for the PoC. |
| TC-05 | **Docker for HTTPBin** | The HTTP echo backend (`kennethreitz/httpbin`) runs as a Docker container on port 8080. Docker Desktop or Rancher Desktop must be available on the developer machine. |
| TC-06 | **Localhost-only deployment** | All service URLs are hard-coded to `localhost`. No remote deployment is supported in the current PoC. |
| TC-07 | **IntelliJ IDEA** | The project launch configuration (`WebsocketSwingClient.launch`) targets IntelliJ IDEA. Other IDEs may require manual configuration. |

## 2.2 Organizational Constraints

| ID | Constraint | Background / Rationale |
|---|---|---|
| OC-01 | **PoC scope** | This is a proof of concept only. Production-readiness (security hardening, scalability, monitoring) is explicitly out of scope. |
| OC-02 | **Single repository** | All components (Java backend, Node.js server, Vue.js client) are kept in one repository for simplicity. |

## 2.3 Conventions

| ID | Convention | Background / Rationale |
|---|---|---|
| CV-01 | **MVP pattern for Java GUI** | The Java Swing HTTP client follows the Model–View–Presenter pattern to separate UI rendering from business logic. |
| CV-02 | **Observer pattern for event propagation** | The `EventEmitter` / `EventListener` pair is used to propagate backend responses without tight coupling. |
| CV-03 | **JSON for all inter-process communication** | Data exchanged between clients and the Node.js relay server is serialized as JSON. |
