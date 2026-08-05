# ADR 4 — Every mutation reports what actually happened

**Status:** accepted
**Date:** 2026

## Context

ADR 3 stopped the model from destroying documents. It did not stop a subtler problem: the UI and the database disagreeing about whether an edit happened.

The failure looked like this. The model calls a tool. The frontend optimistically applies the change and tells the user it's done. The backend write fails, or is a no-op because the requested change was already present, or is rejected by a guard. Now the operator sees an updated plan that does not exist on the server. They keep editing on top of a lie.

The reverse also happened: a tool succeeded, the frontend also applied it locally, and the same edit got written twice.

## Decision

Tool calls return an explicit outcome, not a boolean and not a message:

- `applied` — the mutation changed persisted state
- `no_change` — the request was valid but the state already matched, so nothing was written
- `failed` — the mutation was attempted and did not succeed
- `blocked` — a guard refused it, for example a destructive edit that wasn't clearly intended

Two rules follow from it:

1. **The backend commits first.** The instruction to update the UI is only emitted after the write to the plan and the session record has landed. A dropped connection can lose the notification; it cannot leave the client ahead of the server.
2. **The frontend does not double-apply.** When the response carries an action identifier, the client treats the server as the source of truth and does not write its own copy.

## Consequences

**Good.** "The AI said it changed something but nothing changed" became a diagnosable event with a status attached rather than a user complaint. Repeating the same instruction twice now produces `no_change` and creates no second action record — verified against a replayed session.

**Bad.** Every tool has to be honest about which of the four it returns, and `no_change` in particular is easy to get wrong: a tool that always reports `applied` looks fine in tests and lies in production. This is enforced by convention and tests rather than by the type system, which I'm not fully happy with.

**Why it's worth the ceremony.** The pattern generalises past AI. Any system where a client optimistically renders the result of a remote mutation has this bug available to it. Making the outcome an explicit part of the contract, rather than inferring it from the absence of an error, is cheap and removes an entire category of "it says it worked".
