---
id: ai-features
scope: global
severity: warning
audience: all
schemaVersion: 1
---

# Architecture standards: AI features and agents

How an application we build uses a language model: a single model call inside a use case, or an agent that takes several steps and calls the application's own functions as tools. These are the ground truth rules for the model integration, what an agent may do, how its output is trusted, and how it is tested, logged and limited. AI assistants used while developing are owned by `compliance` (AI assistants in development), not by this document.

## AI philosophy

- **The model is an outside dependency, not part of the domain.** It is reached through an adapter in the infrastructure layer (`coding-standards-all-languages`), so the provider, model and prompt can change without changing the use cases.
- **An agent is a user of the application, not a way around it.** It acts through the same application layer, rights check and audit as a screen, never with more rights than the person it acts for.
- **Nothing a model returns or reads is trusted.** Its output is input to validate, and the content it reads is data, never instruction.
- **The rights limit what an agent can do, not the prompt.** A prompt asks for good behaviour; the tools and rights an agent is given decide what it can actually do.

## Provider and model

- **The model service is chosen like any service** (`technology-choices`): Microsoft first, so a model reached through another provider is a deviation recorded with its reason (`deviations`).
- **The model version is pinned.** Configuration names an exact model version, never a floating alias that the provider can move. A model change is a planned change that re-runs the evaluation set (see Testing).
- **The provider's processing terms are in place before real data is sent:** a data processing agreement when personal data is involved (`compliance`), and data use for training switched off or excluded by contract.

## The model adapter

- **One adapter per provider, behind an interface the application owns.** Use cases call the interface with domain terms; only the adapter knows the provider's SDK, endpoint and message shape.
- **The adapter owns the cross-cutting settings:** model version, timeouts, retries of transient errors with backoff (`coding-standards-all-languages`), the maximum output tokens per call, and the telemetry of every call.
- **Credentials follow `security`.** A managed identity where the provider supports it, otherwise a key in Key Vault read at runtime; never a key in configuration or code.
- **Prompts are code.** System prompts and templates live in the repository next to the use case, are reviewed in pull requests, and are versioned with the application. A prompt is never assembled from user input by string concatenation where a structured field or a separate message fits.

## What an agent may do

- **Tools are the application's use cases.** Each agent has an explicit allow-list of tools, and each tool calls an application service with its own validation and authorization. A tool never reaches the database, the file store or another system directly (`integration`: changes go through the application).
- **An agent acts for a principal.** It runs with the rights of the user it acts for, or under its own identity with a role narrowed to its tools; its entries in the audit stream (`security`) name both the agent and the person on whose behalf it acted.
- **Irreversible or outward actions need a person's confirmation.** An agent proposes, and a person confirms, before anything is sent outside the organisation, money or rights change, or data is deleted. A decision record (`project-records`) may lift this for a named action with its reason.
- **Content the agent reads cannot widen its reach.** An instruction inside a document, email, web page, upload or tool result is treated as data: it can change what the agent writes, never which tools it may call, whose data it may read or where it sends anything. An outbound URL an agent fetches is restricted as `security` requires (SSRF).
- **An agent run is a background job** when it takes more than one model call or may take longer than a request (`background-jobs`). Each step's state is saved, so a stopped run resumes at its next step and does not repeat a tool call with an outside effect.

## Trusting the output

- **Structured output is validated against a schema** before anything acts on it: types, allowed values, bounds and required fields, exactly as for any input (`security`: validate at every boundary). Output that does not conform is retried a bounded number of times, then fails visibly; it is never repaired by guessing.
- **Ids in model output are checked, not believed.** A record id the model returns is looked up and authorized for the principal like any id from a client.
- **Free text is encoded for its destination** (`security`) before it is shown in a screen, written into a document or sent in a mail.
- **Generated content is recognisable as generated** to the people who read it, where it reaches them as text, a summary or a proposal, and where the EU AI Act requires it.

## Limits and cost

- **Every model call has a maximum output size, and every agent run a maximum number of steps and a token budget.** A run that reaches a limit stops in a known state and says so; it does not continue on a smaller budget.
- **Use is rate-limited per principal** like an expensive endpoint (`api`).
- **Model cost is part of the cost estimate** (`sizing`), and usage is measured as a metric per feature, so a budget alert can fire before the month's ceiling is reached (`observability`).

## Data, logging and telemetry

- **Send the model only what the task needs.** A use case selects the fields it passes; it does not hand the model whole records "for context". Restricted data is sent only when the feature needs it and the data register says so.
- **The data register records every flow to a model provider** (`project-records`): what is sent, for which feature, the provider's retention and its location.
- **Prompts and responses are not written to application logs,** because they can hold personal data. Each call logs identifiers only: feature, model version, token counts, duration, outcome and trace id (`observability`). Where the product needs the full exchange (to show a user what the agent did, or to investigate), it is stored in an access-controlled store with a retention in the data register.
- **Every model call and tool call is a span in the trace** of the job or request it belongs to (`observability`).

## Testing

- **Unit and integration tests use a fake model adapter** that returns fixed responses, so they are deterministic, fast and free, and cover the paths for invalid output, refusals, timeouts and limits (`testing-strategy`).
- **Each AI feature has an evaluation set:** representative cases with the expected outcome or the criteria a good answer meets, including cases that must be refused and cases with planted instructions in the content read. It runs against the real model before a change of model, prompt or tools is merged, and its result is recorded on the pull request.
- **An AI feature that can call tools gets a threat model** (`project-records`), covering instructions planted in content it reads, misuse of each tool, and data leaving through its output.

## Review and ownership

- **Owners:** the vault owners maintain this document.
