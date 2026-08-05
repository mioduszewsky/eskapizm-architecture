# ADR 3 — The model calls typed tools, it does not write the document

**Status:** accepted, 29 tools, 114 unit tests
**Date:** 2026

## Context

The operator builds a travel plan by talking to an assistant. A plan is a large structured document — days, accommodations, flights, transfers, activities, restaurants, budget lines, checklists, practical information — and it gets edited dozens of times before a customer sees it.

The naive design is: give the model the current plan, ask for the updated plan, save what comes back.

That design fails in a specific and expensive way. Ask for one hotel to change and the model returns the whole document, having silently dropped three days it didn't consider important. The customer's itinerary is now missing a week and nobody notices until they're on the trip.

## Decision

The model never emits the plan. It calls typed tools, each one a narrow mutation the backend validates and applies:

`add_day` · `modify_day` · `remove_day` · `reorder_days` · `add_accommodation` · `modify_accommodation` · `remove_accommodation` · `group_accommodations` · `add_flight` · `modify_flight` · `remove_flight` · `add_transfer` · `modify_transfer` · `remove_transfer` · `add_activity` · `modify_activity` · `remove_activity` · `reorder_activity_locations` · `add_route_stop` · `modify_route_stop` · `remove_route_stop` · `add_restaurant_location` · `add_checklist_item` · `add_practical_info` · `update_budget` · `update_budget_item` · `add_budget_note` · `update_metadata` · `set_image` · `replace_text`

Twenty-nine narrow operations instead of one wide one. The blast radius of a confused model is now a single field.

## Consequences

**Good.** Data loss stopped being a class of bug. Each tool is independently testable, which is why this layer has 114 unit tests — the highest density in the codebase, deliberately, because failures here are invisible until they reach a customer. Tool definitions also act as documentation of what the domain actually permits.

**Bad.** Twenty-nine tools is a lot of surface to maintain, and every new plan capability means a new tool, a renderer, a test, and a prompt update rather than a prompt update alone. There's real tension between giving the model enough tools to be useful and few enough to choose correctly.

**What kept biting.** Most production bugs in this layer were not the model being stupid — they were the tool contract being narrower than the intent. An operator asked to change an attraction and the model called `modify_activity` correctly, but the tool only handled flat fields, so a request to add a details panel did nothing and the model reported success. Same shape of bug when accommodations needed grouping headers: the model tried to express it through single-item edits because no grouping tool existed.

The pattern is consistent enough to state as a rule: when an agent confidently claims it did something and nothing changed, suspect the tool schema before the prompt.
