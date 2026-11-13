# Decisions log

Dated. Newest first. Each entry says who decided and whether it is settled or open.

## Settled

| Date | Decision | Who |
|---|---|---|
| 2026-11-13 | The app is called **FleetZone** | Patryk |
| 2026-11-13 | Hosted on the same server as Nexus, operated by the friend | Patryk |
| 2026-11-13 | ESI: users have granted all scopes in Nexus | Patryk |
| 2026-11-13 | In-house app; no public launch, no paid features | Patryk |
| 2026-11-13 | Nexus is Alliance Auth | Patryk |
| 2026-11-13 | FleetZone is a side project, theorycraft only, built by the friend. No PT or hackathon time | Patryk |
| 2026-11-13 | Ideas only from other EVE projects. Write all code fresh | Patryk |
| 2026-11-13 | v1 includes all four modules: Fleet ops, Intel, Chain mapper, Dashboard. Each can be enabled or disabled | Patryk |
| 2026-11-13 | Modular, clean UI, quick updates. Fleet-first | Patryk |
| 2026-11-13 | Web first; Tauri later only if needed | Patryk |

## Open (decided at handoff)

| Question | Recommended default | Status |
|---|---|---|
| Integration mode: (a) Django app inside Alliance Auth, (b) external service plus bridge, (c) own SSO | (a) | Open. Conflicts with the earlier "new backend for speed" choice. The friend decides |
| Is the OIDC plugin installed on Nexus? | Unknown | Open |
| When does the Alliance Auth dev instance exist? | Same server; date unknown | Open |
| Trust model: alliance-only or coalition | Alliance-only | Open |
| Do co-hosted apps share an ESI error budget by IP? | Assume yes | Open; test |
| Fleet-member reads need the fleet boss's token? | Use the tracking FC's token | Open; test in Phase 0 |
| Module order and "v1 done" per module | As in the coding plan | Open |
| Notifications: Discord only, or Web Push too | Discord only | Open |
| Attendance source of truth | Fleet snapshot plus FC correction | Open |
| Readiness basis: active or trained level | Active, with trained shown | Open |
| Timer set for the MVP | Structure reinforcement, fuel, unanchoring | Open |
| Raw notification storage | Parsed fields only | Open |
| Companion app in v1 | No | Open |

## Note on the earlier backend choice

Patryk chose "new backend, speed" before Nexus was identified as Alliance Auth. Mode (a) makes the backend a Django app inside Alliance Auth, which overrides that choice. The friend should raise it with Patryk at handoff.
