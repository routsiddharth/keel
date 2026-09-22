# Keel

**Status: nothing is built yet. Two of us. This document exists so we stop re-deciding things we've already decided.**

## The gap

The AI labs sell subscriptions to consumers and forward-deployed engineers to the Fortune 500. Everyone in between gets neither.

A 200-person company can't clear a $400K engagement minimum, so it builds agents in-house. Those agents come out over-permissioned — one API key with access to everything, because scoping it properly was a week nobody had. Unmonitored, because logging was a later ticket. Hard to audit, because nobody designed for the question.

It ends one of two ways. Security finds out and freezes deployments. Or a customer finds a silent failure before anyone on the team does.

## What Keel is

The layer underneath: connectors into a company's systems, a permission boundary around them, and a record of everything an agent did — plus a visual builder on top, so the people who own the work can create agents without going through engineering.

We install it and configure it in person. Then we hand over the builder.

The bet: this is the same work a forward-deployed engineer does in their first three months, it is substantially the same work at every company, and it can be done once in a form that gets reused. Productize the FDE.

## Blocks are the permissions

An agent is assembled out of blocks, the way a Shortcut or a Scratch program is. Each block is one concrete action:

- Read new tickets in Zendesk
- Look up an order in our database
- Post a message to #support
- Issue a refund under $50

There is no freeform block. There is no "run this code." An agent cannot do something there is no block for — not *shouldn't*, **can't**.

Which means "what can this agent do?" is answered by reading it. No scope audit, no trusting that whoever built it was careful. A support lead and a security reviewer look at the same screen and both understand it.

This is the entire safety model. Everything else follows from it.

## Who does what

**Our engineers** configure inputs and outputs, define what's allowed, build the blocks specific to that company's systems, and keep the palette over time — adding blocks, retiring them, changing what a block is permitted to touch.

**Their people** build agents.

Neither can do the other's job, on purpose. A customer's boundary can tighten the day their security posture changes without touching a single agent anyone built.

## Who builds agents

The person whose work it is. The support lead, the ops manager, the recruiter. Not an engineer. Not the analyst who knows SQL. Someone who has never automated anything in their life and knows their process cold because they do it every day.

Reference points: Scratch, Apple Shortcuts.

The design principle is iOS — abstract away everything the user doesn't need to see. Not fewer capabilities, less surface. A beginner and a power user open the same Settings app.

## Where it runs

Inside the customer's walls. Their data doesn't leave.

We're already sending engineers on site to configure it, so self-hosting is marginal extra work, and together the two facts make one story: our engineers came to you, and nothing you own left the building. Held against that, the reckless thing in the room is the homegrown agent they already have running on a god-mode key.

## What this rules out

Writing these down so we don't rediscover them as surprises.

- **No self-serve signup.** Every customer starts with us in the room. Growth is capped by our engineering time until block reuse actually compounds.
- **The palette is the ceiling.** If a block doesn't exist, the customer waits for us. For that window we are the bottleneck we're selling against.
- **Self-hosted means shipping is hard.** New blocks and fixes have to reach installs we don't control.
- **The whole bet is customer #2.** If the second install isn't meaningfully cheaper than the first, we're a consulting firm with extra steps.

## Decided

- Agents are built and run in Keel. We are not a safety layer underneath agents someone else wrote — the builder is the product.
- The builder is visual blocks, Scratch / Shortcuts style.
- Built for non-engineers, specifically the person who owns the work.
- Our engineers own the palette and the permission boundary. Customers own their agents.
- Self-hosted, inside the customer's walls.

## Open

- **Which systems first?** Slack, Gmail, Zendesk, Salesforce, Postgres, a generic internal HTTP API is the obvious opening guess. Not picked.
- **Where does the model show up in the block model?** Some blocks are deterministic (fetch a record, post a message). Some are judgment (decide the category, draft the reply). Does the person building see a difference, and should they?
- **How do agents start?** Schedule, event trigger, someone presses a button?
- **What happens when an agent fails halfway?** Stop, retry, hand to a human?
- **How does someone try an agent before it's live?** There has to be an answer and it's probably load-bearing.
- **What is the audit trail for** — compliance, debugging, or both? Those are different products.
- **How do updates reach installs we don't control?**
- **Pricing.**
