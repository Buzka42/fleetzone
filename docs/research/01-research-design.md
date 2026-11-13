# EVEMerc rewrite: research design (wave 2)

## 1. Blocking unknowns, ranked

1. **Nexus integration mode.** Three options with very different plans: (a) EVEMerc runs *inside* Nexus as a plugin, (b) it is an external app calling a Nexus API or ESI proxy, or (c) Nexus gives identity and groups only, and EVEMerc needs its own ESI tokens. Option (a) dictates the backend language, and option (c) contradicts "Nexus carries all ESI".
   - **From the owner or Nexus admin:** what the software is, who runs it, whether third-party apps can be registered, which scopes it holds, and whether the builder gets a dev instance.
   - **From its repo or docs:** endpoints, token model, group and role model, webhooks.
   - **From a probe (only with admin consent):** fingerprint the login page.
2. **Log ingestion in a browser.** Intel depends on local chat and game log files, and a web app cannot tail files freely. This may force a companion app into v1 or reduce intel to paste and ESI only, which conflicts with "Tauri later if ever".
3. **Whose tokens power the fleet features.** Fleet composition needs the fleet boss's token, and readiness needs every member's skills. Both depend on unknown 1.
4. **Builder's stack and hosting envelope.** This is an owner input, not research.
5. **Chain-mapper domain rules.** Paste format, diff semantics, mass and lifetime states.
6. **Policy and licence boundaries.** CCP developer licence, log-reading rules, and where "ideas only" ends against GPL sources.

## 2. Second wave: five agents

Shared output schema for every row: `claim | source URL | date accessed | confidence (high/med/low) | VERIFIED or UNVERIFIED | planning implication`. Each agent ends with at most five open questions.

**A. Auth-platform integration models**
- **Goal:** map how third-party apps integrate with each plausible Nexus candidate.
- **Questions:**
  - For Alliance Auth, SeAT, Neucore and any public EVE project named "Nexus": is there a REST API, an OAuth/OIDC provider or an ESI proxy?
  - Are tokens reachable only in-process?
  - How are groups and roles exposed?
  - What is the plugin model and its language?
  - What fingerprints identify each one (URL paths, cookies, static asset paths)?
- **Sources:** official docs, repos, plugin directories, developer wikis.
- **Unblocks:** the choice between modes (a), (b) and (c), and the owner's question list for the Nexus admin.

**B. Browser-side log ingestion**
- **Goal:** establish what a pure web app can read.
- **Questions:**
  - What are the EVE chat and game log locations, encoding, flush behaviour and line formats?
  - What does the File System Access API support per browser, and how do permissions persist?
  - Does clipboard paste (PySpy-style) suffice for triage of local?
  - How do existing web-based intel tools ingest logs?
  - What is the smallest companion design if the browser falls short?
- **Sources:** MDN, caniuse, CCP support articles, intel-tool docs and repos (ideas only).
- **Unblocks:** whether intel v1 is web-only or needs a companion; which intel features are cut.

**C. Chain-mapper domain model**
- **Goal:** a spec-grade rule set.
- **Questions:**
  - What is the probe-scanner paste format, and how do signature IDs behave across downtime?
  - What are the diff rules for add, update and stale?
  - What are the wormhole mass and lifetime states and their visible cues?
  - Where does static wormhole-type data come from?
  - How should routing combine chain and k-space, and how do users treat stale connections?
  - What is the feature baseline across Pathfinder, Tripwire, Wanderer and wormhole.systems?
- **Sources:** EVE University wiki, CCP patch notes, mapper docs.
- **Unblocks:** mapper data model and v1 feature cut.

**D. Fleet readiness and notification mechanics**
- **Goal:** data mechanics, not the tool landscape (wave 1 covers that).
- **Questions:**
  - What are the doctrine fit formats (EFT, ESI fittings)?
  - How are skill requirements derived from SDE dogma attributes?
  - What does a gap calculation need?
  - How is attendance captured, and what are its failure modes?
  - What are the ping delivery channels (Discord webhooks, Web Push) and their limits?
  - What are the notification and timer types, and what latency does caching impose?
- **Sources:** ESI and SDE docs, Discord and Web Push docs, the wave-1 ESI output.
- **Unblocks:** fleet and dashboard schemas; whether notifications can be near-real-time.

**E. Policy, licensing and opsec**
- **Goal:** set the hard boundaries.
- **Questions:**
  - What do the CCP developer licence terms say on monetisation, attribution and data retention?
  - What is CCP's stance on log reading versus client and cache access?
  - What are the zKill and Eve-Scout usage terms?
  - Where is the practical GPL-3 line between idea and expression?
  - What are the known spy and intel-poisoning patterns in alliance tools, and their mitigations (audit logs, tiered access, reporter reputation)?
- **Sources:** CCP legal pages, third-party policies, licence texts, published incident write-ups.
- **Unblocks:** permission model, audit requirements, feature vetoes.

## 3. Owner decisions (not research)

- Which stack the builder is fluent in, and whether a forced language (Nexus plugin mode) is acceptable.
- Hosting budget and who operates it.
- Scale: one alliance or multi-tenant.
- Whether a companion app is acceptable in v1 if agent B says web-only intel is crippled.
- The order of the four modules, and what "v1 done" means for each.
- The trust model: who sees intel and the chain map, and whether it extends to coalition allies or stays alliance-only.
- Whether Discord is assumed as the ping channel.
- Open source or private.
- Whether the Nexus admin will grant a dev instance and app registration.

## 4. Verification gates

- **Nexus:** nothing is plannable until either the admin's written answers or a repo or docs link exists, plus one authenticated call demonstrated against a dev instance. Agent A's output is planning input only for the fork analysis.
- **Platform facts (A):** primary docs or source at a named version. Forum posts alone stay UNVERIFIED.
- **Browser capability (B):** MDN plus a compatibility table, and a 30-line prototype by the builder that tails a real log file.
- **Game mechanics (C, D):** two independent sources, or one CCP source, dated after the last relevant patch.
- **Policy (E):** quoted clauses with URL from the rights holder only. Community interpretations are flagged.
- **Cross-wave:** any wave-2 claim that touches ESI scopes or rate limits must match the wave-1 output. Conflicts are escalated, not averaged.

## 5. Nexus hypothesis

**Hypothesis (unconfirmed):** Nexus is a custom or rebranded system rather than stock Alliance Auth. The distinct name and the "carries all ESI" framing suggest a purpose-built hub. Against that, Alliance Auth and SeAT are the commonest bases for alliance auth, so a rebranded instance of either is the leading alternative.

**Confirming tests, cheapest first:**
1. Ask the owner or admin: "what is it built on, and is there a repo?"
2. With consent, view the login page passively. Alliance Auth would show Django markers (`csrftoken` cookie, `allianceauth` static paths, `/admin/`). SeAT would show a Laravel session cookie and SeAT assets. Neither set of markers points to custom.
3. Check how its SSO app appears under the EVE account's authorised applications, and which scopes it requests.

If it is Alliance Auth, expect pressure towards mode (a), a Django plugin. That collides with the free choice of backend and should be put to the owner before any stack decision.
