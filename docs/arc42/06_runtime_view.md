# 6. Runtime View

## 6.1 Scenario 1 – HTTP Form Submission (Swing HTTP Client)

This scenario describes the user filling in a form in the Swing HTTP client and submitting it to the HTTPBin echo service.

```
User        PocView       PocPresenter      PocModel      HttpBinService    HTTPBin
  │              │               │               │                │            │
  │  types text  │               │               │                │            │
  │─────────────►│               │               │                │            │
  │              │ DocumentChange│               │                │            │
  │              │───────────────►               │                │            │
  │              │               │ setValue()    │                │            │
  │              │               │──────────────►│                │            │
  │              │               │               │                │            │
  │ clicks submit│               │               │                │            │
  │─────────────►│               │               │                │            │
  │              │ actionPerformed               │                │            │
  │              │───────────────►               │                │            │
  │              │               │ action()      │                │            │
  │              │               │──────────────►│                │            │
  │              │               │               │ post(formData) │            │
  │              │               │               │───────────────►│            │
  │              │               │               │                │  POST /post│
  │              │               │               │                │───────────►│
  │              │               │               │                │  response  │
  │              │               │               │                │◄───────────│
  │              │               │               │◄───────────────│            │
  │              │               │               │ emit(response) │            │
  │              │               │◄──────────────│(EventEmitter)  │            │
  │              │ clearFields() │               │                │            │
  │              │◄──────────────│               │                │            │
  │              │               │               │                │            │
```

**Key Steps:**
1. User types into any text field → `PocPresenter` `DocumentListener` fires → `ValueModel` updated.
2. User clicks the submit button → `PocPresenter` calls `PocModel.action()`.
3. `PocModel` collects all `ValueModel` values and calls `HttpBinService.post()`.
4. `HttpBinService` sends `HTTP POST` to `http://localhost:8080/post` with JSON payload.
5. HTTPBin returns the echoed JSON response.
6. `PocModel` invokes `EventEmitter.emit(responseBody)`.
7. `PocPresenter` listener clears all form fields.

---

## 6.2 Scenario 2 – Customer Selection via Vue.js and WebSocket

This scenario describes an operator searching for a customer in the Vue.js browser client and pushing the data to the Java Swing WebSocket client.

```
Operator    Vue.js Client   Node.js Server   Swing WS Client
  │               │                │                │
  │  types search │                │                │
  │──────────────►│                │                │
  │               │ searchPerson() │                │
  │               │ (in-memory)    │                │
  │               │                │                │
  │  clicks row   │                │                │
  │──────────────►│                │                │
  │               │ selectResult() │                │
  │               │                │                │
  │  clicks send  │                │                │
  │──────────────►│                │                │
  │               │ sendMessage()  │                │
  │               │ {target:"textfield",            │
  │               │  content:JSON} │                │
  │               │───────────────►│                │
  │               │                │ broadcast      │
  │               │                │───────────────►│
  │               │                │                │ onMessage()
  │               │                │                │ parse JSON
  │               │                │                │ populate fields
  │               │                │                │
```

**Key Steps:**
1. Operator types search criteria in Vue.js form → `searchPerson()` filters in-memory data.
2. Operator clicks a result row → `selectResult()` stores selected customer.
3. Operator clicks "Nach ALLEGRO übernehmen" → `sendMessage()` sends `{ target: "textfield", content: <customerJSON> }` via WebSocket.
4. Node.js server receives the message and broadcasts it to all connected clients.
5. `WebsocketClientEndpoint.onMessage()` in the Swing client receives the message, parses the JSON, and routes by `target`:
   - `"textfield"` → populates all Swing form fields using `SearchResult` DTO.
   - `"textarea"` → updates the text area field.

---

## 6.3 Scenario 3 – Textarea Real-Time Sync (Vue.js → Swing)

```
Operator    Vue.js Client   Node.js Server   Swing WS Client
  │               │                │                │
  │  types in     │                │                │
  │  textarea     │                │                │
  │──────────────►│                │                │
  │               │ watch fires    │                │
  │               │ sendMessage(   │                │
  │               │  "textarea")   │                │
  │               │───────────────►│                │
  │               │                │ broadcast      │
  │               │                │───────────────►│
  │               │                │                │ update textarea
  │               │                │                │ field in GUI
  │               │                │                │
```

**Key Steps:**
1. Operator types in the Vue.js textarea component.
2. Vue.js watcher detects change → `sendMessage(value, "textarea")` via WebSocket.
3. Node.js broadcasts to all clients.
4. Swing WebSocket client receives message; `target === "textarea"` → updates the Swing `JTextArea`.
