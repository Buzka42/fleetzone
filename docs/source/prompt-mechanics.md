You are writing a design document for FleetZone, a fleet-first, modular EVE Online alliance web app. Output ONLY the markdown document body. No tools. No preamble.

Target file: docs/MECHANICS.md

Audience: the friend who builds FleetZone (a competent developer, not an EVE expert) and the owner. The owner will hand over to the friend; the friend will decide open questions.

Context, fixed by the owner:
- Name: FleetZone. Web app. Fleet-first, modular UI, clean interface, quick updates. All four modules exist in v1, but each can be enabled or disabled.
- Hosted on the same server as Nexus (Alliance Auth). Operated by the friend. In-house alliance app, not public.
- Nexus = Alliance Auth. Nexus carries ESI. Users have granted all ESI scopes in Nexus.
- FleetZone is a side project. Ideas only: ideas and behaviour learned from other EVE projects are fine; no copied code or assets.
- Dev instance of Alliance Auth: not yet available; it is expected on the same server.

Your task: explain the app MECHANICS in full, in plain language, module by module, so a builder can understand what the app does before reading code. Use the plan and research below as the factual base. Do not invent facts. Where something is a design choice you propose, tag it [D]. Where the research tagged it unverified, keep the tag [U]. Verified facts keep [V].

Required sections:
1. Purpose and users. Roles (member, FC/fleet boss, director/admin, auditor) and what each can see and do. Map roles to Alliance Auth groups as a proposal [D].
2. System context. A mermaid diagram: members' browsers, FleetZone frontend, FleetZone backend, Alliance Auth, EVE SSO, ESI, zKillboard R2Z2, Discord webhooks, the friend's server. Label what crosses each boundary.
3. Login and consent. Sequence diagram (mermaid): member logs in through Nexus, tokens stored in AA, FleetZone reads through the chosen integration mode. Explain what "consent" means now that all scopes are granted, and what a "lapsed token" looks like to a user.
4. Module mechanics. For each of the four modules:
   - Purpose in one sentence.
   - Step-by-step user flow (click level), including the empty state and the error state.
   - Inputs (what data comes from where, and how fresh it is).
   - Outputs (what the user sees, with the as-of timestamp rule).
   - Rules and calculations: readiness gap, attendance provenance, tri-state decay, mass and lifetime states, signature paste diff, notification triggers, timer display. Be exact about formulas and thresholds, with their tags.
   - Failure modes and what the user sees for each.
5. Modular UI. How modules are enabled or disabled, defaults, what a disabled module hides, how a module declares its routes, data and permissions. Keep this concrete enough to build.
6. Data freshness and the as-of rule. Every displayed number carries when it was fetched. Cache floors from ESI (fleet members 5s, skills about 120s, fittings 300s, notifications 600s). What the UI shows during an outage.
7. Notifications and pings. Triggers, channels (Discord webhook first, Web Push optional), limits, dead-webhook handling, and quiet behaviour.
8. Audit and opsec. What is logged, who can read the log, intel-poisoning defences (reporter reputation, cross-check before broadcast, unverified labels), and what changes now that all scopes are granted.
9. Out of scope for v1, and why.
10. Open questions for the friend at handoff, each with the default that applies if the friend does not change it.

Length: 3000-4000 words. Use headings, short paragraphs, tables where they help. Keep the register plain and direct.

Source material follows. The plan (PLAN-v0-fable.md) and the research findings (02-research-brief-and-findings.md) are below.
