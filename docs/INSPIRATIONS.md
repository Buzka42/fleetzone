# Inspirations

FleetZone takes **ideas** from these projects. Nothing is copied. Each idea is reimplemented from scratch, and we aim to make it better. The table lists what we took and what we changed.

**Licences are listed for information.** The rule holds whatever the licence: no code, no assets, no copied text.

| Project | Licence | Stack (for context) | Idea we take | How FleetZone improves it |
|---|---|---|---|---|
| [eve-wrench](https://github.com/eve-wrench/eve-wrench-app) | MIT | Rust, GPUI | Selective copy with a diff preview, and backup before every change | Versioned history for every fleet config change, with per-item restore. Applied to doctrines and roles, not game files |
| [evebuddy](https://github.com/ErikKalkoken/evebuddy) | MIT | Go, Fyne | Character and corp overviews, a notification engine, a training-queue view | Per-member notification rules, quiet hours, a test button, and a fleet-wide readiness view instead of one character at a time |
| [eve-pi](https://github.com/calli-eve/eve-pi) | MIT for non-CCP code | Next.js, Three.js | Extractor-expiry countdowns, click-to-group characters | One timer board for the whole fleet, with stall alerts; the squad view is the default layout |
| [Triff.Tools](https://github.com/NarcisussX/Triff.Tools-Source) | GPL-3.0-or-later | React, Express, Pyfa sidecar | Module registry with saved per-user state; fit-to-skill requirement handoff | Manifest-driven modules with a fleet-first default set; readiness per member, shown as a column |
| [PySpy](https://github.com/Eve-PySpy/PySpy) | MIT | Python, wxPython (last release 2019) | Paste local, get a threat table in seconds; threat-triage columns | Shared intel table with reporter attribution; zKill lookups cached once for the whole alliance, not per user |
| [EVELiveIntel](https://github.com/jeremieroy/EVELiveIntel) | BSD-2-Clause | PHP, D3 (2014) | Tri-state system status with time decay; ego-centric map | Each status carries source, reporter and timestamp. Decay is visible on screen |
| [wormholesystems](https://github.com/wormholesystems/wormholesystems) | MIT | Rust, SvelteKit | Paste-diff signature sync; mass and lifetime per connection; ID-only WebSocket events with REST refetch | Adds a sequence number so clients detect missed events; accepts chat-log lines as well as scanner paste |
| [EVEMerc (desktop)](https://github.com/Buzka42/evemerc) | Owner's project | Tauri, Svelte | Fleet workspace, modular intel panels, the EVE log folder reader | Web rewrite. The desktop app stays as reference only |

## Wormhole mappers (for behaviour, not code)

| Tool | Where | Relevant behaviour |
|---|---|---|
| [Wanderer](https://github.com/wanderer-industries/wanderer) | Community Edition self-hostable (MIT per the research) | EOL auto-delete after a set time; connection health shown by colour and shape |
| Tripwire | Self-host only; the public instance closed mid-2024 | Fast paste import; delete-missing prompt with undo |
| Pathfinder | Self-host; reported as no longer actively developed | Heaviest feature set; the reference for wormhole type data |
| Nexum ([eve-nexum.com](https://eve-nexum.com/compare/)) | Open source | Comparison page; single source, treat it as a lead |

## Alliance tools (what they cover, so we don't repeat them)

| Tool | What it covers | Gap FleetZone can fill |
|---|---|---|
| [Alliance Auth](https://gitlab.com/allianceauth/allianceauth) (GPL) | This is Nexus. Auth, Discord and TeamSpeak roles, fleet schedule, timers, SRP | Readiness and doctrine views are add-ons, and fragmented |
| [SeAT](https://github.com/eveseat/seat) (GPL) | Corp and alliance ESI warehouse: wallets, assets, members | No ping, roster or doctrine flyability flow |
| EVE-HR | Corp management, fittings checker, SRP | Corp-only; alliance support not yet available |
| Fleet-Up | Doctrine and fleet ops with flyability check | Shut down in 2019; the gap it left is still open |
| EVEShip.fit (MIT) | Fit viewer with shareable links | Single fit only; no doctrines or rosters |
| NeuraEVE | Paid Discord bot: pings, RSVP, attendance | Lives in Discord; no ESI readiness or web roster |

## Reference for architecture, not behaviour

- [EF-Map log parser](https://ef-map.com/log-parser): the browser folder-picker pattern (Web Worker parse, IndexedDB, no upload by default). Read for shape only.
- [dscan.info](https://dscan.info/howto): paste-based triage in the browser.

## What each source is NOT for

- Do not copy Triff.Tools code. It is GPL-3.0-or-later. Its ideas are fine.
- Do not reuse PySpy's proprietary intel database. It relied on an external service that is probably no longer maintained.
- Do not reuse EVELiveIntel's runtime. Its IGB trust flow is gone, and its static map data predates the wormhole and Pochven updates.
- Do not hard-depend on any wormhole mapper's API. Only self-hosted or per-group instances exist, and none is a shared chain API.
