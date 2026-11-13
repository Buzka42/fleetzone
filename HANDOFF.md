# Handoff: FleetZone

For the friend who builds FleetZone. Read this, then `docs/MECHANICS.md`, then `docs/CODING_PLAN.md`.

## Context in one paragraph

Patryk runs an alliance. The alliance already uses **Nexus**, which is **Alliance Auth** (open-source Django). Nexus holds every member's ESI tokens, because members have granted all scopes. FleetZone is an in-house app that sits on the same server as Nexus and uses those tokens through Nexus, never by exporting them. It is a side project: no pay, no public launch.

## What you decide first (from the owner's list)

Patryk has said he will answer most questions after this handoff, so these are yours to decide. Each has a default in `docs/MECHANICS.md` section 10 and `docs/CODING_PLAN.md` section 15.

1. Integration mode: (a) a Django app inside Alliance Auth, (b) an external service plus a bridge app, or (c) your own EVE SSO. Default: (a).
2. Whether the OIDC plugin is installed on Nexus.
3. The dev instance: when it exists, and where.
4. Which AA groups map to the five roles.
5. Whether a fleet-member read needs the fleet boss's token. This is unverified; test it in Phase 0.

## Start on day one (no dependency on the dev instance)

- Set up the repo, CI and lint.
- Write the EFT parser, with tests on real EFT samples.
- Write the scanner-paste parser and the signature diff, with convergence tests.
- Write the notification YAML parser, defensive per type.
- Write the skill-gap calculator (`gap = max(0, required − active)`), with the dogma attributes listed in `CODING_PLAN.md`.
- Import the pinned SDE build.
- Build the UI shell and the module manifest loader.

Full list: `CODING_PLAN.md` section 12.

## Blocked until the dev instance exists

- Anything that reads ESI through Nexus.
- Login and group mapping.
- The fleet tracking test.
- The structure-timer read, and which character reads it.

## Things not to do

- Do not copy code from the inspiration repos (`docs/INSPIRATIONS.md`). Read for behaviour, then write it fresh.
- Do not log tokens. Do not return tokens from any endpoint.
- Do not call ESI directly from a module. Everything goes through the fetch layer.
- Do not show a failed read as "not ready", "absent" or "clear". It is "no data".
- Do not add a write scope to ESI in v1.

## Questions for Patryk

Send them to Patryk. He said he'll answer after handoff, but he's the one who knows the alliance's trust model and who holds which roles in Nexus.
