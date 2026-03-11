# 8. Cross-Cutting Concepts

## 8.1 Domain Model

The domain is centered around **customer (person) data** and **payment beneficiary accounts**.

### Person / Customer

| Field | Type | Description |
|---|---|---|
| `FIRST_NAME` | String | First name |
| `LAST_NAME` | String | Last name |
| `DATE_OF_BIRTH` | String | Date of birth |
| `STREET` | String | Street address |
| `ORT` | String | City |
| `ZIP` | String | Postal / ZIP code |
| `MALE` | Boolean (radio) | Gender: male |
| `FEMALE` | Boolean (radio) | Gender: female |
| `DIVERSE` | Boolean (radio) | Gender: diverse |

### Payment Beneficiary (Zahlungsempfänger)

| Field | Type | Description |
|---|---|---|
| `IBAN` | String | International Bank Account Number |
| `BIC` | String | Bank Identifier Code |
| `VALID_FROM` | String | Date from which the account is valid |

### Other

| Field | Type | Description |
|---|---|---|
| `TEXT_AREA` | String | Free-text area for additional notes or responses |

---

## 8.2 JSON Message Format (WebSocket)

All WebSocket messages follow a shared envelope format:

```json
{
  "target": "<routing key>",
  "content": "<payload string or serialized JSON>"
}
```

| Field | Values | Description |
|---|---|---|
| `target` | `"textfield"` | Route content to form fields (customer data as JSON) |
| `target` | `"textarea"` | Route content to the text area component |
| `content` | JSON string or plain text | Payload to be placed in the targeted GUI component |

When `target` is `"textfield"`, the `content` field is a serialized JSON object using `ModelProperties` keys:

```json
{
  "FIRST_NAME": "Max",
  "LAST_NAME": "Mustermann",
  "DATE_OF_BIRTH": "1980-01-01",
  "STREET": "Musterstraße 1",
  "ZIP": "12345",
  "ORT": "Musterstadt",
  "IBAN": "DE89370400440532013000",
  "BIC": "COBADEFFXXX",
  "VALID_FROM": "2024-01-01"
}
```

---

## 8.3 Error Handling

| Layer | Current Handling | Notes |
|---|---|---|
| `HttpBinService` | Reads response body; no exception propagation to UI | Errors silently fail |
| `WebsocketClientEndpoint` | No explicit exception handling | Malformed JSON will cause a runtime exception |
| Vue.js `connect()` | No WebSocket error handler defined | Connection failures are silent |
| Node.js server | Logs connection and close events to console | No retry or dead-letter logic |

> **Note:** Error handling is minimal in this PoC. Production use would require proper error propagation and user-facing messages.

---

## 8.4 Logging

| Component | Logging Mechanism |
|---|---|
| Node.js Server | `console.log()` with timestamps |
| Java Swing clients | No logging; `System.out.println()` for some debug output |
| Vue.js client | Browser developer console |

---

## 8.5 Build and Dependency Management

| Component | Tool | Configuration File |
|---|---|---|
| Java Swing clients | Maven | `websocket_swing/pom.xml` |
| Node.js WS Server | npm | `websocket_swing/node-server/package.json` |
| Vue.js Web Client | Vue CLI (Yarn / npm) | `websocket_swing/node-vue-client/package.json` |

**Java Dependencies (Maven):**

| Dependency | Version | Purpose |
|---|---|---|
| `org.glassfish.tyrus.bundles:tyrus-standalone-client` | 1.13 | WebSocket client implementation (JSR 356) |
| `javax.json` (via Java EE) | Bundled | JSON parsing and generation |

**Node.js Dependencies:**

| Package | Version | Purpose |
|---|---|---|
| `websocket` | 1.0.35 | WebSocket server and client implementation |

---

## 8.6 User Interface Conventions

- The Swing UI uses a `GridBagLayout` for form layout.
- Labels and button text are in German (e.g., "Anordnen", "Nach ALLEGRO übernehmen"), reflecting the business domain language.
- The Vue.js UI uses a three-panel layout: search form → results table → payment account table.
