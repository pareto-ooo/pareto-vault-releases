---
id: observability
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Infrastructure standards: Observability

How a deployed application shows whether it works: telemetry, trace propagation, alerts, availability tests and dashboards. These are the ground truth rules for every application on the default stack. What may never be logged is owned by `security`, log routing by `governance`, log retention by `compliance`, and availability targets by `sizing`; they are not repeated here.

## Observability philosophy

- **We know before the user tells us.** A failure that hurts users raises an alert. The first report of an outage never comes from a user.
- **One request, one trace.** A user action can be followed from the browser through the API, its database and the events it publishes, to every subscriber, by one trace id.
- **An alert means someone acts now.** A condition that needs no action is a dashboard line, not an alert. Alerts that are routinely ignored are fixed or removed.
- **Telemetry costs money.** Ingestion is a cost driver (`sizing`), so what is collected is a choice.

## Telemetry

- **Each application has one workspace-based Application Insights resource per environment,** connected to the workload's Log Analytics workspace (`governance`) and instrumented with the Azure Monitor OpenTelemetry distribution (`default-stack`). The frontend and the backend of one application share it, so a trace crosses both.
- **The single-page app uses the Application Insights JavaScript SDK** with the same connection string, for page views, failed API calls and unhandled errors (`coding-standards-vuejs`). Its cookies are disabled, so telemetry needs no consent, and it records no form input or page content (`security`).
- **Trace context propagates with W3C Trace Context on every hop.** The single-page app sends `traceparent` on every API call, the API continues the trace into its outgoing calls and Service Bus messages, and a subscriber continues it from the message. The `traceId` in an error response (`api`) is this trace id.
- **An event keeps the trace of the request that caused it.** The outbox row stores the trace context (`database`), and the publisher sends it as the message's `traceparent` property, outside the event envelope.
- **Background work is visible.** Each run of a scheduled or singleton hosted service records its start, outcome and duration. The age of the oldest unpublished outbox row and the number of dead-lettered messages are metrics.
- **A few business metrics show that the application does its job,** such as orders placed or payments failed. They are custom metrics with identifiers only, never personal data.
- **Sampling starts at 100%.** It is lowered only when ingestion cost requires it, non-production first, and the rate is set per environment in configuration.
- **Non-production has a daily ingestion cap.** Production has an alert on ingestion volume instead of a cap, because a cap drops the telemetry an incident needs.

## Alerts

- **Alert rules and action groups are infrastructure code,** per environment, next to the resources they watch.
- **What alerts in production follows the business criticality:**

  | Business criticality | Alerts | Who is notified |
  |----------------------|--------|-----------------|
  | `NonCritical` | Availability test failing | The team channel |
  | `BusinessSupport` | The above, plus failed request rate and exceptions above their normal level | The team channel |
  | `BusinessCritical` | The above, plus the response time target of the workload profile, dead-lettered messages and outbox backlog age | The incident contact (`project-records`) during agreed support hours, the team channel otherwise |
  | `MissionCritical` | The same as `BusinessCritical` | The incident contact, at any hour |

- **Non-production alerts go to the team channel only.** They never page anyone.
- **Every alert names its runbook** (`project-records`), so whoever is notified knows the first step.

## Availability tests

- **Every deployed web application has a standard availability test** in Application Insights, from several locations: on the frontend's public URL and on the API's `/health/ready` (`app-service`). The test of a custom domain checks the certificate's remaining lifetime (`security`).
- **Availability tests run in production and acceptance.** Development and test environments skip them, unless they test the tests.

## Dashboards

- **Each application has one workbook, in infrastructure code,** with request rate, failure rate, response time at the percentile of the workload profile, dependency failures, and background backlog. It is the first screen in an incident.

## Review and ownership

- **Owners:** the vault owners maintain this document.
