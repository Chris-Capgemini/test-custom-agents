# 1. Introduction and Goals

## 1.1 Requirements Overview

The **Allegro Modernization PoC** is a proof-of-concept application demonstrating an approach for modernizing the Allegro system. It enables operators to:

- **Search** for customer and person records using multiple criteria (name, address, ZIP code, city).
- **View** search results in a tabular list with sorting and selection.
- **Select** payment beneficiary accounts (IBAN, BIC, valid-from date) linked to a customer.
- **Submit** selected customer and payment data to the Allegro backend.
- **Display** server responses in real time.

The PoC validates two complementary integration approaches between a desktop GUI and a backend system:

| Approach | Client | Protocol | Backend |
|---|---|---|---|
| HTTP-based | Java Swing | REST/HTTP POST | HTTPBin echo service |
| WebSocket-based | Java Swing + Vue.js | WebSocket | Node.js relay server |

## 1.2 Quality Goals

| Priority | Quality Goal | Motivation |
|---|---|---|
| 1 | **Functionality** | The PoC must successfully demonstrate end-to-end data submission from GUI to backend. |
| 2 | **Interoperability** | Desktop (Java Swing) and web (Vue.js) clients must be able to exchange data via WebSocket. |
| 3 | **Modifiability** | The MVP architecture (Model–View–Presenter) must keep business logic and UI decoupled to ease future changes. |
| 4 | **Simplicity** | The system is a PoC; complexity should be kept minimal to reduce development time. |

## 1.3 Stakeholders

| Role | Concern |
|---|---|
| **Developers** | Need a clear, reproducible local setup and a well-structured codebase. |
| **Architects** | Need an understanding of the chosen patterns and integration approaches. |
| **Business Analysts** | Need to understand the mapped business domain (customer data, payment beneficiaries). |
| **Operations** | Need to know which services must be running and how they are started. |
