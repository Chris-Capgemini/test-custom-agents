# 11. Technical Risks

## 11.1 Risk Summary

| ID | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R-01 | Hard-coded localhost URLs prevent deployment outside developer machine | High | High | Externalize URLs into configuration files or environment variables before production use |
| R-02 | No error handling for failed HTTP / WebSocket connections | High | Medium | Add proper exception handling and user-facing error messages |
| R-03 | Mock customer data (5 records) is not representative of real Allegro data | High | High | Replace `search_space` with a real backend search API |
| R-04 | Manual JSON parsing in Java is fragile and hard to maintain | Medium | Medium | Replace with a standard JSON-binding library (e.g., Jackson or Gson) |
| R-05 | Node.js relay server broadcasts to ALL clients; no access control | High | Medium | Add authentication and per-client message routing for production |
| R-06 | Vue 2 has reached end-of-life (EOL December 2023) and is no longer maintained | High | Medium | Migrate to Vue 3 or an alternative framework before production |
| R-07 | `ViewData` class is empty and unused, indicating incomplete design | Low | Low | Remove or implement as intended |
| R-08 | No automated tests (unit, integration, or E2E) | High | High | Add test coverage using JUnit (Java) and Jest/Cypress (Vue.js) |
| R-09 | Tyrus WebSocket client version 1.13 may have known vulnerabilities | Medium | Medium | Check for and apply security updates before production use |
| R-10 | HTTPBin Docker image (`kennethreitz/httpbin`) is not actively maintained | Medium | Low | Replace with a custom or maintained echo endpoint for production |

## 11.2 Technical Debt

| Item | Location | Description |
|---|---|---|
| Empty class | `com.poc.model.ViewData` | Class body is empty; either remove or implement intended functionality |
| Hard-coded data | `Search.vue` (`search_space`) | Mock customer records embedded in component; must be replaced by a real data source |
| Hard-coded URLs | `HttpBinService.java`, `websocket/Main.java`, `Search.vue` | Service endpoints are not configurable |
| Manual JSON parsing | `HttpBinService.java`, `websocket/Main.java` | Custom streaming JSON parsing is error-prone; use a JSON library |
| No input validation | All clients | Form fields accept any input without validation |
| No security | All components | No authentication, authorization, or transport encryption (TLS/WSS) |
| German UI labels only | `PocView.java`, `Search.vue` | UI is not internationalized |
