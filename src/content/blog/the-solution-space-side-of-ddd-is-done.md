---
title: The Solution-Space Side of DDD Is Done
description: "What's left of internal bounded contexts when you build with event sourcing, vertical slices, and a model-first practice — and what agentic AI changes about that."
pubDate: 2026-06-28
updatedDate: 2026-06-28
series: NoBoundedContexts
seriesPart: 1
seriesTotal: 1
lastVerified: 2026-06-28
---

> **Takeaway.** Inside a monolith you own end-to-end — event-sourced, vertical-sliced, model-first — strategic DDD survives on the problem-space side, and its solution-space side (bounded contexts as drawn artifacts, tactical patterns as code shapes) has nothing structural left to do. The only wall worth drawing is the one around the outside.

**TL;DR**

- **Strategic DDD is problem-space work** — naming subdomains, building ubiquitous language, modeling the business — and survives in full. What collapses is the **solution-space side**: bounded contexts as drawn artifacts, tactical patterns as code shapes.
- Bounded contexts were invented to defend against leak paths — shared tables, shared in-memory state, shared services, multi-team language drift — that **event sourcing with Dynamic Consistency Boundaries**, **vertical slice architecture**, and **a model-first practice** have already closed.
- Inside such a monolith there is **one swimlane**, not many. Subdomains stay in the business as a way of talking about the work; they have no representation in the solution space — no module, no namespace, no label.
- A single parse-time rule (every fully-qualified event name unique) plus the model-first practice does the linguistic work strategic DDD used to do — singularly, not multiplied across internal subdomains.
- The **translation slice** still earns its keep — at the *periphery*, between the owned monolith and legacy or external systems. Inside the monolith, with one context, there is nothing to translate.
- Drawing internal BCs anyway is **not free**: it costs design hours, language-drift maintenance, institutional ownership disputes, and migration pain when the business shape changes under you.
- **Agentic AI** is the force multiplier, not the prerequisite. It takes the residual cost of being wrong about a boundary from "manageable" to "trivial."

---

You're building event-sourced software with vertical slices and event modeling. You've drawn bounded contexts, because the literature tells you to. And to bridge them, you've built a structural pattern that folds a foreign event into a local read-model, runs a processor against that model, and emits a local event in the consuming context when the processor decides to act. Call it a **translation slice**, a process manager, a cross-context bridge, an internal anti-corruption layer — the name varies, the shape doesn't.

The contexts cost real design. You drew lines, named modules, chose integration patterns, maintain a context map alongside the code. The bridge costs real runtime — a projection that has to stay caught up, a processor that has to drain it, a command path that has to fire. Are the contexts earning that cost? Is the bridge?

This post argues that for a specific class of system, internal bounded contexts have nothing structural left to do — and the bridges between them are not narrowly useful, they don't exist by design. The class of system is:

- **Event sourcing**, specifically with **dynamic consistency boundaries** (DCB) rather than aggregates per consistency requirement.
- **Vertical slice architecture**, where each slice owns its command, its decision state, its projection, its tests, with no in-memory state shared across slices.
- **A model-first practice** — a single design artifact (a board, a DSL, a YAML spec, schemas-as-code) is the source of truth, and code shapes are templated from it. In the codebase this post draws on, that practice is **event modeling**; the argument runs on the category, not the specific practice.
- **Optionally: agentic AI** for cross-cutting code edits. Force multiplier, not prerequisite.

One topology note before the argument begins, because the rest of the post depends on it: **inside the owned monolith, there is one swimlane.** Not many, not one per subdomain — one. Subdomains exist in the *business* — in conversations, in org charts, in how people describe the work — but they have no representation in the solution space. No module, no namespace, no swimlane, no label. Subdomains are a problem-space concept and stay there; the solution space has events, commands, queries, slices, and one lane. Separate swimlanes are reserved for **legacy systems** (parts of the broader business owned by other teams, on other cadences, with their own languages) and **external systems** (third parties we don't own at all). One inside. N legacy + M external outside.

Think of it as a walled city. Inside the walls, everyone speaks one language; the neighborhoods (Manuscripts, Reviews, Editorial) are how people talk about where they live and what they do, but they aren't separately governed districts and they don't have walls between them. Outside the walls are other cities — with their own languages, their own customs, their own pace. Translation booths sit at the gates, not in the streets.

![The walled city — one outer wall, no internal walls, translation booths at the gates](/images/fig-01-walled-city.svg)

*Figure 1 — The walled city: one owned monolith with neighborhoods as ghost outlines, external and legacy systems outside, translation slices on the wall at the gates.*

Either way the conclusion is the same: internal bounded contexts as solution-space architecture are abolished by these conditions. What survives is the one outer wall.

## What bounded contexts were originally for

Start with **Parnas, 1972**[^1]: modules exist to hide design decisions likely to change. A boundary is justified by what it conceals; coupling is bad because shared knowledge forces shared change. If a boundary doesn't hide a decision that's actually likely to change, it isn't a boundary — it's furniture.

**Evans, 2003**[^2], added the social and linguistic layer. A bounded context is the scope within which a ubiquitous language stays unambiguous and a domain model can be internally consistent without negotiating with the rest of the world. The motivation was as much political as technical: large systems have multiple teams, multiple stakeholders, multiple legitimate ways to talk about the same nominal entity. A BC says *inside here, "Order" means this one thing.*

**Strategic DDD lives on two planes** — though canonical DDD literature insists they are one. Evans's *Model-Driven Design* chapter[^2] rejects splitting analysis-model from code-model: the model lives in the code, in the language of the business, or there is no model. This post argues the two planes come apart cleanly under the conditions below, and the divergence from Evans is deliberate. The **problem-space** plane: subdomains, business modeling, ubiquitous language — understanding what the business does and the vocabulary that work demands. The **solution-space** plane: bounded contexts as drawn artifacts, context maps, tactical patterns (aggregates, repositories, value objects) as code shapes. Conventional DDD treats the two as inseparable; this post argues they come apart cleanly. The problem-space plane stays — subdomains live in the business, ubiquitous language gets done in modeling conversations, the work of understanding the domain doesn't go anywhere. The solution-space plane has nothing structural left to do inside the code. The artifacts disappear; the work the artifacts were trying to discipline survives, on the other plane.

**Vernon, 2013**[^3], made BCs operational with strategic patterns — anti-corruption layer, customer-supplier, published language — each naming a way two contexts can integrate; each assuming contexts cost something to bridge because they're owned by different teams on different cadences. **Khononov, 2021**, and his subsequent *Balanced Coupling* work[^4] sharpened this into a formula. Coupling has three dimensions — integration strength (shared knowledge), distance (architectural separation), and volatility (rate of change) — and adding distance is only justified when strength × volatility warrants the cost. The formula cuts both ways for this post. When strength × volatility is high — genuinely-different-cadence subdomains owned by different teams — Khononov's framework prescribes distance, and the post scopes those cases out (§9). When the conditions below collapse the strength term — single team, single deployment, one model — the framework prescribes *against* distance internally, which is exactly the inside of the monolith this post argues for.

Every one of those motivations assumes a coupling surface *other than events*. Shared models, shared language, shared tables, shared teams. The BC was always a wall against a specific kind of leak.

![The bounded context, shrinking — from Evans's many fortified BCs to one outer wall](/images/fig-02-many-walls-to-one.svg)

*Figure 2 — The bounded context, shrinking. Evans 2003: many fortified BCs with ACLs between them. Modular monolith: BCs as soft-walled modules sharing deployment and vocabulary. This post: one outer wall, no internal labels, events scattered inside.*

## For DDD-aligned readers

A DDD-aligned reader has three legitimate fears about the argument so far. Each is worth naming before the post goes further.

**"You're just renaming bounded contexts as swimlanes."** Sharper than that. Inside the monolith there is *one* swimlane, for the whole business — not many. Subdomains live in the business (in conversations, in org charts, in how people describe what they do); they have no representation in the solution space at all. No module per subdomain, no namespace per subdomain, no swimlane per subdomain. You don't draw `Manuscripts/X` versus `Reviews/Y` because the code never knows about Manuscripts or Reviews as structural concepts. Multiple swimlanes are reserved for parts of the world you don't own. The "swimlane label" that survives is the one outer boundary between the owned monolith and everything else; it's not a renamed sub-BC, because there are no internal sub-BCs to rename. What changed conditions made redundant is the structural artifact, the multiplicity, *and* the very idea that subdomains need a solution-space form.

**"This kills strategic DDD."** It doesn't, not on the problem-space plane where most of strategic DDD's work lives, and not at the periphery where the rest of it still applies. Subdomain analysis, business modeling, ubiquitous language work — none of that collapses. That's the problem-space plane, untouched. Genuinely-external integration — third-party APIs, ERPs, legacy systems, anything you don't own — still needs ACLs, published languages, customer-supplier negotiations: that's strategic DDD's solution-space application at the periphery, where its conditions still hold. What collapses is strategic DDD's solution-space application *internally*: the practice of carving the inside of a fully-owned monolith into bounded contexts as a separate strategic step. When one team owns the whole thing, on one codebase, in one process, behind one CI build, with a model-first practice doing the linguistic work, strategic design as a separate step has nothing structural to enforce that the model doesn't already enforce. Strategic DDD survives — on the problem-space plane everywhere, and on the solution-space plane at the periphery.

**"Ubiquitous language requires explicit boundaries."** The model-first practice *is* the language work. Events are named in one shared model; the parse-time assertion refuses duplicates; the visual board catches collisions the moment they're minted. You're doing the linguistic work — you're just not calling it strategic design, and you're not multiplying it across internal subdomains that share a team and a deployment.

**"Evans rejected this split. Why think it works?"** Evans's anti-dualism in *Model-Driven Design* is a response to a specific dysfunction: analysis architects throwing UML diagrams over a wall to developers who built something only loosely related, with language drifting between the two artifacts. Evans's solution was to fuse them — one model, in code, in the language of the business. That fusion still happens in this architecture; what changes is the artifact doing the fusing. Where Evans assumed code-as-model with strategic-design overlays on top, this post argues the model is the model-first spec, the code is templated from it, and there are no parallel strategic-design artifacts to drift away from. Evans's *problem* — drift between two artifacts — is solved by collapsing to one artifact, not by carving the inside of the code into bounded contexts. The anti-dualism stands; the dualism this post draws is not the dualism Evans rejected.

**"Homonyms force internal boundaries — Customer-as-shopper vs. Customer-as-account."** Evans's defining example looks like proof internal subdomains need enforced vocabulary. In this architecture, that's not how homonyms are handled. When two parts of the business genuinely mean different things, the modeling work surfaces it at design time and produces two distinct event names — `OrderPlaced` and `AccountOpened`, not two `CustomerTouched` events. The homonym dissolves at the modeling table, before the code is written. The cost is exactly the work strategic DDD allocates to discovering the boundary — except the output is two events in one namespace, not two contexts with bridges between them. Where the homonyms persist into code despite the modeling work, the team isn't actually unified; the conditions for the argument don't hold, and §9's scoping note applies. Inside a deliberately-unified single-team monolith, homonyms are a modeling failure, not an architectural feature.

**"Without internal boundaries, you get a Big Ball of Mud at the event level — 200 events flat, no grouping, no way to hold the system in your head."** Foote and Yoder's term[^9] is defined by coupling surfaces, not directory layout: shared mutable state, hidden dependencies between modules, no enforceable boundaries between components. None of that exists here. Events on the log are immutable facts; the only coupling surface between slices is event name plus payload schema, declared in the model, enumerable by construction. A flat namespace with every connection published is structurally the opposite of BBoM, however it pattern-matches visually. The cognitive-load half of the worry is the subject of §5: to work on a slice you need its command, its folded events, its emitted events, and nothing else — the other 199 aren't in your head. Change locality, navigability, and linguistic discipline — the rest of what BBoM lacks — are carried by the model-first practice and the name-uniqueness rule, explicitly, before any code is written. The only mud that *could* form is two events colliding on a name, and §6 catches that at the moment of naming.

![Two planes — subdomains in problem-space, events in solution-space, no clustering](/images/fig-03-two-planes.svg)

*Figure 3 — Two planes. Subdomains live in problem-space (top) as cloud-bubbles. The code lives in solution-space (bottom) as a flat row of business events with no clustering. Arrows cross each other to show that events carry subdomain meaning but the code is not organized by subdomain.*

Evans wasn't wrong. The conditions he wrote into in 2003 — multi-team systems, separate deployments, schemas leaking through ORMs, no event store as the system of record, no model-as-source-of-truth, no agents to keep code in lockstep with the model — don't all hold here. The patterns that protected against those conditions are responses to leak paths this architecture has closed. What survives — semantic clarity, language consistency, the right to refuse to merge two concepts — survives. It just lives in one model, not in a separately-maintained design artifact.

## The CRUD-era arguments don't apply

Most "we abandoned bounded contexts and regretted it" stories are really *"we abandoned bounded contexts while still sharing tables and helpers and god-services."* That's not an argument for BCs — it's an argument against shared mutable state masquerading as one.

In a DCB + vertical-slice monolith, the historical leak paths are gone:

- **Shared mutable state across modules** — gone. Each slice owns its decision state.
- **Shared database tables** — gone. The event store keeps tagged events in one log; consistency is a query-time, tag-filtered, optimistic-locking concern over that log[^5], not a schema.
- **Shared in-memory objects** — gone. Slices share no in-memory state. The only coupling surface is the events on the log; cross-context dispatch (to legacy or external) happens by message.
- **Shared services with leaky abstractions** — gone. No services. Just events.

That whole historical pain bucket is empty. And **Dynamic Consistency Boundaries** — Sara Pellegrini's reformulation, now codified at [dcb.events](https://dcb.events/) — explicitly kill the aggregate as a structural unit. A 2025 DCB-practitioner interview frames the aggregate's structural role bluntly: *"I will never again build an event-sourced aggregate class for command handling."*[^6] The quote is about aggregates — a distinct DDD concept from bounded contexts. What dies in DCB is the aggregate-as-structural-unit, not the BC; DCB itself stays silent on whether you draw bounded contexts at all. It just removes the BC's historical job as consistency boundary, leaving the BC unanchored in its other roles. What BCs *do* after their consistency role is gone is an open architectural choice — the modular-monolith literature retains them as softer-walled modules, others drop them entirely. This post's stance is that once VSA and a model-first practice are also in place, the choice collapses to: nothing internal.

In canonical DDD, aggregates are a tactical pattern *inside* a bounded context — the BC is what gives the aggregate its identity, invariants, and language. Drop internal BCs and aggregates have nowhere to live; reach for aggregates and BCs come back with them. DCB is what makes the no-internal-BC stance coherent at the tactical layer, by replacing the aggregate's structural job with a tag-filtered query over the log. The three conditions don't just stack — they hold each other up.

What about the *refactor-insurance* case for BCs — *"draw boundaries upfront because moving things across them later is expensive"*? DCB already softens it: re-drawing a boundary by adding or removing consumers is a re-projection over the existing log[^7], not a data migration. Renaming or splitting events is still a stored-history migration — agents or no agents — but the rest is cheap by construction. **Layer agentic AI on top** and the code side of that refactor (re-tagging projections, rewriting fixtures, patching references) goes from expensive-but-possible to mechanical. With or without agents, the refactor-insurance argument for upfront BCs is much weaker than the literature assumes; with agents, it's nearly gone.

Which leaves the question most often pressed against this argument: **if events are the only coupling surface across slices, what actually breaks differently in a 200-slice no-BC monolith versus a 20-slice one?**

## Most of what "feels" like a scale-cost isn't

Walk through the usual list and most of it dissolves on contact with vertical-slice + DCB + events-only coupling.

**Spooky reaction at a distance — "if I publish event X, who reacts?"** Not a BC problem. In any event-driven system, "anyone could subscribe" is the floor. The BC's claimed trick is to keep internal domain events internal and let only translated integration events escape — but that's a public-surface annotation, not a structural property. You can have it in a no-BC monolith with a flag on the event and lose nothing. BCs aren't the mechanism here; explicit public-surface declaration is.

**Cognitive load from "too many slices."** Also not real, if VSA is doing its job. To work on slice S you need its command, its emitted events, its folded events, and nothing else. The other 199 slices aren't in your head. The 200-count only matters at navigation time ("which slice handles this?"), and that's a catalog problem orthogonal to BCs.

![The slice is the window — only command, folded events, emitted events in scope](/images/fig-04-slice-as-window.svg)

*Figure 4 — The slice is the window. 200 slice tiles, one lit; alongside, a zoomed view of the lit slice showing the only three things in scope when you work on it: command, folded events, emitted events.*

(One caveat that lands only if you're using agents: a 2025 LMPL paper on LLM modularity warns that agent-generated code routinely presents surface-level modular structure while violating deeper modular principles[^8]. That's a *within-a-slice* coherence failure, not a *too-many-slices* one — but it's the tax on "agents make refactors cheap": cheap mechanical code with semantic-coupling risk that mechanical structure used to catch. The assertion-layer mitigation below is partly aimed at that tax.)

**Event fan-in — the cost the previous draft of this argument couldn't quite dismiss.** It also dissolves in a model-first workflow. When the model-first spec is the single source of truth, every slice declares which events it folds, and code shapes are templated from it. The spec *contains* the fan-in for every event by construction; emitting a fan-in report from it is a one-line iteration. When an event's shape changes, the validator already knows every slice that depends on it. You never *discover* fan-in; the model published it.

**That leaves exactly one cost.** Two unrelated parts of the system both want an event called `Approved`. In a 20-slice model you notice; at 200, two people (or two agent sessions) add the name on different days and a downstream projection happily folds both into something that means neither. Pure naming hygiene — but one that scales worse with slice count than the others.

That's the whole list. One cost. Tiny.

## The mitigation: name uniqueness is a side effect of modeling

Inside the monolith, every business event lives in one namespace — `OrderPlaced`, `ReviewSubmitted`, `ManuscriptApproved` — and the modeling practice catches collisions at design time. When the practice has a visual surface (a board, a diagram, the kind of canvas event modeling renders the model on), redundancy is unmissable; when it's text-only (a YAML catalog, a schema directory, an ADR set), the parse-time uniqueness rule below catches it mechanically. Either way, one namespace plus one moment of design is enough. Legacy and external systems get their own swimlane prefixes (`Legacy.Acme/InvoicePosted`, `Stripe/PaymentReceived`), and uniqueness is enforced across all of them by one validate-time rule:

![The board catches name collisions at design time](/images/fig-05-board-collision.svg)

*Figure 5 — The board catches collisions in the moment of naming. Two events named "Approved" on the same business lane light up red, with a warning speech bubble — meant for two different parts of the business, but indistinguishable in code. The parse-time uniqueness assertion is the backstop for YAML edits that bypass the board.*

> **Name-uniqueness rule:** every fully-qualified event name must appear at most once. Two events declaring the same name fails parse with a pointer to both sites.

One rule. Runs at spec parse, before codegen sees anything. The modeling practice catches collisions in the moment; the assertion is the backstop for direct-edit cases and the gate across business / legacy / external. It's the canonical place to put cross-cutting spec guarantees — model-induced ambiguity belongs at the model boundary, never in tolerant emitters or runtime handlers.

Notice what this adds up to. The work BCs were doing — naming things, grouping them, refusing redundancy, making local consistency visible — *is being done by the act of modeling*, not by an architectural discipline overlaid on top. The bounded context as a separate design artifact is redundant in a model-first practice, because the model is already the artifact a BC design would produce. And it's redundant *singularly*, not in multiplied form: one model, one namespace, one inside.

What's left at the inside is no label at all — just the bare event name, in one shared namespace. The only labels are the ones marking the boundary between the monolith and what isn't the monolith. **The bounded context has shrunk to the wall around the outside.** Not many walls, not nested walls — one wall, where the owned system ends.

## Where this leaves the translation slice

Translation slices live at the boundary between the owned monolith and everything else — third-party APIs, legacy systems, webhook receivers, external integrations. That boundary is where Evans's anti-corruption layer was always doing its real work, and nothing argued here changes that. The foreign side has its own vocabulary, its own change cadence, its own model; if you let it bleed into the monolith unfiltered, you get the corrosion Evans was warning about. Translation slices keep that out.

![The gate at the wall — translation slices only at the periphery](/images/fig-06-gate-at-wall.svg)

*Figure 6 — The gate at the wall. A close-up on one of the city's gates: outside, a legacy system's cryptic foreign-vocabulary events; inside the gate, a translation slice folds, processes, and emits local events; inside the wall, business events flow freely between slices, no translators needed.*

Internally, this topology has *no* translation slices, because it has no internal cross-context bridges to span. The monolith is one context. Cross-context implies two; there's one. If a query in the manuscripts area needs to react to an event from the reviews area, it folds the event directly into its own projection — same namespace, no bridge required. If a process needs to emit a local event in response to events elsewhere in the monolith, it's a same-context Automation pattern (fold + processor + command), not a Translation. The pattern only crosses swimlanes when crossing swimlanes is the point — and inside the monolith, by design, it isn't.

The translation slice exists at the periphery — at legacy and external boundaries — and there it's not on trial; it's doing exactly the work Evans named it for. A translation slice's two jobs (sanitize foreign vocabulary into a local read-model; emit local events in response to a derived view of foreign state) both live at the periphery, both unaffected by anything argued here.

What about the earlier-draft worry that a process manager pattern could emerge internally — a sequence of business events triggering a local command? That's not a translation slice. That's a same-context reaction: an event in the local context triggers a command in the local context, no bridge, no foreign vocabulary to translate. In Event Modeling's vocabulary this is the **Automation pattern**; outside that vocabulary the shape is the same. In a single-internal-context architecture, that pattern subsumes every internal "act on observed state" case the translation slice used to handle in multi-internal-context designs.

## The cost of drawing them anyway

So far the argument has been that internal bounded contexts aren't *needed* in this architecture. There's a separate question worth answering: even if drawing them is harmless, why not draw them anyway, just in case? The honest answer is they aren't harmless. Drawing internal BCs has real, ongoing costs that don't appear in the architecture diagram.

**Design cost.** Strategic design is a discipline. Context maps require workshops, naming arguments, decisions about which patterns connect which contexts (customer-supplier? open host service? published language?). Time spent debating where the line between `Manuscripts` and `Reviews` falls is time not spent modeling the events that actually drive the business. And the line is often arbitrary — there's no fact of the matter about which subdomain a `ReviewerAssigned` event "belongs" to when both touch it. Hours die in those debates.

**Naming and language drift.** Once two contexts share a nominal entity, you need a translation between their renderings of it. The translation accumulates surface area over time. A field added to one context's `Review` requires a decision about whether to surface it in the other context, with what name, through which pattern. Every cross-context type becomes a small contract that has to be maintained, evolved, and explained. None of that work exists if both sides are in one model.

**The boundary becomes political.** This is the cost the canon doesn't talk about openly. Once a BC has a name, a wiki page, and an implicit owner, it acquires institutional weight. Changing it stops being a code refactor and starts being a renegotiation. People defend "their" context's boundaries against encroachment. Boundary disputes become team disputes. The BC map starts dictating team structure rather than reflecting it — Conway's law, running backwards.

**Evolution and migration.** Businesses change. The subdomain you carved a BC around in year one may merge with another in year three, or split in five. When that happens, the BCs you drew need surgery: renames, splits, merges, rebuilding the integration patterns at every bridge. Each migration is expensive precisely because the BC was the anchor in the *design* even when no code anchored to it — the bridges, the published languages, the context map all assumed the boundary was where it was, and undoing that assumption is the bulk of the work. You pay migration cost on a boundary that may have been wrong from the start.

**The "just in case" trap.** *"Let's draw BCs upfront so we can split the monolith later."* It's tempting, and it fails twice. First, the BCs you draw now are unlikely to be the right seams later — business reality shifts, and the boundary that mattered in year one isn't the boundary that matters in year four. Second, in this architecture the retroactive cost of drawing a boundary at the moment it actually matters is low: the event log is the seam, agents make the code refactor mechanical, and the new boundary fits the actual shape of the change rather than a speculative one. You'd be paying expensive design cost upfront for a cheap option you may never need to exercise — and when you exercise it, you'll probably need to redraw it anyway.

**Opportunity cost.** Every hour spent on strategic design is an hour not spent on the model. Model-first practice IS the design work — strategic design overlaid on top duplicates it. Two designs of the same system, maintained by the same humans, with one eventually drifting from the other. The drift itself is the cost.

The earlier sections argued you *can* skip drawing internal BCs. This section argues you *should*. The two arguments stack: in this architecture, drawing internal BCs is unnecessary, and drawing them anyway is actively expensive — in design hours, in language drift, in institutional friction, in migration cost when reality changes the shape of the business under you.

## What this argument doesn't cover

Three honest scoping notes before the close.

**Multi-team scale, and what agentic AI changes about it.** The argument assumes a team that owns the monolith end-to-end, with a unified cadence and vocabulary. The classical worry is that at scale humans fragment vocabulary because cross-cutting work — refactoring shared concepts, propagating naming changes, keeping documentation in sync — gets expensive faster than headcount grows, and teams cope by walling off subdomains. Agents change that cost curve. Cross-cutting work that was punishing at 50 engineers becomes mechanical at 500; the historical threshold for "multi-team enough to need BCs" moves higher in headcount space when the coordination work the threshold was tracking is no longer human-bottlenecked. Where subdomains genuinely move at different *business* cadences — different stakeholders, different release rhythms, different regulatory clocks — the canonical-DDD arguments still apply, just to fewer cases than the old heuristic suggested. The conditions for the argument hold whenever a team owns the monolith end-to-end; agents make that "team" a bigger structural unit than it used to be.

**Onboarding and learning the system.** A new engineer six months in needs to grok the topology. One internal swimlane gives them one model to read end-to-end; the model board is the topology. The post's claim is that this is more navigable than N separate internal contexts each with its own integration patterns. But the navigability burden scales with slice count regardless, and tooling (catalog, search, fan-in reports) carries more of the weight as the system grows.

**The "we might split this monolith later" bet.** Scoping the argument to a fully-owned monolith is, itself, a bet. If extraction is imminent, the BCs you didn't draw become the seams you don't have. The counter: the event log is the seam — anything else is wiring around it — and retroactive boundary-drawing is a re-projection over the existing log, which is cheap by construction (cheaper still if you have agents). Translation slices can be introduced at the new boundary the moment the split happens. The same logic applies to **buy-later** decisions: if a capability you build today might be replaced by an external vendor, the model already tells you what the translation contract would have to look like — a query for events crossing the candidate slices' boundary, both directions. The placement analysis is cheap; the translation work only starts when you commit. But it's a bet worth naming.

## Where this lands

Internal bounded contexts are abolished, not just demoted. Subdomains stay in the business — Manuscripts is a thing people do; Reviews is a thing people do — but they don't cross into the solution space. No separate swimlanes, no separate namespaces, no separate translation seams, no module-per-subdomain, no label-per-subdomain. The code has events, slices, and one lane. The boundary that survives is the *one* outer wall around the monolith: between you and legacy, between you and external.

Translation slices to legacy and external systems keep doing what they always did. They're where the canonical-DDD strategic patterns (ACL, OHS, customer-supplier, published language) earn their keep — exactly because legacy and external are owned by other teams, on other cadences, with their own languages. The conditions Evans wrote into still hold *there*; they just don't hold inside.

The CRUD-era arguments against this position don't apply, because the leak paths aren't here. The canonical-DDD arguments apply at the periphery — where they're sufficient — and don't apply inside, because the conditions don't hold. The bounded context as solution-space architecture has one survivor: the wall around the outside. Everything inside is one neighborhood.

Agents make the consequences fall faster — refactors are mechanical, retroactive corrections are cheap, the cost of being wrong about where to draw a line goes to nearly zero. But the argument doesn't depend on them. Event sourcing with DCB, vertical slices, and a model-first practice are sufficient on their own. Agentic AI is the force multiplier that takes the residual cost from "manageable" to "trivial."

So: do bounded contexts still earn their keep? Strategic DDD does — on the problem-space plane in full, and on the solution-space plane at the periphery. Inside an event-sourced, vertical-sliced, model-first monolith you own end-to-end, the solution-space artifacts have nothing structural left to do. Subdomains stay in the business as a way of talking about the work — that's the problem-space plane, alive and active. The solution-space plane has events, slices, and one lane; the modeling practice and a single assertion do the work strategic design used to do. One wall around the outside. One language inside. The translation slice has a job — at the boundary, doing what it was invented for. Inside, the structural artifact, the discipline, and the bridges between are defending leak paths this architecture has already closed.

---

[^1]: David L. Parnas, *On the Criteria To Be Used in Decomposing Systems into Modules*, Communications of the ACM **15**(12), 1053–1058 (1972), [DOI 10.1145/361598.361623](https://dl.acm.org/doi/10.1145/361598.361623).

[^2]: Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (Addison-Wesley, 2003), ISBN 978-0-321-12521-7.

[^3]: Vaughn Vernon, *Implementing Domain-Driven Design* (Addison-Wesley, 2013), ISBN 978-0-321-83457-7.

[^4]: Vlad Khononov, *Learning Domain-Driven Design* (O'Reilly, 2021); *Balancing Coupling in Software Design* (Addison-Wesley, 2024); companion site [coupling.dev](https://coupling.dev/); [InfoQ — *Why Coupling Is the Hardest Software Engineering Problem*](https://www.infoq.com/presentations/video-podcast-vlad-khononov/); [Tech Lead Journal #188 with Vlad Khononov](https://techleadjournal.dev/episodes/188/).

[^5]: [dcb.events — Dynamic Consistency Boundaries](https://dcb.events/), Sara Pellegrini's reformulation of consistency in event-sourced systems; companion piece by Milan Savić, [*Dynamic Consistency Boundaries*](https://milan.event-thinking.io/2025/05/dynamic-consistency-boundaries.html) at event-thinking.io.

[^6]: [*Kill Aggregate? — An Interview on Dynamic Consistency Boundaries*](https://docs.eventsourcingdb.io/blog/2025/12/15/kill-aggregate-an-interview-on-dynamic-consistency-boundaries/), EventSourcingDB blog, 15 December 2025; see also [dcb.events / topics / aggregates](https://dcb.events/topics/aggregates/).

[^7]: Milan Savić, [*Dynamic consistency boundaries*](https://javapro.io/2025/10/28/dynamic-consistency-boundaries/), JAVAPRO International, 28 October 2025.

[^8]: Anastasiya Kravchuk-Kirilyuk, Fernanda Graciolli, Nada Amin, [*The Modular Imperative: Rethinking LLMs for Maintainable Software*](https://namin.seas.harvard.edu/pubs/lmpl-modularity.pdf) (PDF), LMPL '25 (1st ACM SIGPLAN Workshop on Language Models and Programming Languages, Singapore, October 2025). See also Moti Rafalin, [*LLMs vs. brownfield reality: Why refactoring enterprise systems is hard*](https://vfunction.com/blog/llms-vs-brownfield-reality-why-refactoring-enterprise-systems-is-hard/), vFunction, March 2026.

[^9]: Brian Foote and Joseph Yoder, [*Big Ball of Mud*](http://www.laputan.org/mud/), Chapter 29 in Neil Harrison, Brian Foote, and Hans Rohnert (eds), *Pattern Languages of Program Design 4* (Addison-Wesley, 2000); originally presented at the Fourth Conference on Pattern Languages of Programs (PLoP '97), September 1997.
