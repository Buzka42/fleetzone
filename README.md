# FleetZone

FleetZone is an in-house, fleet-first, modular web app for one EVE Online alliance. It runs on the same server as Nexus (Alliance Auth) and reads ESI through Nexus.

It has four modules, all in v1, each can be switched on or off:

- **Fleet ops:** doctrines in EFT, readiness per member, Discord pings, attendance from fleet snapshots.
- **Intel:** paste-first local and chat sightings, decaying tri-state system status, ego-centric map.
- **Chain mapper:** wormhole signature paste-diff, mass and lifetime states, routing.
- **Dashboard:** character overviews, notifications, structure timers.

Status: **design only**. No code exists yet. The Alliance Auth dev instance does not exist yet either, so the integration is untested.

## Read in this order

1. [`HANDOFF.md`](HANDOFF.md): what the friend does first, and what is blocked.
2. [`docs/MECHANICS.md`](docs/MECHANICS.md): what the app does, module by module.
3. [`docs/CODING_PLAN.md`](docs/CODING_PLAN.md): how to build it, phases and tasks.
4. [`docs/DECISIONS.md`](docs/DECISIONS.md): what the owner decided, and what is still open.
5. [`docs/INSPIRATIONS.md`](docs/INSPIRATIONS.md): the EVE projects that inspired it, and the ideas-only rule.
6. [`docs/LINKS.md`](docs/LINKS.md): the reference links for Nexus, ESI, SSO and the data sources.
7. [`docs/research/`](docs/research/) and [`docs/source/`](docs/source/): the research and the planning inputs.

## Rules

- **Ideas only.** Behaviour learned from other projects is fine to reimplement. Copying their code or assets is not.
- **Tags.** Facts in the docs carry [V] verified, [U] unverified or [D] design choice.
- **Tokens never leave Alliance Auth.** No endpoint returns an ESI token, and no log line contains one.
- **Every external value shows its as-of time.** ESI is cache-driven, not push.

## Repo

Private, `Buzka42/fleetzone`. Owner: Patryk. Operator: the friend.
