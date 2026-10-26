---
id: deviations
scope: global
severity: warning
applies: all
schemaVersion: 1
---

# Architecture standards: Deviations

How a project deviates from a principle or standard in this vault. Principles guide, they are not laws: a deviation is possible, as long as it is recorded and signed off by an architect. These rules describe when the work can continue while the sign-off is pending and when it must wait.

## Deviation philosophy

- **Deviating is allowed, hiding it is not.** A choice that goes against a principle or standard is always recorded, whoever makes it and however small it seems.
- **An architect signs off.** Every deviation is validated by an architect, who approves or rejects it.
- **Only what is hard to undo waits.** Whether the work may continue before the sign-off depends on how easily the choice can be reversed if the architect rejects it.

## Recording a deviation

- **A deviation is recorded when the choice is made,** as a decision record (`project-records`), before the work that depends on it is merged.
- **The record states:** the principle or standard deviated from, the choice made, the reason, the alternatives considered, the impact, and whether it is easy or hard to reverse.
- **The record carries the outcome:** approved or rejected, by which architect, and when.

## Easy or hard to reverse

| | Easy to reverse | Hard to reverse |
|---|---|---|
| Meaning | Contained within the workload and can be changed later with limited rework | Changes the platform, a contract, a data store, a security or compliance boundary, or other teams' work |
| Example | Choosing one compute service over another within Azure | Moving a workload from Azure to another cloud |
| Before sign-off | **Not blocking.** The work continues along the chosen direction | **Blocking.** The affected work waits for the sign-off |

- **When in doubt, it is hard to reverse.** A deviation that cannot clearly be classified as easy to reverse is treated as blocking.
- **Security and compliance deviations are always blocking.** A deviation from a security, compliance or cloud governance rule waits for sign-off.
- **A security deviation has its own approvers.** Adopting any additions or deviations from the security rules needs consultation with the information security officer and the approval of the IT director, and additionally the enterprise architect if architectural change is involved. These are roles, not people: each project records who holds each role in its project definition before its first deviation. One person may hold several roles when the organisation has no separate holder, and the record says so.
- **A rejected easy deviation is reworked.** When the architect rejects a non-blocking deviation, the work is changed to the approved direction, and the rework is planned like any other work.

## Blocking only what is affected

- **A blocking deviation blocks only the work that depends on it.** Other work in the same phase that does not depend on the decision continues.
- **The blocked work is visible.** It is marked as waiting for the architect's sign-off, so the decision can be prioritised.
