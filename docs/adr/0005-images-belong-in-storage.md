# ADR 5 — Images belong in object storage, not in the database

**Status:** accepted, migration completed
**Date:** 2026

## This one is a correction, not a design

Early on, images that operators pasted into plans were stored as base64 data URLs inside the plan document. It was the shortest path: no upload flow, no storage bucket, no separate lifecycle, no broken-link states. The image was simply part of the record.

It worked until the bill showed up as bandwidth.

## What went wrong

A plan document is read constantly — by the operator editing it, by the customer viewing it, by every render of the AI workspace. With images inlined, each of those reads pulled the full binary payload out of Postgres and across the network, base64-encoded, which is about a third larger than the original bytes.

Egress climbed toward the limits of the free tier. The direct cause wasn't traffic growth; it was that every read of a text document was secretly a read of several megabytes of images.

## Decision

Images move to Supabase Storage. The plan holds a reference. A migration function converted existing inlined images to stored objects and rewrote the documents to point at them.

## Consequences

**Good.** Plan documents went back to being small, so reads are cheap and fast. Egress dropped back under the free tier, which was the actual business requirement. Images now have their own lifecycle — they can be cached by the CDN, replaced, and served at different sizes.

**Bad.** There's now an upload path, a bucket policy, and the possibility of a reference pointing at an object that no longer exists. Storage listings had to be locked down to admin-only during a later hardening pass, because a public listing of a bucket is its own small disclosure problem.

**Still open.** The plan builder can still produce base64 in some paths. The migration fixed the accumulated data; it did not fully remove the ability to create the problem again. That is a real, known, unfinished piece of work rather than something I'm claiming is done.

## Why it's in this write-up

Two reasons. First, it's the clearest example here of a decision that was correct for the first month and wrong by the sixth — the kind that only surfaces when something is actually being used.

Second, the diagnosis was the interesting part, not the fix. The symptom was a bandwidth number, several layers away from the cause. Moving files to storage is not an insight; noticing that a text read was really an image read is.
