# ADR 1. Serve a SPA, prerender it at the edge for crawlers

**Status:** accepted, running in production
**Date:** 2026

## Context

Organic search is the top of the funnel. The blog and destination pages are what bring travellers in months before they buy, so being invisible to crawlers would have removed the cheapest acquisition channel the business had.

The app itself is a React SPA. A lot of it is authenticated, stateful, and interactive (the trip app, the admin panel, the AI planning workspace). Those parts gain nothing from server rendering.

The obvious answer was to migrate to Next.js and get SSR. I didn't.

## Decision

Keep the Vite SPA. Put a Cloudflare Worker in front of it that:

- passes static assets straight through to origin,
- detects crawlers by user agent and serves them a prerendered HTML snapshot from KV,
- renders that snapshot on demand through Cloudflare's Browser Rendering API, waiting for network idle so React and the data calls have finished,
- caches bot responses for 24 hours,
- additionally serves the cached HTML to *human* visitors on a small set of priority routes, where it's simply the fastest way to a good LCP.

The crawler list includes AI agents alongside search engines: `gptbot`, `chatgpt-user`, `perplexitybot`, `claudebot`, `claude-web`, `google-extended`, `anthropic-ai`, `cohere-ai`, `meta-externalagent`, `bytespider`.

## Why not SSR

Migrating to Next.js would have meant rewriting routing, data fetching, and auth across a codebase that was already large, to solve a problem that only affects the public marketing surface, maybe fifteen percent of the routes. The authenticated product would have carried the complexity of a framework it didn't need.

It also would have introduced a server that has to stay up, be patched, and be paid for. With one developer and no infrastructure budget, "there is no server" was worth a lot.

## Consequences

**Good.** Public pages are indexable and answerable by AI crawlers. The interactive product stayed simple. Cost stayed near zero. The whole thing is one file of Worker code that a person can read in ten minutes.

**Bad.** The prerender path is a second rendering mode, which means a class of bug where the snapshot and the live app disagree. Cache invalidation after a content change is a manual concern. And it depends on user-agent detection, which is a heuristic, not a guarantee. An unlisted crawler gets the empty shell.

**Watch out for.** Asset caching bit me hard once. After a deploy, Cloudflare on the custom domain kept serving a stale HTML fallback with a 200 for a newly hashed JS bundle, so users got a blank root and Vite reported a failed dynamic import. Fixed by isolating asset names per commit and setting `Cache-Control: no-store` on the JS and CSS paths, then verifying in a clean browser profile that the bundle came back as `application/javascript` with a cache bypass.

That incident is the reason I now treat "it deployed" and "it is actually being served" as two separate claims.
