# 12. Glossary

| Term | Definition |
|---|---|
| **Allegro** | The target legacy backend system that this PoC aims to integrate with and modernize. |
| **arc42** | A standard template for software architecture documentation. See https://arc42.org/. |
| **BIC** | Bank Identifier Code. An international standard for identifying banks, used in international wire transfers. |
| **Broadcast** | Sending a message to all connected clients simultaneously. Used by the Node.js relay server. |
| **EventEmitter** | A component that manages event listeners and notifies them when an event occurs (Observer pattern). |
| **EventListener** | A functional interface defining the `onEvent(String)` callback invoked by `EventEmitter`. |
| **GridBagLayout** | A flexible Swing layout manager that places components in a grid with variable row and column sizes. |
| **HTTPBin** | An HTTP echo service (`kennethreitz/httpbin`) that returns the details of the HTTP request it receives, used as a mock backend. |
| **IBAN** | International Bank Account Number. A standardized format for identifying bank accounts across countries. |
| **IntelliJ IDEA** | The Java IDE recommended for running and developing the Swing clients in this project. |
| **JSR 356** | The Java API for WebSocket, providing a standard for WebSocket communication in Java applications. |
| **JSON** | JavaScript Object Notation. A lightweight text-based data interchange format used for all inter-process communication in this project. |
| **ModelProperties** | A Java enum defining all form field keys used throughout the Swing HTTP client. |
| **MVP (Model–View–Presenter)** | An architectural pattern separating an application into three components: Model (data/business logic), View (UI rendering), and Presenter (mediator/controller). |
| **Node.js** | A JavaScript runtime used to implement the WebSocket relay server. |
| **Observer Pattern** | A design pattern where an object (subject) maintains a list of dependents (observers) and notifies them of state changes. Used via `EventEmitter`/`EventListener`. |
| **PoC (Proof of Concept)** | A small, time-boxed project to validate the feasibility of an approach. This repository is a PoC. |
| **Relay Server** | A server that receives messages and forwards them to other clients without processing the content. |
| **Swing** | Java's GUI toolkit for building desktop applications. |
| **Tyrus** | The reference implementation of JSR 356 (Java WebSocket API) used as a standalone WebSocket client library. |
| **ValueModel** | A generic wrapper class (`ValueModel<T>`) used to hold and retrieve individual form field values in the Swing HTTP client. |
| **Vue.js** | A progressive JavaScript framework for building web UIs. The web client in this project uses Vue 2. |
| **WebSocket** | A communication protocol providing full-duplex communication channels over a single TCP connection. Used in the second integration scenario. |
| **ws://** | Unencrypted WebSocket URL scheme (as opposed to `wss://` for TLS-encrypted WebSocket). |
| **Yarn** | A JavaScript package manager used to manage dependencies and run scripts in the Vue.js project. |
| **Zahlungsempfänger** | German for "payment beneficiary". Refers to the bank account information (IBAN, BIC) associated with a customer. |
