# Pareto Vault

Living patterns and how-to guidance for building on the Pareto platform. The source of truth for conventions that evolve as real code teaches us better shapes. This vault is Pareto-specific.

### What pp-core does with these documents

pp-core reads this folder at various moments during discovery, and planning of a phase. It is then used to find the relevant content to reference for planning, finding standards, and passing it to the implementation phases.

## Ownership and cadence

- **Owners:** **TODO:** name the vault owners.
- **Reviewer:** the vault owners (enforced through ADO branch policies).
- **Cadence:** as-needed. A vault PR is not gated by platform releases.

## How changes propagate to consumers

**Pushing to this vault does NOT auto-flow to consumers.** Versions are annotated tags on `main` using SemVer 2.0.0 versioning (`v1.0.0` … `v1.2.1`); each consumer pins a tag in `client.vaultRef` and keeps fetching that tag until someone bumps it. The pin is the governance feature: a consumer moves deliberately, and a phase built against `v1.2.1` can be audited against `v1.2.1` forever.

**Projects read the public copy.** The vault is written and reviewed in the private repo `pareto-ooo/Pareto-Vault`. Every `vX.Y.Z` tag pushed there is published by the `Publish release` workflow to the public, read-only repo [`pareto-ooo/pareto-vault-releases`](https://github.com/pareto-ooo/pareto-vault-releases), under the same tag and without history, `.planning/` or `.github/`. Projects point `client.vaultRepo` at that public repo, so using the vault needs no access to this one. An older tag that a project still pins is published by running the workflow by hand with that tag.

**Everything in a release becomes public.** A publish is blocked when a file matches `.github/scripts/publish-denylist.txt` (client names, e-mail addresses, Azure ids, credentials). That list is a safety net, not a review: never write client facts, internal names or real identifiers into the vault.

## Adding content

Add a new .MD file following roughly the formatting of the existing documents:
 1. A small header block with static information
 2. A short description of what it covers and why
 3. The actual content of the document regarding the topic
 4. Update `INDEX.md` to be current

### Philosophy

The goal is an easy browsable data set of all knowledge required to judge, plan, and develop a software project within the company.
Due to a proper index, the plan is: Relevant files get collected by the planning step and listed as mandatory reading. Implementation agents get this list, and read it.
As a result, we want to design this document with the following in mind:
- Do not repeat yourself 
    - prevents future rot or contradictions
- Do not refer to other documents
    - creates hard references that are hard to keep track of, prevent future rot
- Do not contradict other items in the vault, be additive
    - Additive rule build constructively to make a complete ruleset
    - Contradictions cause confusion and problems upon agent interpretation

### The `header` block

The header block in each file contains some basic information like a version and how important it is. The documentation for what is expected lives in pp-core.

### Keeping `INDEX.md` current

`INDEX.md` at the vault root is what planning sessions read to decide which documents to open. Every session reads the whole file, so it has to stay short and precise. Add, rename or remove its line whenever a pattern document is added, renamed or removed. To rewrite it from scratch, follow these rules:

1. One `## <folder>` heading per top-level folder, in any order.
2. One line per document: `` - `<path from vault root>` — <what it covers>; <helps decide which choice> ``. The separator is an em dash with a space on each side.
3. Describe the decision the document informs, not just its subject. Sessions match their topics against these words.
4. Keep each line under 250 characters.
5. List every pattern document. Leave out `README.md`, `BACKLOG.md` and `INDEX.md` itself.
6. Legacy folder gets excluded from the `INDEX.md` as well.
7. Paths are relative to the vault root: no leading `/`, no `..`.
8. Start the description with `[cross-cutting]` when the document holds rules every decision answers to, whatever its topic: who approves what, naming, where files go. pp-core's Discovery Sprint reads every cross-cutting document in every session, on top of the documents it picks by topic, so keep the mark to short documents whose rules a session could break without any of its topics naming them.

The closing clause (`used when planning …`) is written from the planner's point of view: it says in which kind of work the document should be read, judged from the document's content. A coding-standards document says it applies when planning or writing code; a security document names the security-sensitive work it covers. Do not give a document a clause its content does not support.

Example from the current index:

```markdown
## db
- `db/soft-delete.md` — hard delete by default and when soft delete is justified; used when planning delete, archive or retention behavior to decide whether rows vanish or are flagged

## security
- `security/secrets.md` — Key Vault per app per env, narrow RBAC, no rotation code; used when planning anything needing credentials, connection strings or API keys

## practices
- `practices/` — machine-readable one-rule-per-file practices pp-core matches against planned files and injects into PLAN.md; used automatically at plan time and never chosen manually
```