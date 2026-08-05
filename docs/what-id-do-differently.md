# What I'd do differently

Written for the next platform, not as an apology for this one. Everything here is still true of the codebase as it stands.

## Deployment discipline from day one

The worst thing I found in this project was three Edge Functions running in production with no source in the repository — one of them holding an invoicing API token. They'd been deployed by hand during a debugging session and never committed.

I had been checking `git status` and treating a clean tree as "everything is shipped". That only catches drift in one direction. Production can be ahead of the repository just as easily as behind it, and nothing warns you.

Now: everything deploys through CI, and the check is a diff between the deployed function list and the repository list, not a look at local changes.

## The admin panel deserved to be a product

It grew feature by feature as operations needed things — orders, then plans, then the CMS, then affiliates, then tasks, then the issue queue. Each addition was reasonable. The sum is a panel with no coherent information architecture, where the useful thing is often three clicks deep behind filters that were added for one specific afternoon.

The public product got design attention because customers see it. The internal product got none, and it's the surface my co-founder spends the most hours in. That was the wrong allocation.

## Test the AI layer's contracts, not just its units

114 unit tests on the planning layer, and the bugs that actually reached production were mostly not unit-level. They were mismatches between what an operator asked for, what the tool schema could express, and what the renderer could display — three things that each worked correctly in isolation.

A tool can pass every test and still be unable to represent the thing a user just asked for. What I was missing was tests at the level of "operator says X, does the observable state change accordingly", exercised against realistic sessions rather than synthetic fixtures.

## Stop writing base64 images at the source

ADR 5 migrated the accumulated data out of the database. It did not fully remove the ability to create the problem again — some builder paths can still produce inlined images. Fixing the data without closing the door is half a fix, and I'm listing it here rather than pretending otherwise.

## Reproduce the user's actual sequence before claiming a fix

Several bugs came back after I'd marked them fixed, because I'd verified against a synthetic case rather than the sequence the operator actually performed. A plan built by a script is not a plan built by a person pasting a markdown document, asking for three changes, and then referring back to "the file I sent earlier".

The recurring lesson is that in an interactive system, the state you reach matters as much as the state you're in. Verification has to walk the same path.

## Pick the boring integration first

The pieces that consumed the most time were rarely the interesting ones. Payments, invoicing, and the Polish e-invoicing regime took disproportionate effort against zero user-visible payoff — but they're also the pieces where being wrong is a legal problem rather than a bad review.

If I were sequencing again I'd build the money path end to end, with reconciliation, before building anything that looks impressive in a screenshot.
