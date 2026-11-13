You are writing the CODING PLAN for FleetZone, a fleet-first, modular EVE Online alliance web app. Output ONLY the markdown document body. No tools. No preamble.

Target file: docs/CODING_PLAN.md

Audience: the friend who builds FleetZone. A competent developer, not an EVE expert. The owner hands over to the friend, and the friend decides open questions. docs/MECHANICS.md (already written, provided below) explains what the app does. Your document says how to build it.

Fixed constraints:
- In-house, hosted on the same server as Nexus (Alliance Auth), operated by the friend.
- Nexus = Alliance Auth. ESI tokens live inside AA. All ESI scopes are already granted by users.
- Integration mode: the plan recommends (a), a Django app inside Alliance Auth that serves a JSON API to a separately built frontend. Keep core logic framework-free so that mode (b) (external service plus bridge app) changes only the transport layer. Make the (a) default explicit and say what changes for (b) and (c).
- Ideas only from other EVE projects: write all code fresh. No copied code, no copied assets.
- Dev instance of AA is not available yet. Phase 0 is gated on it.

Required sections:
1. Summary: what gets built, in what order, and what is blocked on what.
2. Stack defaults [D]. Give one default per layer and a one-line reason: backend language and framework, frontend framework and build tool, styling, database, job scheduler, test runners, lint and format, CI. Note the alternatives briefly. Keep defaults boring and fast to build.
3. Repository layout. A tree, with what each folder owns. Show the framework-free core (EFT parser, paste parsers, SDE importer, gap calculator, signature diff, mass and lifetime state machine, routing, fetch-layer policy) separated from framework glue.
4. Architecture. Layers, dependency direction, where the fetch layer sits, where the module registry is read, and how a module plugs in (manifest, routes, permissions, jobs, entities). Include one short code sketch of the manifest shape and one of the fetch-layer interface. Sketches only; no full implementation.
5. Data model. Tables and key columns for every entity in MECHANICS section 4 and section 5, with keys, indexes and as-of timestamp columns. Include the audit table design (append-only: no update or delete grants at the DB level).
6. API surface. Endpoint list grouped by module, under `/m/<id>/` as MECHANICS section 5 requires. Method, path, role, and a one-line purpose. Include the auth model: session from AA, role check per endpoint, default deny.
7. ESI and fetch layer, step by step. Cache floors, Expires and ETag handling, the two rate systems, the circuit breaker, the headroom rule for Nexus, the scope allow-list enforcement, and what the code does on 420, 429, 5xx and timeouts.
8. Jobs and scheduling. Each job: what it reads, its cadence, its cache floor, what it writes, and how it stops when its module is disabled.
9. Testing strategy. Unit tests with real fixtures (EFT samples, scanner paste samples, notification YAML samples, a recorded ESI response set), contract tests against the API, tests for the Expires rule and the breaker, tests for the signature diff convergence, and the dev-instance integration test. Name each test class by behaviour.
10. Security and operations on the shared server. Secrets handling (no tokens in logs, no tokens in API responses), separate DB user and schema, systemd units, reverse proxy, backups, log rotation, rate-limit headroom shared with Nexus, the kill-switch runbook steps.
11. Phases with tasks. For each phase (0 to 6, as in the plan): a task list where each task has a one-line goal, its dependencies, and acceptance criteria written as checkable statements. Phase 0 includes the dev-instance spike and the verification checklist. Phase 1 includes the module shell, permissions, fetch layer, audit log, SDE import and CI.
12. First two weeks for the friend. A concrete ordered list of tasks that can start before the dev instance exists (parsers, calculator, SDE import, UI shell, fixtures). Mark which tasks need the dev instance.
13. Definition of done, and the review gate per phase: what evidence is attached, and who signs off.
14. Risks and how the plan contains each one (short table).
15. Open decisions the friend must make at handoff, each with the default from MECHANICS section 10 that applies if they do not.

Rules:
- Every claim about an external system carries a tag from the research: [V] verified (from a named source, possibly a third-party mirror), [U] unverified, [D] a design choice in this document. Do not invent endpoints, field names or limits. If you need a value the research does not give, write [U] and a test that will settle it.
- Keep code sketches under 15 lines each.
- Length: 4000-5500 words. Use headings, numbered tasks, tables. Plain, direct language.

Source material follows: docs/MECHANICS.md first, then the plan (PLAN-v0-fable.md), then the research findings (02-research-brief-and-findings.md).
