# EVEMerc Web Rewrite: Build Plan

**Status:** blocked at Phase 0 on three owner answers (see section 1).
**Tags:** V = verified from a named source, U = unverified, C = community or legal interpretation. "Design" marks my recommendation, not a research finding. Several V tags rest on third-party mirrors, so they stay leads until checked against CCP primary docs (section 12).

---

## 1. Blockers

The builder cannot start anything backend-shaped without these.

| # | Blocker | Who clears it | Blocks |
|---|---|---|---|
| B1 | Does the builder get shell/deploy access to the Nexus (Alliance Auth) host? | Owner | Integration mode, language, deploy |
| B2 | Is `allianceauth-oidc-provider` installed on Nexus? Presence is unknown; check `/o/.well-known/openid-configuration` and `/sso/callback` on the real host (V for paths). | Owner or Nexus admin | Mode (b) |
| B3 | An AA dev instance. Currently "later, on the same server". | Owner | Phase 0 exit; no integration code can be proven without it |
| B4 | Integration mode (a/b/c), which follows from B1–B3. | Owner | Backend choice |
| B5 | Which ESI scopes Nexus already requests from members, and whether the admin will add more. Missing fact. | Nexus admin | Readiness, fleet, dashboard scope |
| B6 | Trust model: who sees intel (alliance-only or coalition). | Owner | Data model tenancy, permissions |
| B7 | Who hosts and operates EVEMerc. This person also carries the CCP kill-switch duty (V). | Owner | Deploy, on-call, licence compliance |

Until B1–B4 clear, the builder can still do the backend-neutral work: the UI shell, the EFT, scanner-paste and chat-log parsers, the SDE import, and the readiness gap calculator as pure libraries with tests.

---

## 2. Integration mode decision

The earlier owner choice "new backend, speed" conflicts with "Nexus carries all ESI". The facts that constrain the choice:

- AA extends through in-process Django apps with `auth_hooks` (V).
- AA keeps ESI tokens in the django-esi Token model, in-process, and no core endpoint hands them out (V).
- AA core has no documented public REST API for groups or tokens (U).
- OIDC is available only through a third-party plugin, with a groups claim (V).

| | (a) Plugin inside AA | (b) External app + AA bridge | (c) Own EVE SSO |
|---|---|---|---|
| Language | Python/Django | Free choice, plus a small Django app inside AA | Free choice |
| Deploy | On the Nexus host, in AA's release cycle | Separate service, plus a bridge app on Nexus | Fully separate |
| Login | AA session | OIDC plugin (`/o/authorize/`, `/o/token/`, `/o/userinfo/`, groups claim) (V) | EVE SSO Authorization Code + client secret (V) |
| Token storage | django-esi Token, reused in-process (V) | Tokens stay in AA; the bridge serves ESI data, never tokens (design) | Second token store and separate CCP app registration; members consent twice |
| ESI rate sharing | One client and one budget with AA; EVEMerc can burn Nexus's error budget | Same budget, behind the bridge | Separate registration. Whether budgets are shared when co-hosted on one IP is a missing fact: check it. |
| Needs | B1, B3 | B1 (to install the bridge), B2, B3 | Nothing from Nexus |
| Honors "Nexus carries all ESI" | Yes | Yes | No; the owner must reverse that decision |
| Main cost | Tied to the AA stack and upgrades | Two codebases and a bridge API to design and secure | Duplicate consent, a second secret store, more opsec surface |

**Recommendation (design):** choose (a), built as a Django app that exposes a JSON API to a separately built front-end bundle. This reuses the token store and group model. The UI stays fast and portable, and the core logic stays in framework-free packages, so a later move to (b) is a transport change rather than a rewrite. Fall back to (b) if the owner allows one small bridge plugin but not a full app on Nexus. Use (c) only if Nexus access is refused.

Do not pick a framework, database or host before B4 is answered.

---

## 3. Modules and v1 scope

The shell holds a module registry with per-alliance enable/disable. The default-on set is Fleet ops and Intel; Chain mapper and Dashboard are opt-in until the owner says otherwise.

### 3.1 Fleet ops
- **In scope:**
  - Doctrines as sets of EFT fits; EFT parsing is tolerant (V).
  - Per-member readiness from SDE skill requirements: dogma attributes 182/183/184/1285/1289/1290 with levels 277/278/279/1286/1287/1288 (V). Gap = max(0, required − active level), with a trained-vs-active split shown (V).
  - Pings to Discord.
  - Attendance from fleet membership snapshots (`esi-fleets.read_fleet.v1`, 5s cache) (V), with provenance per row and FC correction.
- **Cut:** SRP (AA has it), fleet invite/move (write scope), pushing fits to the client, calendar RSVP as a primary source, and skillqueue ETA (schema U; add after verification).
- **Exit:** a doctrine pasted in EFT yields a readiness table for consenting members with as-of timestamps. A test fleet produces an attendance sheet whose rows each show source and time.

### 3.2 Intel
- **In scope:**
  - Paste-first sightings (local list, chat lines).
  - Threat triage through zKill lookups.
  - Tri-state system status (clear / unknown / hostile) with timed decay.
  - Ego-centric map around the user's system.
  - Reporter and status on every report.
- **Cut:** fully automatic log tailing (enhancement, see section 6), Dotlan data (no API, permission U), and coalition federation unless B6 says so. The killmail stream is also cut from v1; if added later, build it on R2Z2 with a cursor, since RedisQ's sunset date of 31 May 2026 (V) has already passed.
- **Exit:** a pasted local list becomes triaged sightings in one action. A status visibly decays without input. Every report shows reporter, source and age.

### 3.3 Chain mapper
- **In scope:**
  - Probe-scanner paste with a New / Updated / Missing preview, and no auto-delete (V).
  - Manual mass state (Fresh / Reduced / Critical) and lifetime state (V; lifetime bands from a third-party guide, verify).
  - Type codes from the pinned SDE; K162 resolved only after the far side is visited (V).
  - Auto-ageing toward EOL, with Expired set manually only.
  - Routing over chain edges plus k-space gates, with EOL/critical excluded by default and an override (V for the stale policy, U for routing details).
- **Cut:** Thera/Turnur shortcut routing (follow-on; Eve-Scout reuse terms U), state derived from visuals (V: manual aid only), and import from other mappers (no shared API exists).
- **Exit:** two users paste the same system and converge with no silent deletions. A route honors the mass and lifetime filters. Post-downtime signature IDs are treated as new records (rule is U; this is the safe default).

### 3.4 Character/corp dashboard
- **In scope:**
  - Character overview (location, ship, online, skills) for consented scopes.
  - Notifications feed (600s cache; text is YAML with inconsistent fields, so parse defensively) (V).
  - Structure timers from corp structures (state, `state_timer_end`, `fuel_expires`, `unanchors_at`) (V).
- **Cut:** wallet, assets and industry (SeAT territory), and any notification types beyond the owner's MVP list.
- **Exit:** the timer board matches in-game values for a test structure. An unknown notification type renders as raw text rather than crashing.

---

## 4. Data model outline

Names only. Every row carries the alliance tenant and an as-of timestamp where it came from ESI.

- **Alliance**, **Corp**, **Member** (the AA user), **Character** (many per member).
- **TokenScopeGrant**: character, scope, granted/revoked times, last successful use. In modes (a) and (b) it mirrors AA's token state; in (c) it owns it.
- **Doctrine**, **Fit** (hull, slot-flagged modules, source EFT text), **SkillRequirement** (derived, tagged with SDE build).
- **ReadinessSnapshot**: character, fit, gaps, active vs trained level, as-of.
- **Fleet**, **Ping** (target, channel, delivery result), **AttendanceRow** (provenance: fleet snapshot / RSVP / manual; corrected-by; confidence).
- **IntelReport** (reporter, source: paste / picker / companion; status; cross-check result), **SystemStatus** (tri-state, decay deadline).
- **ChainMap**, **Signature** (ID, group, type, first/last seen, downtime epoch), **Connection** (type code, mass state, lifetime state, flagged-at).
- **Notification** (parsed fields, plus raw text if the owner approves storage), **Timer** (kind, source, end time).
- **NotificationChannel** (Discord webhook or Web Push subscription; both are secrets) (V for the push endpoint).
- **AuditEvent**: append-only.
- **ModuleSetting**, **SdeBuild**.

---

## 5. ESI client rules

1. **One fetch layer.** No module calls ESI directly. In (a) and (b) the layer wraps AA's client; in (c) it is EVEMerc's own.
2. **Cache discipline.** Honor `Expires` and `ETag`; never poll before `Expires`, which is a ban risk (V). Store the as-of time with every result and show it in the UI.
3. **Rate budget.**
   - Track the legacy error budget (about 100 non-2xx per minute, then 420s) and the per-route-group token buckets (2xx=2, 3xx=1, 4xx=5, 5xx=0) from the `X-Ratelimit-*` headers (V).
   - Add a circuit breaker that halts a route group before the budget empties.
   - In (a) and (b), reserve headroom for AA's own jobs.
4. **Known cache floors.** Fleet members 5s, skills about 120s, fittings 300s, notifications 600s (V). Schedules derive from these.
5. **Per-member consent.**
   - Alliance-wide readiness requires each member's own token and scope (V).
   - Request scopes per module, at the moment a member enables it.
   - Show what each scope unlocks, and render "no consent" and "token lapsed" as distinct states, never as "not ready".
   - Scopes: `esi-skills.read_skills.v1`, `esi-fleets.read_fleet.v1`, `esi-location.read_location.v1`, `read_online.v1`, `read_ship_type.v1` (V via mirror; confirm).
6. **Token handling (mode c only).** Authorization Code with client secret, with the secret and refresh tokens server-side and encrypted (V). Assume refresh tokens may rotate and persist any new one on every refresh, since rotation enforcement is U.
7. **Open question.** Whether a fleet-boss token is required to read fleet members is U. Test it in Phase 0 before the attendance design freezes.

---

## 6. Intel ingestion

- **Baseline: paste.** Clipboard paste on an explicit user action, never a silent read. PySpy and dscan.info show that paste is enough for local triage. It works in every browser.
- **Enhancement: Chromium directory picker.**
  - The File System Access API is Chromium desktop only; Firefox, Safari and Android have no directory picker.
  - Read `Chatlogs` and `Gamelogs` as UTF-16LE with BOM, polling size and mtime because the files are not a streaming tail (V; flush latency U).
  - Parse in a Web Worker and upload only derived sightings, never raw logs.
  - Off by default, read-only and user-consented, because log parsing is community-tolerated with no published CCP permission (C).
- **Companion: only if the gate fails.** A tray tail-and-forward process posting parsed intel over localhost, with no ESI secrets and no game-memory access (cache scraping and client interaction are banned, V).

**Gate (measured at the end of the Intel phase):**
- Pass: paste plus picker delivers a sighting to other users within a latency the owner sets (proposed 30s), for the owner's target share of intel reporters. No companion is needed.
- Fail: measured log flush latency breaks the target, or too many reporters are on non-Chromium browsers. The companion then becomes a v1.1 item, subject to the owner decision in section 11.

---

## 7. Notifications

**Discord webhook first.**
- Limits: content 2000 characters, 10 embeds, 6000 combined embed characters (V). Truncate with a link back to the app.
- No fixed per-webhook rate exists, so read the `X-RateLimit` headers and queue (V).
- A 404 means the webhook is dead: disable the channel, stop sending and alert an admin (V).
- Never retry invalid requests in a loop; the ban budget is 10000 invalid requests per 10 minutes per IP (V).

**Web Push second, if the owner wants it.**
- Requires HTTPS, a service worker, VAPID and `userVisibleOnly` (V).
- Keep the payload under about 3.5 KB (ceiling 4096 bytes) (V): send an ID and summary, then fetch the detail.
- Use no FCM-specific features unless Firebase is chosen (its quotas are U).

All notifications carry the data as-of time, since ESI is cache-driven and not push (V).

---

## 8. Opsec and policy

These are MVP requirements. They ship in Phase 1 and gate every later phase.

- **Licence.**
  - Non-commercial; no paywalled features (V).
  - Footer carries the verbatim notice "© 2014 CCP hf. All rights reserved. EVE, EVE Online, CCP, and all related logos and images are trademarks or registered trademarks of CCP hf." plus a non-affiliation line (V).
  - A written kill-switch runbook: stop hosting, revoke ESI and purge cached Game Data within 5 business days (V). It has a named owner (B7).
- **Fresh code only.** Ideas come from other projects, with no copied code or assets. Reimplementing learned behaviour is not a derivative work as a general line (C); get counsel if it ever matters. The builder does not keep GPL/AGPL sources open while writing the equivalent module (design).
- **Access.** Per-user SSO only, no shared credentials. Tiered least-privilege roles mapped from AA groups, with staged onboarding so new members see less (design; the source example is U).
- **Audit.** An append-only AuditEvent for role changes, intel edits, attendance corrections, signature deletes and exports.
- **Intel poisoning.** Reporter reputation, plus an ESI/zKill cross-check before any broadcast. Unverified reports are labelled as such.
- **External sources.**
  - zKill only through its API, used responsibly and without scraping (V); exact quotas are U, so throttle conservatively.
  - Eve-Scout data is cached about 5 minutes, credited, and never the sole truth (terms U).
  - The SDE build is pinned.

---

## 9. Phases and exit gates

| Phase | Work | Exit gate and its evidence |
|---|---|---|
| **0. Decide and spike** | Clear B1–B7. Stand up the dev AA instance. Spike the chosen mode: log in, read groups, make one authenticated ESI call through Nexus. Test the fleet-boss question. Verify scopes on CCP docs. | A signed decision record for the mode. A screen recording or log of login → group → ESI call on the dev instance. A verification checklist with a result on each item. |
| **1. Foundation** | Module shell and registry, permissions, ESI fetch layer, SDE import, audit log, licence footer, kill-switch runbook, CI. | A test showing no request before `Expires`. A circuit-breaker test. Audit entries for a role change. A runbook reviewed by the owner. |
| **2. Fleet ops** | Section 3.1. | A doctrine → readiness demo on real consenting characters. An attendance sheet from a live test fleet with provenance. A dead-webhook test. |
| **3. Intel** | Section 3.2, plus the picker enhancement. | A paste → triage demo. A decay test. The ingestion gate measurement from section 6. |
| **4. Chain mapper** | Section 3.3. | A two-user paste-diff session log with zero silent deletes. Routing filter tests. |
| **5. Dashboard** | Section 3.4. | Timers matching in-game values. A fuzz test on notification YAML. |
| **6. Alliance beta** | Real users, load on the rate budget, opsec review. | One week with no rate-limit trips. An audit log review. Owner sign-off on each module's "v1 done". |

Phases 2–5 follow the fleet-first default; the owner may reorder them.

---

## 10. Risks, ranked

1. **Integration undecided or the dev instance never arrives.** Mitigation: Phase 0 is a hard gate; only backend-neutral libraries are built before it.
2. **EVEMerc exhausts Nexus's shared ESI budget.** Mitigation: the central layer, reserved headroom and the circuit breaker.
3. **Members do not consent to scopes, leaving readiness sparse.** Mitigation: per-module consent with a clear payoff, and a consent-coverage figure shown next to every readiness number.
4. **Research resting on mirrors is wrong** (scopes, lifetime bands, downtime IDs). Mitigation: the verification plan in section 12, and conservative defaults.
5. **Infiltration or intel poisoning.** Mitigation: section 8 from day one.
6. **Four modules is a lot for one side-project builder.** Mitigation: strict cut lists, one module per phase, and the owner defines "done" for each.
7. **The log path is slow or unsupported in the browser.** Mitigation: the paste baseline and the gate.
8. **AA upgrades break a plugin** (mode a). Mitigation: a thin Django layer, portable core logic, and the dev instance used as an upgrade canary.
9. **Third-party terms change** (zKill, Eve-Scout). Mitigation: each source sits behind an adapter and can be switched off.
10. **A licence breach by accident** (a donation perk gating a feature). Mitigation: the monetisation rule sits in the contributor notes.

---

## 11. Owner decisions

1. **Integration mode:** a, b or c. Recommended: a.
2. **Hosting and operator:** who runs it and holds the kill-switch duty.
3. **Trust model:** intel is alliance-only or coalition; who sees chain maps.
4. **Module order and "v1 done" per module:** confirm or edit sections 3 and 9.
5. **Notifications:** Discord only, or Web Push too. Recommended: Discord only for v1.
6. **Attendance source of truth:** fleet snapshot, RSVP, or FC-corrected sheet. Recommended: snapshot plus FC correction.
7. **Readiness basis:** active or trained level as the headline. Recommended: active, with trained shown.
8. **Timer set for MVP:** which structure and notification timers.
9. **Raw notification storage:** keep the raw YAML or parsed fields only.
10. **Companion app:** acceptable in v1, later, or never.

---

## 12. Verification plan

| Check | Who | Before gate |
|---|---|---|
| OIDC plugin presence and fingerprint paths on real Nexus | Nexus admin | 0 |
| Scopes Nexus already holds; willingness to add more | Nexus admin | 0 |
| All scope names and cache times against CCP primary docs | Builder | 0 |
| Fleet-boss token requirement; refresh rotation behaviour | Builder, on the dev instance | 0 |
| Whether co-hosted apps share an ESI error budget | Builder | 0 |
| Licence notice wording and monetisation rule | Owner | 1 |
| zKill API quotas; R2Z2 limits | Builder | 3 |
| Chat-log flush latency and rotation on a real client | Builder plus one tester | 3 |
| Eve-Scout reuse terms (ask them) | Owner | Before Thera routing |
| Lifetime bands, downtime ID re-issue, static respawn on CCP or EVE Uni | Builder plus a wormhole-experienced tester | 4 |
| Skillqueue schema | Builder | Before the ETA feature |
| Notification YAML field variants from real samples | Builder | 5 |
| Audit log and role-tier review | Owner plus one trusted director | 6 |

Each gate in section 9 needs its evidence artifact attached and the owner's sign-off. The builder's word alone does not pass a gate.
