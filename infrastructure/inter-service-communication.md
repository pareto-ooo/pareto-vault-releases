---
id: inter-service-communication
scope: scoped
severity: warning
audience: all
applies-to:
  - "**/*.Api/**/*.cs"
  - "**/*Controller.cs"
  - "**/*Subscriber*.cs"
  - "**/*Publisher*.cs"
schemaVersion: 1
---

# Inter-service communication

How apps talk to each other. Authentication, schema validation and idempotency are covered by the security, API and database standards.

## Choosing the shape

- **The cheapest cross-app call is the one not made.** First ask whether the data can live in this app's own database, be cached, or be left out. Two apps that call each other constantly are probably one app.
- **HTTP is for queries.** The caller needs a fresh answer now, and knows who to ask.
- **Service Bus topics are for facts.** The publisher announces something that already happened and does not know or care who listens. Zero or five subscribers tomorrow makes no difference to it.
- **A need for both is two interactions**, modelled as two calls.

## HTTP calls

- **Timeouts and retries are the caller's responsibility.** A non-idempotent call is never retried without an idempotency key.
- **A failing callee degrades the caller only when it must.** A non-critical feature falls back to cached or partial data, or to a local read model filled from events, and never takes the caller down with it.

## Events

- **Events carry identifiers, not state.** The `data` of an event holds only identifiers. Subscribers read current state from the owning app's API.
- **Every event has the same envelope,** as the message body:

  ```json
  {
    "eventId": "0b9f…",
    "eventType": "OrderPlaced",
    "schemaVersion": 1,
    "schemaUri": "https://…/order-placed.v1.json",
    "occurredAt": "2026-10-27T09:30:00Z",
    "aggregateType": "Order",
    "aggregateId": "4c1e…",
    "data": { "orderId": "4c1e…", "customerId": "9a07…" }
  }
  ```

  `eventId` is unique per event and is also the Service Bus message id, so subscribers and duplicate detection deduplicate on it. `occurredAt` is UTC. The outbox stores the envelope in columns (`database`).
- **Every topic has a checked-in JSON Schema**, and the event references it (`schemaUri`).
- **Breaking event changes go to a new topic version** (`<topic>.v2`), additive changes stay on the topic. A CI step blocks breaking changes to the original version.
- **Service Bus access is RBAC** (Data Sender, Data Receiver) per managed identity. No connection strings.

## Never

- Service Bus as request/response (publish a request, wait for a response event). That is HTTP with extra latency, no error mapping and no ordering.
- HTTP calls used to notify several services in a row. Publish one event.
- Chains of synchronous calls on the request path. Latencies add and failures multiply.
