# FleetZone Coding Plan

**Target:** `docs/CODING_PLAN.md`
**Audience:** the friend who builds and operates FleetZone.
**Companion:** `docs/MECHANICS.md` says what the app does. This document says how to build it.

**Tags**

| Tag | Meaning |
|---|---|
| [V] | Verified in the research from a named source. Several rest on third-party mirrors, so check them against CCP primary docs. |
| [U] | Unverified. Each one names the test or check that settles it. |
| [D] | A design choice in this document. The friend may change it. |

---

## 1. Summary

**What gets built.** A Django app that runs inside Alliance Auth (Nexus) and serves a JSON API [D], plus a separately built static frontend. All game logic lives in a framework-free Python package, so a later move to an external service changes only the transport layer.

**Order**

1. Framework-free core libraries and the UI shell. These need no AA instance and start now.
2. Phase 0: spike on the AA dev instance and work through the verification checklist.
3. Phase 1: foundation (module shell, permissions, fetch layer, audit, SDE import, CI, ops).
4. Phases 2 to 5: Fleet ops, Intel, Chain mapper, Dashboard, one module per phase.
5. Phase 6: alliance beta.

**What is blocked on what**

| Work | Blocked on |
|---|---|
| Core libraries, fixtures, UI shell, CI | Nothing |
| Phase 0 exit | The AA dev instance (it does not exist yet) |
| Any code that touches AA sessions, groups or tokens | Phase 0 exit |
| Fetch-layer transport, all ESI jobs | Phase 0 header capture (task 0.4) |
| Attendance design freeze | The fleet-boss token test (0.5) |
| Structure timer board | The nominated character and its corp role (0.5) |
| Intel name resolution | The lookup endpoint, which is [U] (0.5) |
| Phases 2 to 5 | Phase 1 gate |

---

## 2. Stack defaults [D]

| Layer | Default | Reason | Alternatives |
|---|---|---|---|
| Backend language | Python, the version Nexus runs | Mode (a) is an in-process Django app [V] | None in mode (a) |
| Backend framework | Django, the version AA pins; plain views with one small endpoint decorator | Adds no dependency to AA's environment, so AA upgrades break less | django-ninja, Django REST framework |
| Core logic | Pure Python package, standard library only | Portable to modes (b) and (c) | None |
| Frontend | React with TypeScript | Largest pool of help and examples | Svelte, Vue |
| Build tool | Vite | Fast and needs little configuration | None worth the time |
| Client data | TanStack Query | Handles refetching and stale data, which fits the as-of rule | Plain `fetch` |
| Styling | Tailwind CSS | No custom CSS architecture to design | CSS modules |
| Database | The DB server Nexus already runs, with a separate database and user | One server to back up; isolation by grants | A dedicated PostgreSQL instance |
| Job scheduler | The Celery setup AA runs, on a dedicated `fleetzone` queue | No second scheduler on the host | systemd timers calling management commands |
| Backend tests | pytest, pytest-django | Standard | unittest |
| Frontend tests | Vitest; Playwright for a few end-to-end flows | Vitest shares the Vite config | Jest |
| Lint and format | ruff (lint and format), mypy on `core/`; ESLint and Prettier | One Python tool; strict types where the logic lives | Biome for the frontend |
| CI | GitHub Actions | Free for this size | Any runner that runs the same commands |

Two [U] items sit under this table: that AA runs Celery, and which database engine Nexus uses. Neither is in the research. Task 0.1 records both. Use only the Django ORM so the engine does not matter.

---

## 3. Repository layout

```
fleetzone/
├── core/                      # framework-free; imports nothing from Django or AA
│   ├── eft/                   # EFT fit parser
│   ├── paste/                 # local list, chat lines, probe scanner parsers
│   ├── sde/                   # SDE importer: reads the export, emits plain rows
│   ├── readiness/             # skill merge and gap calculator
│   ├── sigdiff/               # New / Updated / Missing diff and merge
│   ├── wormhole/              # mass and lifetime state machine, auto-ageing
│   ├── routing/               # graph build and pathfinding with filters
│   ├── intel/                 # decay, triage labels, reputation, broadcast check
│   ├── notif/                 # notification YAML parsers, timer rules
│   ├── fetchpolicy/           # Expires rule, rate state, breaker, scope allow-list
│   └── ports.py               # Protocols the glue must implement (clock, token, transport)
├── backend/
│   └── fleetzone/             # the Django app installed into AA
│       ├── shell/             # registry, auth adapter, permissions, audit, notify, SDE tables
│       ├── fetch/             # fetch layer: policy from core + AA transport
│       ├── modules/
│       │   ├── fleet/         # manifest.py, models.py, views.py, jobs.py
│       │   ├── intel/
│       │   ├── chain/
│       │   └── dash/
│       ├── transport/         # URL wiring, session auth, JSON encoding (changes in mode b)
│       ├── auth_hooks.py      # AA url and menu hooks [V]
│       └── dbrouter.py        # sends FleetZone models to their own database
├── frontend/
│   ├── src/shell/             # nav, banners, as-of components, API client
│   ├── src/modules/<id>/      # one folder per module, lazy-loaded
│   └── src/workers/           # chat-log parser for the directory picker
├── fixtures/                  # real samples shared by backend and frontend tests
│   ├── eft/  scanner/  local/  chatlog/  notifications/  esi/  sde/
├── ops/                       # systemd units, proxy snippet, backup script, runbooks
├── docs/                      # MECHANICS.md, CODING_PLAN.md, gates/, decisions/
└── .github/workflows/ci.yml
```

Ownership rules [D]:

- `core/` owns every rule that has a number or a state in MECHANICS. It takes and returns plain data.
- `backend/` owns storage, HTTP, sessions and job wiring. It contains no game rules.
- A CI test fails the build if anything under `core/` imports `django`, `allianceauth` or `esi`.
- The picker's chat-line parser exists twice (Python and TypeScript). Both run against the same files in `fixtures/chatlog/`.

---

## 4. Architecture

### Layers and dependency direction

```
frontend  ──HTTP/JSON──▶  transport  ──▶  modules  ──▶  shell + fetch  ──▶  core
                                                             │
                                                             └──▶  AA (session, groups, tokens)
```

- Arrows point at what a layer may import. `core` imports nothing.
- `core/ports.py` defines the interfaces `core` needs: clock, token provider, ESI transport. The glue implements them. Tests use fakes.
- Modules import the shell and core. A module never imports another module (MECHANICS section 5).

### Where the fetch layer sits

The fetch layer is a shell service. Its decisions (may this call be made now?) live in `core/fetchpolicy`. Its I/O lives in `backend/fleetzone/fetch` and wraps AA's django-esi client and Token model in-process [V]. No other code reaches ESI. CI greps for ESI client imports outside `fetch/`.

### Where the module registry is read

| When | What is read | Effect |
|---|---|---|
| Django app start | Every `modules/*/manifest.py` | Routes are registered, permissions collected, jobs registered. A route with no permission fails startup. |
| Every request | `module_setting` (cached for 10s [D]) | A disabled module's paths return 404 |
| Start of every job run | `module_setting` (not cached) | A disabled module's job exits without work |
| Every fetch call | The manifest `scopes` and `module_setting` | A call outside the allow-list or from a disabled module is refused |
| `GET /shell/modules` | Manifests and settings | The frontend builds nav and lazy-loads module bundles |

### How a module plugs in

A module is a folder with one manifest. The sketch below shows the shape, not final code.

```python
MANIFEST = ModuleManifest(
    id="fleet", name="Fleet ops", icon="users", default_enabled=True,
    routes=[Route("GET", "doctrines", view=list_doctrines, perm="fleet.view")],
    permissions={"fleet.view": "recruit", "fleet.ping": "fc"},
    scopes={"esi-skills.read_skills.v1", "esi-fleets.read_fleet.v1"},
    entities=[Doctrine, Fit, ReadinessSnapshot, Fleet, AttendanceRow, Ping],
    jobs=[Job("tracking_snapshot", run=snapshot_fleets, every_s=60, floor_s=5)],
    notifications=["fleet.ping"],
    depends_on=["shell.sde", "shell.notify"],
)
```

- `permissions` names the lowest role that holds each permission. Recruit, member, FC and director form a chain. Auditor stands apart and holds only audit permissions.
- `routes` mount under `/m/<id>/`. The frontend folder `src/modules/<id>/` mirrors them.

The fetch-layer interface:

```python
@dataclass(frozen=True)
class EsiRequest:
    module_id: str; operation: str; scope: str | None
    character_id: int | None; params: Mapping[str, Any]

@dataclass(frozen=True)
class FetchResult:
    state: str        # fresh | cached | stale | no_token | lapsed | opted_out | denied | refused
    data: Any | None; as_of: datetime | None
    expires: datetime | None; reason: str | None

class FetchLayer(Protocol):
    def get(self, req: EsiRequest) -> FetchResult: ...
```

`get` never raises for an expected failure. The caller always receives a state, which is how "no data" stays distinct from "not ready".

### What changes per integration mode

| Part | (a) default | (b) external service plus bridge | (c) own SSO |
|---|---|---|---|
| `core/` | Unchanged | Unchanged | Unchanged |
| `transport/` | Django views under AA, AA session | Standalone service; login through the AA OIDC plugin (`/o/authorize/`, `/o/token/`, `/o/userinfo/`, groups claim) [V] | Standalone; EVE SSO Authorization Code with client secret [V] |
| Auth adapter | Reads AA user and groups in-process | Reads the groups claim | Own user table and role admin |
| Token provider | django-esi Token model [V] | None; a bridge app in AA performs the call and returns data, `Expires` and `ETag`, never tokens [D] | Own encrypted token store; persist any new refresh token on every refresh, since rotation is [U] |
| Database | Separate database on Nexus's DB server | Own database | Own database |
| Consent | First-run notice | First-run notice | A real second EVE consent; members grant scopes again |
| Rate budget | Shared with Nexus | Shared, behind the bridge | Separate registration; sharing by IP is [U] |

---

## 5. Data model

Conventions [D]:

- Tables are prefixed `fz_`. They live in a separate database reached through `dbrouter.py`.
- AA users and characters are referenced by plain integer ID, with no foreign key, because the tables are in another database.
- Every row derived from ESI has `as_of` (UTC, when it was fetched). Rows entered by a person have `created_at` and `created_by`.
- There is no tenant column. FleetZone serves one alliance, and coalition sharing is cut.
- PK is a surrogate `id` unless stated.

### Shell

| Table | Key columns | Keys and indexes |
|---|---|---|
| `fz_module_setting` | `module_id`, `enabled`, `changed_by`, `changed_at` | PK `module_id` |
| `fz_user_pref` | `user_id`, `hidden_modules` (JSON), `pinned_centre_system_id` | PK `user_id` |
| `fz_role_mapping` | `role`, `aa_group_name` | Unique (`role`, `aa_group_name`) |
| `fz_consent_ack` | `user_id`, `notice_version`, `acked_at` | Unique (`user_id`, `notice_version`) |
| `fz_character_optout` | `character_id`, `user_id`, `set_at` | PK `character_id` |
| `fz_token_state` | `character_id`, `scope`, `state` (ok, lapsed, none), `last_ok_at`, `lapsed_since`, `as_of` | Unique (`character_id`, `scope`); index `state` |
| `fz_esi_cache` | `operation`, `params_hash`, `character_id`, `etag`, `expires`, `body` (JSON), `as_of`, `last_status` | Unique (`operation`, `params_hash`, `character_id`); index `expires` |
| `fz_rate_state` | `route_group`, `remaining`, `limit`, `window_reset_at`, `breaker_state`, `opened_at`, `reason`, `as_of` | PK `route_group`; the row `_legacy` holds the error budget |
| `fz_notification_channel` | `name`, `kind`, `secret_ref`, `enabled`, `hourly_cap`, `disabled_reason`, `disabled_at` | Unique `name`. The webhook URL is stored encrypted and is never serialised. |
| `fz_notification_send` | `channel_id`, `trigger`, `subject_key`, `data_as_of`, `status`, `http_status`, `sent_by`, `sent_at`, `folded_count` | Index (`trigger`, `subject_key`, `sent_at`) for the 10-minute fold |
| `fz_sde_build` | `build_number`, `imported_at`, `active` | PK `build_number` |
| `fz_sde_type` | `build`, `type_id`, `name`, `group_id`, `category_id` | Unique (`build`, `type_id`); index (`build`, `name`) |
| `fz_sde_skill_req` | `build`, `type_id`, `skill_type_id`, `level` | Unique (`build`, `type_id`, `skill_type_id`) |
| `fz_sde_system` | `build`, `system_id`, `name`, `security`, `region_id` | Unique (`build`, `system_id`) |
| `fz_sde_gate` | `build`, `from_system_id`, `to_system_id` | Index (`build`, `from_system_id`) |
| `fz_sde_wormhole_type` | `build`, `code`, `type_id`, `attributes` (JSON) | Unique (`build`, `code`) |

### Fleet ops

| Table | Key columns | Keys and indexes |
|---|---|---|
| `fz_doctrine` | `name`, `archived`, `created_by`, `created_at` | Unique `name` |
| `fz_fit` | `doctrine_id`, `name`, `hull_type_id`, `eft_text`, `status` (complete, incomplete), `warnings` (JSON), `sde_build` | Index `doctrine_id` |
| `fz_fit_item` | `fit_id`, `type_id` (nullable), `raw_name`, `slot`, `quantity` | Index `fit_id` |
| `fz_fit_skill_req` | `fit_id`, `skill_type_id`, `level`, `sde_build` | Unique (`fit_id`, `skill_type_id`) |
| `fz_character_skill` | `character_id`, `skill_type_id`, `trained_level`, `active_level`, `as_of` | Unique (`character_id`, `skill_type_id`) |
| `fz_readiness_snapshot` | `character_id`, `fit_id`, `ready`, `gaps` (JSON: skill, required, active, trained), `as_of` | Unique (`character_id`, `fit_id`); index (`fit_id`, `ready`) |
| `fz_fleet` | `esi_fleet_id`, `started_by`, `token_character_id`, `started_at`, `stopped_at`, `paused_at`, `pause_reason`, `snapshots_taken` | Index `stopped_at` |
| `fz_attendance_row` | `fleet_id`, `character_id`, `first_seen`, `last_seen`, `snapshots_seen`, `ship_type_id`, `provenance` (fleet_snapshot, manual, rsvp), `superseded`, `corrected_by`, `corrected_at`, `as_of` | Index (`fleet_id`, `character_id`). Corrections add a row and mark the old one `superseded`. |
| `fz_ping` | `channel_id`, `sent_by`, `text`, `send_id`, `sent_at` | Index `sent_at` |

### Intel

| Table | Key columns | Keys and indexes |
|---|---|---|
| `fz_intel_report` | `system_id`, `kind` (hostile, clear), `reporter_user_id`, `source` (paste, picker, companion), `verification` (verified, unverified), `verified_by`, `counts` (JSON per label), `created_at` | Index (`system_id`, `created_at`) |
| `fz_intel_sighting` | `report_id`, `character_name`, `character_id` (nullable), `label` (watchlisted, active, unknown) | Index `report_id` |
| `fz_system_status` | `system_id`, `state` (hostile, clear), `hold_until`, `contested_until`, `last_report_id` | PK `system_id`. Unknown is the absence of a row or a passed `hold_until`. |
| `fz_watchlist_entry` | `entity_kind` (character, corp, alliance), `entity_id`, `note`, `created_by` | Unique (`entity_kind`, `entity_id`) |
| `fz_reporter_reputation` | `user_id`, `confirmed`, `contradicted` | PK `user_id` |
| `fz_zkill_cache` | `entity_kind`, `entity_id`, `summary` (JSON), `as_of`, `expires` | Unique (`entity_kind`, `entity_id`) |
| `fz_intel_setting` | `hostile_hold_s`, `clear_hold_s`, `radius`, `picker_enabled` | Single row |

### Chain mapper

| Table | Key columns | Keys and indexes |
|---|---|---|
| `fz_chain_map` | `name`, `created_by`, `created_at` | Unique `name` |
| `fz_signature` | `map_id`, `system_id`, `sig_id`, `downtime_epoch` (date), `group`, `type`, `first_seen`, `last_seen`, `last_seen_by`, `not_seen_by`, `not_seen_at`, `deleted_at`, `created_by` | Unique (`map_id`, `system_id`, `sig_id`, `downtime_epoch`) |
| `fz_connection` | `map_id`, `signature_id`, `from_system_id`, `to_system_id` (nullable), `type_code`, `type_in_sde`, `mass_state`, `mass_set_by`, `mass_set_at`, `life_state`, `life_flagged_at`, `life_set_by`, `version` | Index (`map_id`, `from_system_id`), (`map_id`, `to_system_id`) |
| `fz_connection_history` | `connection_id`, `field`, `before`, `after`, `changed_by`, `changed_at` | Index (`connection_id`, `changed_at`) |
| `fz_avoid_system` | `user_id`, `system_id` | Unique (`user_id`, `system_id`) |

### Dashboard

| Table | Key columns | Keys and indexes |
|---|---|---|
| `fz_character_state` | `character_id`, `system_id`, `ship_type_id`, `online`, `location_as_of`, `ship_as_of`, `online_as_of` | PK `character_id` |
| `fz_notification` | `character_id`, `esi_notification_id`, `type`, `sent_at`, `parsed` (JSON, nullable), `raw_text` (unknown types only), `as_of` | Unique (`character_id`, `esi_notification_id`) |
| `fz_structure` | `structure_id`, `corp_id`, `name`, `type_id`, `system_id`, `state`, `state_timer_end`, `fuel_expires`, `unanchors_at`, `as_of` | PK `structure_id`; index `state_timer_end`, `fuel_expires` |
| `fz_dash_setting` | `structure_character_id`, `nominated_by`, `nominated_at` | Single row |

The field names `state`, `state_timer_end`, `fuel_expires` and `unanchors_at` are [V]. The other structure columns are [U] until the 0.4 capture shows the real response. The timer board is a query over `fz_structure`, not a table.

### Audit table

`fz_audit_event`: `id`, `at`, `actor_user_id`, `action`, `module_id`, `subject_type`, `subject_id`, `before` (JSON), `after` (JSON), `request_id`. Indexes: `at`, (`actor_user_id`, `at`), (`subject_type`, `subject_id`).

Append-only is enforced at three levels [D]:

1. **Database grants.** Two DB users exist. `fleetzone_owner` runs migrations at deploy time only. `fleetzone_app` is the runtime user, and on `fz_audit_event` it holds `INSERT` and `SELECT` only. It has no `UPDATE`, `DELETE`, `TRUNCATE` or `ALTER`.
2. **Model.** `save()` on an existing row and `delete()` raise.
3. **Test.** `TestAuditTableRejectsUpdateAndDelete` connects as `fleetzone_app`, attempts both statements, and expects a permission error from the database.

Audit writes happen in the same transaction as the change they describe. Token values and webhook URLs never appear in `before` or `after`.

---

## 6. API surface

Mount point [D]: pages under `/fleetzone/`, API under `/fleetzone/api/`. Paths below are relative to the API root. Role is the lowest role allowed. "Own" means only the caller's data.

### Auth model

- The session is AA's session cookie. FleetZone issues no credentials of its own.
- A request with no session gets 401, and the frontend redirects to Nexus login.
- Roles are resolved on each request from AA groups through `fz_role_mapping`. A user with no mapped group gets 403 on everything except `GET /shell/me`.
- Recruit is `Member` whose account is under 30 days old [D]. The default source is the Django user's `date_joined`. Whether that matches "joined the alliance" on Nexus is [U]; check it in 0.6.
- Every route declares a permission in its manifest. Startup fails on a route without one. This is the default deny.
- Unsafe methods require Django's CSRF token. The frontend is same-origin, so no CORS is configured.
- No endpoint returns a token, a webhook URL or another member's dashboard.

### Shell

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET | `/shell/me` | Any session | Identity, roles, characters with token state, consent status |
| GET | `/shell/modules` | Recruit | Manifests (public fields) and enabled state |
| PUT | `/shell/modules/{id}` | Director | Enable or disable; audited |
| POST | `/shell/consent` | Recruit | Acknowledge the first-run notice |
| PUT | `/shell/characters/{id}/optout` | Recruit, own | Set or clear opt-out |
| PUT | `/shell/prefs` | Recruit, own | Hide modules from own nav |
| GET | `/shell/status` | Recruit | Breaker state and outage banner data |
| GET, PUT | `/shell/role-mapping` | Director (GET also auditor) | View or change the group mapping |
| GET, POST | `/shell/channels` | Director | List or create webhooks; the URL is write-only |
| DELETE | `/shell/channels/{id}` | Director | Remove a webhook |
| GET | `/shell/audit` | Auditor, director | Read the log, paged; the read is logged |
| POST | `/shell/audit/export` | Auditor | Export; logged |

### Fleet ops: `/m/fleet/`

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET | `/doctrines` | Recruit | List doctrines and fits |
| POST | `/fits/preview` | Director | Parse EFT text; return slots and warnings; saves nothing |
| POST | `/doctrines` | Director | Create a doctrine |
| PUT, DELETE | `/doctrines/{id}` | Director | Edit or archive |
| POST | `/doctrines/{id}/fits` | Director | Save a parsed fit |
| GET | `/readiness/me` | Recruit, own | Own characters against all fits |
| GET | `/doctrines/{id}/readiness` | FC | Alliance table with coverage line |
| POST | `/pings` | FC | Send a ping to a chosen channel |
| GET | `/pings/{id}` | FC | Delivery result |
| POST | `/tracking` | FC | Start tracking with the caller's token |
| POST | `/tracking/{id}/stop` | FC | Stop tracking |
| GET | `/fleets` | FC | Tracked fleets |
| GET | `/fleets/{id}/attendance` | FC | Attendance sheet with provenance |
| POST | `/fleets/{id}/attendance/corrections` | FC | Add or remove a row; audited |

### Intel: `/m/intel/`

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET | `/map` | Member | Systems within the radius of the centre, with status and hold deadlines |
| POST | `/paste/preview` | Member | Parse and triage a paste; saves nothing |
| POST | `/reports` | Member | Report hostile or clear |
| GET | `/reports` | Member | Feed |
| GET | `/systems/{id}` | Member | Detail panel |
| POST | `/reports/{id}/confirm` | FC | Confirm for broadcast |
| POST | `/sightings` | Member | Derived sightings from the picker; 404 while the picker is off |
| GET, POST | `/watchlist` | Director | Read or add entries |
| DELETE | `/watchlist/{id}` | Director | Remove an entry |
| GET, PUT | `/settings` | Director | Hold times, radius, picker toggle |

### Chain mapper: `/m/chain/`

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET, POST | `/maps` | Member | List or create maps |
| GET | `/maps/{id}` | Member | Systems, signatures and connections with derived states |
| POST | `/maps/{id}/systems/{sys}/signatures/preview` | Member | Diff a paste into New, Updated, Missing, skipped |
| POST | `/maps/{id}/systems/{sys}/signatures/apply` | Member | Merge; delete only ticked rows |
| DELETE | `/signatures/{id}` | Member | Delete; returns 409 `needs_confirmation` under the 10-minute rule |
| POST | `/signatures/{id}/confirm-delete` | Director | Confirm a delete of another user's recent signature |
| POST | `/connections` | Member | Create a connection from a wormhole signature |
| PATCH | `/connections/{id}` | Member | Set type, mass, lifetime or far side |
| GET | `/connections/{id}/history` | Member | Edit history |
| GET | `/route` | Member | Path with per-hop state and as-of |
| GET, PUT | `/avoid` | Member, own | Avoid-system list |

### Dashboard: `/m/dash/`

| Method | Path | Role | Purpose |
|---|---|---|---|
| GET | `/characters` | Member, own | Character cards |
| GET | `/notifications` | Member, own | Notification feed |
| GET | `/timers` | FC | Timer board |
| GET | `/timers/{structure_id}` | FC | Source, raw values, as-of |
| PUT | `/settings/structure-character` | Director | Nominate the reading character |

Every response that carries fetched data includes `as_of` per value or per row.

---

## 7. ESI and fetch layer, step by step

What `FetchLayer.get` does for each request, in order:

1. **Module check.** If the module is disabled, return `denied`. No network.
2. **Scope allow-list.** If `req.scope` is not in the module's manifest `scopes`, return `denied` and write an audit event. No network. This is the control that keeps wallet, assets and mail unread.
3. **Opt-out.** For readiness and dashboard reads, if the character is opted out, return `opted_out`.
4. **Cache.** Look up `fz_esi_cache`. If now is before `expires`, return `cached` with the stored `as_of`. No code path can skip this step; that is how the "never before `Expires`" rule [V] is enforced.
5. **Floor.** If the cache row has no `expires` yet, apply the known floor [V]: fleet members 5s, skills about 120s, fittings 300s, notifications 600s. For endpoints the research gives no floor for, use 300s [D] until the first response supplies `Expires`.
6. **Breaker.** If the breaker for the route group or the global breaker is open, return `stale` with the cached body and the reason.
7. **Headroom.** If the next call would cross the headroom line (below), open the breaker and return `stale`.
8. **Token.** Ask AA for a valid token for the character with the scope [V for the Token model]. No token returns `no_token`. A failed refresh returns `lapsed` and updates `fz_token_state`. Neither makes an ESI call.
9. **Call.** Send the request with `If-None-Match` when an `ETag` is stored. Timeout 10s [D]. One attempt, with no retry inside the request.
10. **Record.** Update `fz_rate_state` from the response headers before anything else.
11. **Store.**
    - 200: store body, `ETag`, `Expires`, set `as_of` to now, and return `fresh`.
    - 304: keep the body, update `Expires`, set `as_of` to now, and return `fresh`.

### The two rate systems

| System | What the research gives | How the layer tracks it |
|---|---|---|
| Legacy error budget | About 100 non-2xx responses per minute, then 420 [V] | One `_legacy` row. A local count of FleetZone's own non-2xx responses per minute, plus the remaining value from headers if they are present. |
| Per-route-group buckets | Costs: 2xx is 2, 3xx is 1, 4xx is 5, 5xx is 0; read from `X-Ratelimit-*` headers [V] | One row per route group, updated from every response. |

Unknowns, each settled by the 0.4 header capture:

- The exact header names in both systems [U].
- Bucket sizes and window lengths per group [U].
- Whether the django-esi client exposes response headers and `ETag` to the caller, and whether it already retries or caches by itself [U]. If it retries, turn that off or account for it, so one logical call is one request.
- The status code returned when a bucket is empty. This plan assumes 429 [U].

### Headroom rule for Nexus

Default from MECHANICS question 17: the breaker opens at half the remaining budget [D]. In code:

- **Buckets.** FleetZone stops calling a route group when `remaining` falls below 50% of `limit`.
- **Legacy budget.** FleetZone stops all calls when the reported remaining error budget falls below 50%, or when FleetZone's own non-2xx count reaches 10 in a minute [D].
- A 4xx costs 5 tokens [V]. So a character and scope that returned 401 or 403 is not retried until its token changes or 1 hour passes [D].

### Circuit breaker

| State | Behaviour |
|---|---|
| Closed | Calls pass |
| Open | No calls for that group. Results are `stale`. A banner shows through `/shell/status`. |
| Half-open | After the window reset time from headers, or 60s if unknown [D], one probe call passes. Success closes the breaker; failure reopens it for double the time, up to 15 minutes [D]. |

### Responses and failures

| Event | What the code does |
|---|---|
| 420 | Opens the global breaker at once. No call of any kind until the reset time from headers, or 60s if absent [D]. Alerts directors in-app. Never retries. |
| 429 (assumed [U]) | Opens that route group's breaker. Honours a retry-after value if the response carries one [U]; otherwise waits for the window reset. |
| 401, 403 | Marks the token state `lapsed` or the read `refused`. No retry, as above. |
| 404 and other 4xx | Returns `refused` with the status. No retry. A repeat is a bug, and the own-error counter catches it. |
| 5xx | Costs 0 bucket tokens [V] but counts in the legacy error budget as a non-2xx [V]. No immediate retry. The job's next run doubles its interval, up to 15 minutes [D]. Three 5xx in a row on a group open its breaker for 5 minutes [D]. |
| Timeout | Treated as a 5xx locally. Whether ESI counts it is [U]. |

In every failure case the caller gets the last cached body with its old `as_of` and a reason. No status is derived from a failed read.

---

## 8. Jobs and scheduling

All jobs run on the `fleetzone` Celery queue [D], served by one dedicated worker with a concurrency of 2 [D]. Nexus's own workers never run FleetZone jobs, and stopping the FleetZone worker stops all FleetZone ESI traffic.

**How every job stops when its module is disabled.** Jobs are registered through one wrapper. The wrapper reads `fz_module_setting` at the start of each run and returns at once if the module is off. The fetch layer refuses calls from a disabled module as a second lock (section 7, step 1). `TestDisabledModuleMakesNoEsiCalls` covers both.

| Job | Reads | Cadence [D] | Cache floor | Writes |
|---|---|---|---|---|
| `fleet.readiness_refresh` | Skills for every character with a token that is not opted out | Hourly, spread across the hour; on demand when an FC opens a table | About 120s [V] | `fz_character_skill`, `fz_readiness_snapshot` |
| `fleet.tracking_snapshot` | Fleet members, with the tracking FC's token | Every 60s while a fleet is tracked | 5s [V] | `fz_fleet`, `fz_attendance_row` |
| `intel.zkill_lookup` | zKillboard API for pasted pilots | Queue, at most 1 request per second | 1h local cache [D]; zKill quotas are [U] | `fz_zkill_cache`, sighting labels |
| `shell.location_read` | Caller's location, for Intel and Chain | On demand while a map page is open | From `Expires`; the value is [U] | `fz_character_state` |
| `dash.character_refresh` | Location, ship, online and skills for the caller's own characters | On demand when the owner opens Dashboard | From `Expires`; skills about 120s [V] | `fz_character_state`, `fz_character_skill` |
| `dash.notifications` | Notifications for the caller's own characters and for the nominated character | On demand; every 10 minutes for the nominated character | 600s [V] | `fz_notification` |
| `dash.structures` | Corp structures, with the nominated character | Every 15 minutes; the layer skips until `Expires` | From `Expires`; the value is [U] | `fz_structure`; raises reinforcement and fuel triggers |
| `shell.notify_dispatch` | The outbox of pending sends | Queue | Discord `X-RateLimit` headers [V] | `fz_notification_send`; disables a channel on 404 [V] |
| `shell.token_state_sync` | AA token state, no ESI call | Every 15 minutes | None | `fz_token_state` |
| `shell.housekeeping` | Own tables | Daily | None | Removes expired cache rows and timers passed for over 1 hour |

`shell.location_read` belongs to the shell and is declared in both the Intel and Chain manifests. It runs while either module is enabled.

No job handles intel decay or wormhole auto-ageing. Both are computed on read from stored times (`hold_until`, `life_flagged_at`), so a late or stopped worker cannot leave a wrong state.

`fleet.tracking_snapshot` pauses a fleet, with a reason, when the read is refused. It does not retry a refused read (4xx costs 5 tokens [V]).

---

## 9. Testing strategy

### Fixtures

All fixtures are real samples, committed under `fixtures/`, with character names replaced where they are not public.

| Folder | Content | Source |
|---|---|---|
| `eft/` | At least 20 fits: T3 with subsystems, drones, cargo, empty slots, a broken paste | Copied from the game client by an alliance member |
| `scanner/` | Probe scanner pastes: partial scan, full scan, one with extra columns | Game client. The base layout is tab-separated ID, group, type [V]; extra columns are [U]. |
| `local/`, `chatlog/` | Local member list pastes; chat log files in UTF-16LE with BOM, lines as `[YYYY.MM.DD HH:MM:SS] Sender > message` [V] | Game client. The local-list paste layout is [U]; the sample settles it. |
| `notifications/` | Notification YAML per MVP type, plus unknown types | Dev instance capture |
| `esi/` | Recorded responses with full headers, tokens stripped | Task 0.4 |
| `sde/` | A cut-down slice of one pinned build | Official SDE [V] |

### Test classes

| Class | Behaviour proven |
|---|---|
| `TestEftParserReadsRealFits` | Each fixture yields the expected hull, slots and drone counts |
| `TestEftParserRejectsUnknownHullAndKeepsUnknownModules` | Unknown hull is an error with the line; unknown module is a warning and the fit is `incomplete` |
| `TestSkillRequirementsMergeToHighestLevel` | Attribute pairs 182/277 through 1290/1288 [V] merge per fit |
| `TestGapUsesActiveLevelAndTreatsMissingAsZero` | The gap formula, the Alpha case, the missing skill |
| `TestReadinessCoverageSeparatesLapsedNoTokenOptedOut` | The three states never count as not ready |
| `TestScannerPasteSkipsMalformedRows` | Bad rows are listed; valid rows still parse |
| `TestSignatureDiffNeverDeletesUnticked` | Missing rows survive Apply |
| `TestSignatureDiffConverges` | Property test: two users applying overlapping pastes in any order reach the same store |
| `TestSignatureDiffNeverBlanksKnownValue` | A blank in the paste does not overwrite a value |
| `TestPostDowntimeSignatureIsNewRecord` | The same ID in a new epoch is a new row |
| `TestLifetimeAutoAgesButNeverExpires` | 20h to EOL, 4h to "likely expired", never Expired |
| `TestMassNeverChangesWithoutUser` | No time-based mass change |
| `TestRouteExcludesRiskyEdgesByDefault` | EOL and Critical excluded, toggle includes, expired always excluded |
| `TestIntelDecayFromStoredDeadline` | Hostile 10 minutes, clear 5, contested 2, with a fake clock |
| `TestBroadcastRequiresCrossCheck` | Unverified reports never reach the outbox |
| `TestNotificationParserFallsBackToRawText` | Unknown type and broken YAML both render raw; a fuzz run finds no crash |
| `TestFetchNeverCallsBeforeExpires` | Fake transport and clock: zero calls before `Expires`, one after |
| `TestFetchSendsEtagAndHandles304` | `If-None-Match` is sent; a 304 keeps the body and updates `as_of` |
| `TestFetchRefusesUndeclaredScope` | No transport call; audit event written |
| `TestBreakerOpensAtHalfBudget` | Replays recorded headers; the breaker opens at the line |
| `TestBreakerHaltsEverythingOn420` | One 420 opens the global breaker; no further calls |
| `TestServerErrorsBackOffWithoutRetry` | 5xx and timeouts: one call per run, doubling interval |
| `TestDisabledModuleMakesNoEsiCalls` | Jobs exit; the fetch layer denies |
| `TestEveryRouteDeclaresPermission` | Walks the URL map; fails on any route without one |
| `TestRoleMatrix` | Each role against each endpoint, expecting the status from section 6 |
| `TestDisabledModuleReturns404` | For every module route |
| `TestNoTokenInResponsesOrLogs` | A seeded marker token never appears in any response body or log line in the suite |
| `TestAuditTableRejectsUpdateAndDelete` | DB-level denial as the runtime user |
| `TestDeadWebhookDisablesChannel` | 404 disables the channel, alerts directors, sends nothing more |
| `TestCoreImportsNoFramework` | The import boundary |

**Contract tests.** The API is described by one schema file generated from the manifests. Backend tests check every response against it. The frontend's API client types are generated from the same file, so drift fails CI on one side or the other.

**Dev-instance integration test** (`TestLoginGroupsAndOneEsiCallOnDevInstance`, run by hand, not in CI): log in through AA, resolve roles, make one skills read through the fetch layer, repeat it at once and assert that no second ESI call is made, then grep the logs for token strings.

---

## 10. Security and operations on the shared server

**Secrets**

- ESI tokens stay in AA's Token model [V]. FleetZone holds a token only inside one fetch call and never stores, logs or returns it.
- A logging filter redacts `Authorization` headers and anything shaped like a JWT or a Discord webhook URL. The filter is attached to every FleetZone logger and to the dedicated worker.
- Webhook URLs are encrypted at rest with a key from an environment file readable only by the service user (mode 600). API responses show a channel name and state, never the URL.
- DB passwords and the encryption key live in that environment file, not in the repository.

**Database**

- A separate database (schema) on Nexus's server, and two users: `fleetzone_owner` for migrations and `fleetzone_app` for runtime. Neither has any grant on AA's database.
- AA's own DB user gets no grant on FleetZone's database. The database router keeps the two apart in code.

**Processes (systemd)**

How Nexus runs its web and worker processes is [U]; record it in 0.1. FleetZone adds:

| Unit | Purpose |
|---|---|
| `fleetzone-worker.service` | Celery worker for the `fleetzone` queue only; memory limit and `Restart=on-failure` |
| `fleetzone-backup.service` and `.timer` | Nightly database dump |

The API runs inside AA's existing web process in mode (a), so it has no unit of its own. A FleetZone deploy restarts AA's web process. Deploy outside fleet hours and announce it.

**Reverse proxy**

- The existing proxy serves the static bundle from disk at `/fleetzone/`, with long cache headers on hashed assets and none on `index.html`.
- `/fleetzone/api/` passes to AA's app server. Same origin, HTTPS only.
- Response headers: a content security policy with no inline scripts, and `frame-ancestors 'none'`.
- Request body limit on paste endpoints: 256 KB [D].

**Backups**

- Nightly dump of the FleetZone database, kept 14 days [D], stored outside the web root, readable only by the service user.
- A restore test in Phase 1 and again in Phase 6.
- Backups contain cached game data, so the kill-switch purge covers them.

**Logs**

- Logs go to journald or files rotated daily and kept 30 days [D].
- The audit log is in the database and is not rotated.

**Rate-limit headroom**

- FleetZone and Nexus share one ESI budget in mode (a). The headroom rule in section 7 and the dedicated worker are the controls.
- Discord's invalid-request ban budget is 10,000 per 10 minutes per IP [V], and Nexus shares that IP. The dispatcher never retries a 4xx.
- An ops page for directors shows the current budget, breaker state and FleetZone's call count per hour.

**Kill-switch runbook** (`ops/runbooks/kill-switch.md`; owner: the friend). The licence duty is to stop hosting, revoke ESI and purge cached game data within 5 business days of termination [V].

1. Disable all four modules in the director UI. Jobs stop.
2. `systemctl stop fleetzone-worker` and disable the unit.
3. Remove the `/fleetzone/` locations from the proxy and reload it. The app is no longer hosted.
4. Remove the app from AA's installed apps and restart AA.
5. ESI revocation: in mode (a) FleetZone has no CCP registration of its own, so stopping all use is FleetZone's part. Whether termination also requires action on Nexus's registration is a question for the Nexus admin [U].
6. Drop the FleetZone database. Delete all backups and the SDE copy on disk.
7. Write the date, the commands run and their output into `docs/gates/kill-switch-<date>.md`.

Steps 1 to 4 are drilled on the dev instance in Phase 1.

---

## 11. Phases with tasks

"Dev" in the Depends column means the AA dev instance.

### Phase 0: decide and spike (gated on the dev instance)

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 0.1 | Record the host facts | Dev | `docs/gates/phase-0.md` lists the AA version, django-esi version, DB engine, process manager, proxy, whether Celery runs, and whether `/o/.well-known/openid-configuration` answers [V for the path], each with command output |
| 0.2 | Decide the integration mode | 0.1 | A signed decision record exists in `docs/decisions/` |
| 0.3 | Spike: login, groups, one ESI call | 0.2 | A skeleton app is installed in dev AA. A log shows a user ID, its groups and one 200 skills read made with an AA-held token. A grep of the log finds no token. |
| 0.4 | Capture ESI responses with headers | 0.3 | One recorded response per needed read is in `fixtures/esi/` with tokens stripped. The gate file names the rate headers seen and the `Expires` value per read. |
| 0.5 | Work through the verification checklist | 0.3 | Every row below has a result and its evidence |
| 0.6 | Agree the group mapping | Dev | `fz_role_mapping` seed data is reviewed by a director. The meaning of `date_joined` on Nexus is recorded. |

**Verification checklist (task 0.5)**

| Item | Tag now | How it is settled |
|---|---|---|
| Scope names for skills, fleet, location, online, ship | [V via mirror] | Compare with CCP primary docs |
| Scope names and paths for notifications and corp structures | [U] | CCP primary docs, then one call |
| Fleet-member read with a non-boss token | [U] | Two characters in a test fleet; try each token |
| How the app finds a character's fleet ID | [U] | CCP primary docs, then one call |
| Endpoint to resolve names to character IDs | [U] | CCP primary docs, then one call |
| Corp role needed for structure reads | [U] | Read with the nominated character; read with one that lacks roles |
| Rate header names, bucket sizes, empty-bucket status | [U] | The 0.4 capture |
| What django-esi does on its own with caching and retries | [U] | Its docs, then a counted test against a fake server |
| Shared error budget across apps on one IP | [U] | Ask in CCP developer channels; assume yes until answered |
| zKill API fields and fair-use rules | [U] | Read `zkillboard.com/api/docs` |

### Phase 1: foundation

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 1.1 | Repository, tooling and CI | None | CI runs lint, types, both test suites and a frontend build on every push. A failing test blocks merge. |
| 1.2 | Core import boundary | 1.1 | `TestCoreImportsNoFramework` passes |
| 1.3 | Module registry and manifests | 1.1 | Four manifests load. Toggling a module flips its routes to 404 and writes an audit event. `TestDisabledModuleReturns404` passes. |
| 1.4 | Auth adapter and permissions | 0.3, 1.3 | `TestEveryRouteDeclaresPermission` and `TestRoleMatrix` pass. A user with no mapped group sees "no access". |
| 1.5 | Fetch policy in core | 1.1 | The Expires, ETag, scope, breaker, 420 and backoff tests pass against fakes |
| 1.6 | Fetch transport on AA | 0.4, 1.5 | The dev-instance integration test passes. A repeat read makes no second ESI call. |
| 1.7 | Audit log | 1.3 | `TestAuditTableRejectsUpdateAndDelete` passes as `fleetzone_app`. A role-mapping change produces an event with before and after values. |
| 1.8 | SDE importer | 1.1 | A pinned official build [V] imports into `fz_sde_*`. The skill requirements for five hand-checked items match the game. A second build can be imported and activated. |
| 1.9 | UI shell | 1.1 | Nav is built from `/shell/modules`. As-of, stale-badge and banner components exist. The footer carries the CCP notice verbatim [V] and a non-affiliation line. |
| 1.10 | Consent, opt-out and token states | 1.4, 1.6 | The first-run notice is shown once and logged. No token, lapsed and opted out render as three distinct states. |
| 1.11 | Operations | Dev | The worker unit, proxy config, backup timer and log filter are deployed on dev. A restore from backup works. The kill-switch drill, steps 1 to 4, is recorded. |

### Phase 2: Fleet ops

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 2.1 | EFT parser | 1.1 | Both EFT test classes pass on all fixtures |
| 2.2 | Fit skill requirements | 1.8, 2.1 | `TestSkillRequirementsMergeToHighestLevel` passes. A fit with an item missing from the SDE is saved `incomplete`. |
| 2.3 | Gap calculator | 2.2 | The gap and coverage test classes pass |
| 2.4 | Doctrine pages | 1.9, 2.1 | A director pastes, previews and saves. Unknown lines are highlighted. A member sees the list. |
| 2.5 | Readiness job and table | 1.6, 2.3 | The table shows real characters with as-of per row and a coverage line. A failed read keeps the old value with a stale badge. |
| 2.6 | Discord sender and ping | 1.7 | A ping reaches a test channel. `TestDeadWebhookDisablesChannel` passes. Text over 2000 characters [V] is truncated with a link. The hourly cap and the 10-minute fold work. |
| 2.7 | Fleet tracking | 0.5, 1.6 | A live test fleet produces an attendance sheet. Every row shows its provenance and snapshot counts. A refused read pauses tracking with a banner. |
| 2.8 | Attendance corrections | 2.7 | A correction keeps the original row, marks it, and writes an audit event. The words "no-show" appear nowhere. |

### Phase 3: Intel

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 3.1 | Local and chat parsers | 1.1 | Fixture pastes yield the expected names. An empty result shows "Nothing recognised" and submits nothing. |
| 3.2 | Name resolution | 0.5 | Names resolve to IDs through the fetch layer. A failed lookup labels the pilot Unknown. |
| 3.3 | Reports and decay | 1.4, 3.1 | `TestIntelDecayFromStoredDeadline` passes. The UI bar shrinks without a refresh. Recruits get 403. |
| 3.4 | Triage and zKill adapter | 3.2 | Lookups are throttled to 1 per second and cached for 1h. With zKill down, the report still posts. |
| 3.5 | Ego-centric map | 1.8, 3.3 | The map shows the systems within 5 jumps of the location read. It re-centres on change and can be pinned. |
| 3.6 | Reputation and broadcast check | 2.6, 3.3 | `TestBroadcastRequiresCrossCheck` passes. Unverified reports are labelled in the app. |
| 3.7 | Directory picker, off by default | 3.1 | With the setting on, a Chromium user picks the log folder; only derived sightings are uploaded. With it off, the endpoint is 404. |
| 3.8 | Ingestion gate measurement | 3.3 | Ten timed runs from paste to another user's screen are recorded. The gate passes if all are under 30s [D]. |

### Phase 4: Chain mapper

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 4.1 | Scanner paste parser | 1.1 | `TestScannerPasteSkipsMalformedRows` passes |
| 4.2 | Diff, preview and apply | 4.1 | The four signature-diff classes pass. A two-user session log shows zero silent deletes. |
| 4.3 | Connections and state machine | 1.8, 4.2 | The lifetime and mass classes pass. K162 stays "unresolved" until a type is set from the far side. Edit history is kept. |
| 4.4 | Map view | 4.3 | Systems and connections render with state and as-of. A concurrent edit shows the toast. |
| 4.5 | Routing | 1.8, 4.3 | `TestRouteExcludesRiskyEdgesByDefault` passes. "No route" offers the one-click retry. |
| 4.6 | Verify the wormhole facts | None | Lifetime bands and downtime ID re-issue are checked on CCP or EVE University sources and by a wormhole-experienced tester. The results are recorded. The downtime time of day is a setting. |

### Phase 5: Dashboard

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 5.1 | Character cards | 1.6 | Cards show location, ship and online with as-of. One failed read does not affect the other cards. Only the owner can load them. |
| 5.2 | Notification parsers | 0.5 | `TestNotificationParserFallsBackToRawText` passes, including the fuzz run. Raw YAML is not stored for known types. |
| 5.3 | Structures and timer board | 0.5, 1.6 | Timers for a test structure match the in-game values. A refused read shows the banner and keeps the last timers as stale. |
| 5.4 | Timer triggers | 2.6, 5.3 | Reinforcement and both fuel thresholds each fire once, include the as-of time, and never use `@everyone`. |

### Phase 6: alliance beta

| # | Goal | Depends | Acceptance |
|---|---|---|---|
| 6.1 | One week on the real budget | Phases 2 to 5 | Seven days with no 420 and no breaker open caused by FleetZone. The ops page export is attached. |
| 6.2 | Opsec review | 6.1 | The audit log and role tiers are reviewed by the owner and one director. `TestNoTokenInResponsesOrLogs` is run against production logs. |
| 6.3 | Recovery drills | 6.1 | A restore from backup and a full kill-switch drill on dev are recorded |
| 6.4 | Module sign-off | 6.1 | Each module's "v1 done" is signed |

---

## 12. First two weeks for the friend

Ordered. Tasks 1 to 12 need no AA instance.

1. Create the repository, the layout from section 3, and CI (1.1, 1.2).
2. Ask an alliance member for samples: 20 EFT fits, scanner pastes, local-list pastes, chat log files. Commit them as fixtures.
3. Write the EFT parser against the fixtures (2.1).
4. Download one official SDE build [V], pin its number, and write the importer to plain rows (the core half of 1.8).
5. Write the skill-requirement merge and the gap calculator (2.2, 2.3 core).
6. Write the scanner paste parser and the signature diff, with the convergence property test (4.1, 4.2 core).
7. Write the mass and lifetime state machine and the routing over the SDE gate graph (4.3, 4.5 core).
8. Write the local and chat-line parsers, and the intel decay and broadcast-check rules (3.1, 3.3, 3.6 core).
9. Write the fetch policy with a fake transport and clock: the Expires rule, ETag, scope allow-list, breaker, 420 and backoff (1.5).
10. Build the UI shell against a mock API: nav from manifests, as-of components, banners, footer notice (1.9).
11. Build the doctrine paste and preview page, and the signature preview page, on the mock API.
12. Write `ops/`: unit files, proxy snippet, backup script, kill-switch runbook draft.
13. **Needs the dev instance:** tasks 0.1 to 0.6.
14. **Needs the dev instance:** the fetch transport (1.6), the auth adapter (1.4), the audit grants test (1.7), the ops deploy (1.11).

If the dev instance is still missing after task 12, continue with the notification parser skeleton and the frontend pages for Intel and Chain on the mock API. Do not write AA glue from guesses.

---

## 13. Definition of done and review gates

**A task is done when:**

- Its acceptance statements are all true and each has a test or a recorded check.
- CI is green on the main branch.
- New external facts it relied on carry a tag, and any [U] it settled is updated in `docs/`.
- No new code outside `fetch/` calls ESI, and no new route lacks a permission.

**A phase is done when** its gate file `docs/gates/phase-N.md` exists with the evidence below and the sign-offs.

| Phase | Evidence attached | Signs off |
|---|---|---|
| 0 | Decision record; spike log; header capture; completed checklist | Friend; owner countersigns the mode decision |
| 1 | CI run link; the Expires and breaker test output; an audit entry for a role change; DB grant listing; restore log; kill-switch drill log | Friend; owner reads the runbook |
| 2 | Readiness table screenshot on real characters; attendance sheet from a live test fleet; dead-webhook test output | Friend, plus one FC who ran the fleet |
| 3 | Paste-to-triage recording; decay test output; the ten gate timings | Friend, plus one member who reported |
| 4 | Two-user session log; routing test output; the wormhole fact check | Friend, plus one wormhole-experienced tester |
| 5 | Timer board next to an in-game screenshot; fuzz test output | Friend, plus one director |
| 6 | Seven-day budget export; opsec review notes; drill logs | Owner and friend, per module |

The friend decides open questions and owns the technical gates. A second person countersigns every gate, so no gate passes on the builder's word alone.

---

## 14. Risks

| Risk | How the plan contains it |
|---|---|
| The dev instance never arrives | Section 12 lists two weeks of unblocked work. No AA glue is written before Phase 0. |
| FleetZone exhausts Nexus's ESI budget | One fetch layer, the half-budget breaker, a dedicated worker that can be stopped, and the 420 rule |
| django-esi retries or caches in ways the plan does not expect [U] | The 0.5 check with a counted fake server, before task 1.6 |
| Facts from mirrors are wrong (scopes, lifetime bands, downtime IDs) | The 0.5 checklist and task 4.6; conservative defaults until then |
| An AA upgrade breaks the app | Thin glue, no extra dependencies in AA's environment, and the dev instance as an upgrade canary |
| A FleetZone deploy takes Nexus down | Deploy windows, the dev instance first, and a one-line rollback (remove the app, restart) |
| A token or webhook URL leaks | Tokens never stored, the logging filter, write-only URLs, `TestNoTokenInResponsesOrLogs` |
| Intel poisoning or an over-trusted account | Reputation, cross-check before broadcast, a separate auditor role, the append-only audit log |
| Four modules is too much for one builder | One module per phase, the cut list in MECHANICS section 9, gates that stop the next phase |
| zKill or Discord terms or limits change | Each sits behind one adapter with a throttle and an off switch |
| Accidental licence breach | No paid features [V]; the footer notice; the drilled runbook |
| Fresh-code rule broken by accident | No other project's source is kept open while its equivalent is written; fixtures come from the game client only |

---

## 15. Open decisions at handoff

The first 20 are MECHANICS section 10, with their defaults. Each default applies unless the friend changes it.

| # | Decision | Default |
|---|---|---|
| 1 | Integration mode | (a), with the core kept framework-free |
| 2 | Is the OIDC plugin on Nexus? | Assume no; this matters only for (b) |
| 3 | When does the dev instance exist? | Until then, only the section 12 tasks 1 to 12 |
| 4 | Does a fleet-member read need the boss's token? [U] | Use the tracking FC's token; test in 0.5 |
| 5 | Group-to-role mapping | The table in MECHANICS section 1 |
| 6 | Readiness headline | Active level, with trained shown |
| 7 | Attendance source of truth | Fleet snapshot plus FC correction |
| 8 | Snapshot interval | 60s |
| 9 | Decay times | Hostile 10 minutes, clear 5 |
| 10 | Map radius | 5 jumps |
| 11 | Post-downtime signature IDs are new? [U] | Yes, until verified in 4.6 |
| 12 | Store raw notification YAML? | No; parsed fields, raw text for unknown types only |
| 13 | MVP notification types and timers | Structure reinforcement, fuel, unanchoring |
| 14 | Which character reads corp structures? [U] | A director nominates one; verify in 0.5 |
| 15 | Web Push in v1? | No |
| 16 | Directory picker | Off; a director can enable it |
| 17 | ESI headroom for Nexus | The breaker opens at half the remaining budget |
| 18 | Shared error budget by IP? | Assume yes |
| 19 | zKill lookup rate [U] | One request per second, cached 1h |
| 20 | Staged onboarding window | 30 days as Recruit |

Decisions this document adds, all [D]:

| # | Decision | Default |
|---|---|---|
| 21 | API style inside Django | Plain views with one decorator; no extra framework |
| 22 | Database placement | A separate database on Nexus's DB server, with two users |
| 23 | Job runner | AA's Celery on a dedicated queue and worker; systemd timers if Celery is absent |
| 24 | Frontend stack | React, TypeScript, Vite, Tailwind |
| 25 | Readiness refresh cadence | Hourly, plus on demand |
| 26 | Mount path | `/fleetzone/` |
| 27 | Backup and log retention | 14 days and 30 days |
| 28 | Who countersigns gates | The owner, plus the role named in section 13 |
