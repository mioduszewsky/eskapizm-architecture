# ADR 2. Edge Functions instead of a deployed backend

**Status:** accepted, 38 functions in production
**Date:** 2025 to 2026

## Context

The platform needs server-side work: taking payments, issuing invoices, submitting e-invoices to the Polish KSeF system, sending transactional email, syncing tasks both ways with ClickUp, calling the model API, validating discount and affiliate codes, approving commissions, generating sitemaps.

All of it needs secrets. None of it can run in the browser.

The conventional answer is a Node or Python service on a container platform. One developer, no budget, and a business that might not exist in six months made that a bad trade.

## Decision

Every server-side concern is a Supabase Edge Function. Deno, deployed per function, invoked over HTTPS with the caller's JWT. Functions hold the service role key and the third-party secrets; the browser holds an anon key that can barely do anything.

Functions are grouped by responsibility rather than by entity:

- **money.** Payment creation, payment webhooks, invoice creation, invoice resend, PDF export, KSeF status sync
- **AI.** The plan builder chat agent, plan generation, document parsing, the blog assistant
- **lifecycle.** User creation, password reset, profile updates, welcome and confirmation email
- **growth.** Newsletter subscription, discount validation, affiliate visit tracking, code validation, commission approval
- **operations.** ClickUp sync and webhook, sitemap generation, shared plan publishing, feedback

## Why not a real backend

A container service would have added a deployment target, a scaling story, a patching obligation, and a bill, to run code that is mostly short, event-shaped, and idle. The functions model matched the actual workload: bursty, independent, secret-holding.

It also kept authorization in one place. Because RLS in Postgres is the enforcement layer, a function is not the thing deciding who may read what; it's the thing holding a credential. That is a much smaller surface to get right.

## Consequences

**Good.** No servers. Deploys are per function, so a change to invoicing cannot break the AI agent. Cost tracks usage, which for a pre-revenue business meant approximately nothing. Deno's TypeScript-first runtime meant one language across the stack.

**Bad.** 38 functions is past the point where a flat list is comfortable, and shared logic between them is genuinely awkward. The boundaries are deployment boundaries, not module boundaries. Cold starts are real on the rarely-called ones. Local reproduction of the full system is harder than `docker compose up`.

**The mistake I made.** For a while some functions existed only in the cloud, deployed by hand and never committed. I discovered three live functions with no source in the repository, one of which held an invoicing API token. Drift goes both directions: the repo can be ahead of production, and production can be ahead of the repo. I now diff both lists rather than trusting `git status`, and everything deploys through CI.
