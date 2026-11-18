---
id: background-jobs
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Infrastructure standards: Background jobs

How an application runs work that does not finish inside the request that started it: a job is recorded, run by a hosted service, and its status is read back by the caller. These are the ground truth rules for the job record, its lifecycle, retries and resumption. When work is deferred is owned by `api` (synchronous by default, 202 with a handle when deferred) and by `coding-standards-all-languages` (work beyond about 3 seconds is off-loaded); this document says how the deferred work is built.

## Job philosophy

- **A job is data before it is work.** The request that starts it only writes the job record and returns. Everything the worker needs is in that record, so the work survives a restart, a deploy or a scale-out.
- **The database is the queue inside one application.** Jobs of one application live in a table of its own database, run by its own hosted services (`default-stack`). No extra service is added for it (fewer moving parts, `technology-choices`). Work handed to another application goes through Service Bus as `inter-service-communication` describes, not through this table.
- **Every job ends in a state someone can see.** A job never disappears silently: it succeeds, fails with a reason, or is cancelled, and the caller can read which.

## The job record

- **Each application has one `job` table** in its own schema and migrations, following `database` for ids, audit columns and naming. It holds at least: the job id, the job type, one coded status, the input payload, the step reached, progress, the result or a reference to it, the error, the attempt count, the earliest time it may run, the claim expiry, the principal it runs for, the trace parent, and created, started and finished times.
- **One status, from a fixed set:** `queued`, `running`, `succeeded`, `failed`, `cancelled`. It is a coded value and a contract (`database`), so a new status is added, never renamed.
- **The input is enough to run the job.** The payload holds ids and parameters, not copies of records that may change; the worker reads the current data when it runs. Its size is bounded like any input (`security`).
- **Large results go to Blob Storage.** The job row holds a reference, never the file (`default-stack`).
- **The principal is recorded.** The job runs with the rights of the user or service that started it, checked again when the worker acts, not only when the job was created.

## Starting a job

- **The job is written in the same transaction as the change that needs it.** A use case that changes state and starts a job commits both or neither, like the outbox (`database`).
- **The response is 202 with a handle and a `Location` to the status resource** (`api`).
- **A start that a client may retry carries an idempotency key** (`api`), stored on the job, so a retried request returns the existing job instead of creating a second one.

## Running a job

- **Workers claim jobs, they do not lock the table.** A worker claims a small batch of due `queued` jobs with `UPDLOCK, READPAST, ROWLOCK` (as the outbox drain does in `database`), sets them to `running` and sets a claim expiry. Concurrent workers on other instances skip claimed rows.
- **A running job renews its claim while it works.** A job whose claim has expired was abandoned by a crashed or stopped instance and may be claimed again. The claim expiry is longer than the longest single step.
- **A job of several steps records each finished step.** It saves the step reached and the intermediate state it needs after every step, so a reclaimed job resumes at the next step instead of repeating finished ones.
- **Every step is safe to repeat.** A step can run twice when an instance stops after the step but before its record is saved, so steps are idempotent (`coding-standards-all-languages`); a step with an outside effect (a mail, a payment, a call to another system) uses an idempotency key towards it.
- **Progress is written at step boundaries,** as a step name and, where it can be known, a percentage. It is what the caller sees; it is not written more often than it is useful.
- **Each unit of work gets its own DI scope** (`coding-standards-csharp`), and the trace parent stored on the job continues the trace of the request that started it (`observability`).
- **Graceful shutdown.** On stop, a worker finishes or abandons its current step within the host's shutdown timeout and leaves the job claimable; it never marks unfinished work as done.

## Failure and retry

- **Transient failures are retried with backoff.** The attempt count goes up, the earliest run time moves out (exponential with jitter), and the job goes back to `queued`.
- **Retries have a limit per job type.** After the last attempt, or at once for an error that a retry cannot fix (invalid input, missing rights), the job is `failed` with a stable error code and a message without personal data or internals (`api`).
- **A failed job is visible and acted on.** Failed jobs count towards an alert by business criticality (`observability`), and an operator or the user can retry one, which starts a new attempt, not a new job.
- **Cancellation is a request, not a kill.** Cancelling sets a flag the worker checks between steps; the job ends `cancelled` at the next step boundary, and finished steps with outside effects are not undone unless the job type defines how.

## Reading the status

- **The status resource returns the status, step, progress, timestamps and, when finished, the result or its location, or the error.** It is authorized like the work it represents: only the principal who started it, or a role allowed to see that work, can read it.
- **Clients poll with a sensible interval** that the response suggests with `Retry-After`; a screen may instead be told through a push channel when the application has one. Polling is the default because it needs nothing extra.

## Housekeeping

- **Finished jobs are kept for a set retention per job type,** then deleted by a singleton hosted service under a lease (`app-service`). The retention follows the data the job holds; payloads with personal data are recorded in the data register (`project-records`).
- **Scheduled work creates jobs.** A timer that must run once per schedule does so as a singleton under a lease (`app-service`) and creates a job for the actual work, so scheduled work gets the same retries, status and history.

## Review and ownership

- **Owners:** the vault owners maintain this document.
