You are writing the PLAN for a planning effort. Output a single markdown plan document, roughly 1800-2500 words. Do not use tools. Plain markdown. Mark every claim you rely on with its evidence tag from the research below (V = verified from primary or at least named source, U = unverified, C = community/legal interpretation). Do not invent facts. If a fact is missing, list it as a blocker.

# Project
EVEMerc web rewrite: a fleet-first, modular web app for EVE Online alliances. Side project, theorycraft only. A friend (not the owner) builds it, so the plan must be builder-ready: clear modules, data model outline, phases with exit gates, and a short owner-decision list.

# Owner decisions already made
- Web first. Tauri later if ever. The old Laravel/Tauri EVEMerc is reference only.
- v1 modules, all four: (1) Intel: local/chat/gamelog-derived hostile sightings, threat triage, tri-state system status with decay, ego-centric map. (2) Fleet ops: doctrines, pings, attendance, member readiness. (3) Chain mapper: paste-diff signature sync, mass/lifetime, routing. (4) Character/corp dashboard: overviews, notifications, timers.
- Modular UI: modules enable or disable; fleet-first default-on set; clean interface; quick updates.
- Ideas-only: take ideas from other EVE projects and improve them. Write all code fresh. Do not copy code or assets.
- NEXUS = Alliance Auth (owner-confirmed). Nexus carries all ESI. EVEMerc must adapt to work with it. A dev instance is "later, on the same server" (not yet available).
- Hosting/backend choice: owner earlier chose "new backend, speed"; that conflicts with Alliance Auth plugin mode. This is an OPEN decision (see below).

# Findings: Alliance Auth integration (wave 2, agent A)
- AA extends via in-process Django apps with auth_hooks (url, menu, services, theme hooks). Plugin model: Python/Django, deploy on the Nexus host. (V, allianceauth readthedocs custom development)
- AA core has no documented public third-party REST API for groups or tokens. A third-party wrapper (basraah/allianceauth-api) defaults to admin-only. (U)
- AA as an OIDC provider only via third-party plugin allianceauth-oidc-provider (django-oauth-toolkit): /o/authorize/, /o/token/, /o/userinfo/, scopes openid profile email, groups claim. (V)
- AA stores users' ESI tokens in the django-esi Token model, in-process. No core endpoint hands them to external apps. (V)
- Fingerprints to confirm on the host: /sso/callback; /o/.well-known/openid-configuration (if OIDC plugin present). (V for paths, but confirm on the real host)
- Implications: option (a) plugin inside AA = Python/Django, needs host access; option (b) external app = needs an OIDC plugin on AA or a small custom API inside AA; option (c) = EVEMerc runs its own EVE SSO and token store (duplicates AA's token store, with a separate registration).

# Findings: ESI and SSO (wave 1)
- Non-commercial by default under CCP licence. Monetisation only via in-game ISK, voluntary donations that don't gate access, or general ads. No paywalled features. (V, CCP licence)
- Server-side web app: Authorization Code + client secret (HTTP Basic). PKCE is for desktop/mobile. Access tokens about 20 min; refresh tokens long-lived; token POST as x-www-form-urlencoded. Store secret and refresh server-side. (V)
- Refresh-token rotation enforcement for web apps: UNVERIFIED.
- Rate limits: legacy error budget ~100 non-2xx per min, then 420s; new token-bucket per route group (2xx=2, 3xx=1, 4xx=5, 5xx=0); X-Ratelimit-* headers. Honor Expires and ETag. Polling past Expires is a ban risk. (V)
- Fleet scopes: esi-fleets.read_fleet.v1 (members, wings; cached 5s); write scope needed for invite/move. (V)
- Location/online/ship scopes: esi-location.read_location.v1, read_online.v1, read_ship_type.v1. (V, third-party mirror; confirm on CCP docs)
- Skills: esi-skills.read_skills.v1 (cache ~120s), read_skillqueue.v1. (V, third-party mirror)
- Fittings: esi-fittings.read_fittings.v1 / write_fittings.v1; per-character snapshots, not corp doctrines. (V)
- Calendar RSVP: esi-calendar.read_calendar_events.v1 / respond. (V)
- Notifications: /characters/{id}/notifications/, cached 600s, text field is YAML with inconsistent field names. (V)
- Corp structures timers: state, state_timer_end, fuel_expires, unanchors_at, reinforce windows. (V)
- No ESI endpoint for fleet pings, chat or intel. Intel must come from logs (client-side) or paste. (V)
- Per-member readiness needs each member's own token and scope: alliance-wide readiness = per-member consent. (V, follows from scopes)
- Fleet-boss token question for member reads: UNVERIFIED.

# Findings: external data sources (wave 1)
- zKill: REST API (kills, losses, stats, killID lookup, history daily dumps). Docs at zkillboard.com/api/docs. Rate and quota: UNVERIFIED exact.
- RedisQ stream sunset 31 May 2026; replaced by R2Z2 ephemeral API (sequence.json, ~24h retention, ~20 req/s per IP per agent E). Build on R2Z2 with a cursor, not RedisQ. (V, github zKillboard/RedisQ; R2Z2 limits partly U)
- Eve-Scout: public, no-auth Thera/Turnur signature API (api.eve-scout.com/v2/public/...). Formal reuse terms: UNVERIFIED; courtesy polling, cache ~5 min, credit the source. Scout-reported data: never sole truth for WH.
- Official SDE (JSONL/YAML at developers.eveonline.com/static-data) is the canonical static source. Fuzzwork dumps are community conversions. Pin a build number per gap run. (V)
- Dotlan: no documented public API; scrape-only; permission UNVERIFIED. Avoid.
- Chain mappers (Pathfinder, Tripwire, Wanderer, wormhole.systems): each is self-hosted or per-group; no shared chain API exists. Mapper data is per-group and private.

# Findings: landscape (wave 1)
- Alliance Auth already covers auth, Discord/Mumble/TS auto-roles, fleet schedule, timers, SRP. Weakness: heavy self-host; fleet/doctrine readiness only via add-ons, fragmented.
- SeAT: corp/alliance ESI warehouse; audit/inventory first; no native ping, roster or doctrine flyability.
- Fleet-Up: dead since 2019. Gap: no hosted doctrine + ESI flyability + web roster in one login.
- EVE-HR: corp-only, alliance support not yet.
- NeuraEVE: paid Discord bot for pings/RSVP/attendance; Discord silo, no ESI readiness.
- EVEShip.fit: single-fit web tool, MIT. Not a doctrine manager.
- Gaps (wave 1): (1) hosted ping→roster→attendance→SRP loop; (2) alliance-wide doctrine readiness dashboard; (3) nullsec fleet intel overlay tied to fleets; (4) modular lite self-host plugin API.

# Findings: browser log ingestion (wave 2, agent B)
- Chat and game logs: Documents/EVE/logs/Chatlogs and Gamelogs (Windows, macOS, Linux/Proton under compatdata/8500). UTF-16LE with BOM. Lines: [YYYY.MM.DD HH:MM:SS] Sender > message, with a header block naming channel and Listener. Logging is in-game opt-in (default on). Files update on events, not as a streaming tail, so poll size/mtime. (V for locations and encoding, from blog and wiki; exact rotation and flush latency UNVERIFIED)
- File System Access API: Chromium desktop only (105+ full). Firefox and Safari: no directory picker. Android Chromium: no directory picker. Handles persist only with re-permission (Chrome 122+ offers persistent allow).
- Fallbacks: <input type=file>, drag-drop, clipboard paste (Ctrl+V or paste button; never silent reads). PySpy and dscan.info prove paste is enough for local triage.
- EF-Map (web) reads Frontier logs with picker, Web Worker, IndexedDB, no upload by default. Architecture precedent only.
- Smallest companion if the browser falls short: a single tray tail-and-forward process pointed at the logs folder, pushing only new parsed intel over localhost to the web app; no ESI secrets; no game-memory access.
- Policy: log parsing is community-tolerated but CCP has not published a permission for it: treat as C (community interpretation), optional, read-only, user-consented, off by default.

# Findings: chain mapper (wave 2, agent C)
- Probe-scanner paste: tab-separated rows (signature ID, group, type); reconcile with New / Updated / Missing preview. Never auto-delete on paste; operator confirms. (V, wormhole.systems and Tripwire docs)
- Downtime re-issues signature IDs: single-sourced (U). Treat post-downtime IDs as new records unless verified.
- Mass states from Show Info text: Fresh (>50%), Reduced (50-10%), Critical (<10%); independent of lifetime. (V, EVE University wiki)
- Lifetime states: >24h fresh; >4h beginning to decay; <4h EOL; expired. Per-type max lifetimes. (V, third-party guide, verify on CCP/EVE Uni)
- Visual cues (color, wobble) as manual aid only. Do not auto-derive state from visuals. (V)
- Type codes (O477, C247, K162) from the SDE; K162 resolved only after visiting the far side. (V)
- Static wormhole data: pin SDE build; community tables only as lookup aids. (V)
- Stale policy: auto-age toward EOL/critical; Expired manual-only; exclude EOL/critical from routing by default with override. Wanderer auto-deletes EOL edges 4h15m after flag. (V)
- Routing: union of chain edges and k-space gates; filters for avoid-system and min mass/lifetime. Thera-shortcut routing: follow-on. (U)
- Single-sourced claim on static respawn after collapse: U.

# Findings: fleet readiness and notifications (wave 2, agent D)
- EFT is the clipboard format: [hull, fit name] then blank-line-separated sections, Drone xN. Parse tolerant. (V)
- ESI fittings: per-character snapshots with slot flags (HiSlot0-7, MedSlot, LoSlot, Rig, Subsystem, DroneBay, Cargo...). Cached up to 300s. Names 1-50 chars. (V)
- Skill requirements: SDE dogmaAttributes (182/183/184/1285/1289/1290 skill type IDs with 277/278/279/1286/1287/1288 levels). Up to 6 pairs per item. (V)
- Gap calc: for each required skill, max(0, required − active_skill_level). Use active (Alpha clone caps, Expert Systems), show trained-vs-active split. Missing = absent. Join skillqueue for ETA (skillqueue schema U from a third-party mirror). (V/U)
- Attendance sources: fleet membership snapshot (5s cache, fleet-member token); calendar RSVP (accepted/declined/tentative/not_responded; event owner's token). Failure modes: lapsed tokens, departures, aged-out events make absences ambiguous (no-show vs no-data). Store provenance per row; manual FC correction. (V/U)
- Discord webhook: content 2000 chars, 10 embeds, 6000 combined embed chars; no fixed per-webhook rate number; read X-RateLimit headers; 404 = dead webhook, stop. Invalid-request ban budget 10000 per 10 min per IP. (V)
- Web Push: HTTPS, service worker, VAPID, userVisibleOnly; payload ceiling 4096 bytes, practical ~3.5 KB; subscription endpoint is a secret. (V)
- FCM quotas only if Firebase is chosen (U for exact numbers from one source).
- ESI latency: cache-driven, not push. Notifications 600s, fittings 300s, skills 120s, fleet members/wings 5s. Build data as-of timestamps. (V)

# Findings: policy and opsec (wave 2, agent E)
- CCP licence: non-commercial; monetisation limited as above. Verbatim notice: "© 2014 CCP hf. All rights reserved. EVE, EVE Online, CCP, and all related logos and images are trademarks or registered trademarks of CCP hf." Non-affiliation line required. Kill-switch: on termination stop public hosting, revoke ESI, purge cached Game Data within 5 business days. (V)
- Cache scraping and client interaction banned; use ESI and user-supplied logs only. (V)
- zKillboard: AGPLv3 code; hosted terms: responsible API use, no scraping around the API, best-effort uptime. (V)
- GPL-3 line: copyright protects expression, not ideas. Reimplementing learned behaviour fresh is not a derivative. Edge cases by jurisdiction: C, get counsel if it matters. (V on FSF text; C on the line itself)
- Opsec failure modes: deep-cover moles earning roles; shared credentials; over-centralised auditors; intel poisoning (fake scouting, doctored killmails). Mitigations: least-privilege tiered scopes, per-user SSO only, append-only audit log, staged onboarding, reporter reputation and ESI/zKill cross-check before broadcast. (U for the example source; the mitigations are standard practice, treat as design recommendations)

# Known weak spots in this research
- Several VERIFIED tags come from third-party mirrors or blogs (skoli.ru, a GitHub skill repo, odealo, an EVE Uni wiki). Treat endpoint lists as leads until checked against CCP primary docs.
- Agent C's vendor comparison page (eve-nexum.com) is single-sourced and possibly biased.
- Downtime ID rule, static respawn, fleet-boss token requirement: single-sourced or UNVERIFIED.
- Alliance Auth OIDC plugin: VERIFIED from its repo, but whether the owner's Nexus has it installed is UNKNOWN.

# What the plan must cover
1. Integration mode decision (a / b / c) with its branch consequences, gated on: owner confirms AA host access, OIDC plugin presence, and a dev instance. Do not pick a backend until this is answered; give the builder both branches with what changes in each (language, deploy, token storage, rate sharing).
2. Module breakdown, v1 scope per module, what is cut from each, with exit criteria.
3. Data model outline (entities: alliance, corp, member/character, token-scope grant, doctrine, fit, fleet, ping, attendance row with provenance, intel report with reporter and status, chain map, signature, connection, notification, timer). Name only; no SQL.
4. ESI client rules: central fetch layer, Expires/ETag, error budget, per-member consent flow.
5. Intel ingestion: paste-first baseline, Chromium picker as enhancement, companion only if the gate fails. State the gate.
6. Notification design: Discord webhook first, Web Push second; limits; dead-webhook handling.
7. Opsec and policy requirements as MVP, not later hardening.
8. Phases with exit gates. Each gate names its evidence. Phase 0 must include the integration decision and a test against a dev instance.
9. Risks, ranked, with mitigations.
10. OWNER DECISIONS (list, short): integration mode; hosting and who runs it; trust model (who sees intel, coalition or alliance-only); module order and v1 done for each; Discord-only or Web Push too; attendance source of truth; readiness on active vs trained level; timer set for MVP; raw notification storage; whether a companion app is acceptable in v1.
11. BLOCKERS: anything the builder cannot start without.
12. Verification plan: what gets checked by whom, before each phase gate.

Keep it sharp. No filler. Lead with the integration decision and the blockers, since they gate everything else.
