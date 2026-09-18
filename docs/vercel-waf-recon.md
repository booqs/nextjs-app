# Vercel WAF recon — booqs.app (2026-09-17)

Record for tracker issue `axis/vercel-waf-recon`, gating
`booqs/native-api-access`. Companion to `docs/native-api-access.md`
(§8 Option B, §9 dashboard checklist). Read-mostly recon through the
Vercel Firewall REST API with the CLI's stored token, plus one log-mode
rule. No bypass, deny or challenge rule was added or changed.

## Headline

**Option B is already live.** The `native-app-access` rule from §9 exists
on the project in **bypass** mode, active since 2026-07-29 20:16 UTC
(firewall config v7), exactly as the progress note in §9 records. The
recon premise — "confirm Apple's AASA fetcher is blocked the moment the
associated-domains commit deploys" — is moot: that commit (`268c0a2`,
`public/.well-known/apple-app-site-association`) is deployed, and the
file is served unchallenged to a plain client and to a client imitating
Apple's `swcd` fetcher. The native app's blocker (checkpoint on
`POST /api/graphql`) is gone. Web posture is preserved: `GET /` from
curl still returns the 429 checkpoint.

## Firewall configuration (project `booqs`, team `ashakirzianovs-projects`, Hobby plan)

Read from `GET /v1/security/firewall/config/active`, version 7 at the
start of recon.

| Item | State |
|---|---|
| Firewall | enabled |
| Attack Challenge Mode | **off** (`attack-status` reports no anomalies; project security has `firewallSeawallEnabled: false`) |
| Managed ruleset — Bot Protection (`bot_protection`) | **active, action `challenge`** (since 2026-06-24) |
| Managed ruleset — AI bots (`ai_bots`) | active, action `deny` (since 2026-05-25) |
| OWASP core ruleset (`crs`) | `gen`, `rce`, `xss`, `sqli` active in **log** mode; the rest inactive |
| BotID | off |
| JA3 / JA4 fingerprinting | on |
| IP blocklist | empty |
| Custom rule 1 — `Deny all access` | **inactive** (kill switch, deny on all methods) |
| Custom rule 2 — `native-app-access` | **active, bypass**; OR of: path starts `/api/graphql`, `/api/images`, `/api/upload`; path equals `/.well-known/apple-app-site-association` |
| Custom rule slots (Hobby: 3 total) | 2 used before recon, **3 used after** (see below) |

Domains on the project: `booqs.app`, `www.booqs.app`, `booqs.vercel.app`
(all verified). `img.booqs.app` is not on the project (CloudFront, per
§3 of the design doc).

Config history (`GET /v1/security/firewall/config/{n}`):

| v | when (UTC) | change |
|---|---|---|
| 1 | 2026-05-25 | AI bots managed ruleset → deny |
| 2 | 2026-05-25 | Bot Protection → log |
| 3 | 2026-06-24 | insert `Deny all access` (deny) |
| 4 | 2026-06-24 | Bot Protection → **challenge** |
| 5 | 2026-06-24 | update `Deny all access` (deactivated) |
| 6 | 2026-07-29 | insert `native-app-access` in **log** mode |
| 7 | 2026-07-29 | `native-app-access` → **bypass** |
| 8 | 2026-09-17 | insert `recon-log-native-paths` (log) — this recon |
| 9 | 2026-09-17 | reorder: `recon-log-native-paths` ahead of `native-app-access` — this recon |

Firewall activity (match counts, request log) is **not exposed by the
REST API** as far as this recon found: `attack-status` returns only DDoS
anomalies (none), `events` returns an empty `actions` list, and there
is no logs endpoint. Match volume per rule is a dashboard-only view
(Firewall overview, group by rule / path).

## What applies to the four design paths

All four paths are matched by `native-app-access` (bypass), so the Bot
Protection challenge, the AI-bots deny and the OWASP log rules are
skipped for them. Platform DDoS mitigation still applies (not subject
to bypass). Nothing else on the project touches these paths.

## Baseline probes (2026-09-17, from this machine, ~3 s apart, 7 requests + 2 re-checks)

`swcd` UA = `swcd (unknown version) CFNetwork/1568.100.1 Darwin/24.0.0`.

| Probe | Status | Content type | Checkpoint? | Body |
|---|---|---|---|---|
| `GET www.booqs.app/.well-known/apple-app-site-association`, plain curl | 200 | application/json | no | `{"webcredentials":{"apps":["US35J9JD4E.booqs.Booqs"]}}` |
| same, `swcd` UA | 200 | application/json | no | same |
| `GET booqs.app/.well-known/apple-app-site-association` (apex), `swcd` UA | 307 | text/plain | no | redirect to www |
| `GET /api/graphql?query={featured(limit:1){title}}`, plain curl | 200 | application/json | no | GraphQL data |
| same, `swcd` UA | 200 | application/json | no | GraphQL data |
| `POST /api/graphql` `{"query":"query { featured(limit: 1) { title } }"}`, plain curl | 200 | application/json | no | GraphQL data |
| `GET /` plain curl | **429** | text/html | **yes** (Vercel Security Checkpoint) | challenge page |

Re-check after adding the log rule: AASA (`swcd` UA) 200 JSON;
`POST /api/graphql` 200 JSON — unchanged.

Note on the apex: Apple fetches the AASA from the exact domain listed in
the app's associated-domains entitlement. `booqs.app` answers with a
307 to `www.booqs.app`; Apple's fetcher does follow redirects, but if the
entitlement lists only `www.booqs.app` this never matters. Worth a
glance at the entitlement when the native passkey flow is next tested.

## The rule as created

Inserted via `PATCH /v1/security/firewall/config` (`rules.insert`, then
`rules.priority` → index 1), read back from `config/active` v9:

- name: `recon-log-native-paths`
- description: `RECON ONLY (2026-09-17, axis/vercel-waf-recon). Log-mode counter for the native-access paths; ordered before native-app-access so matches are recorded. Safe to delete.`
- active: true
- action: **`log`** (confirmed on read-back; `rateLimit`, `redirect`, `actionDuration` all null)
- conditions (OR-joined groups): path starts `/api/graphql` · path starts `/api/images` · path starts `/api/upload` · path equals `/.well-known/apple-app-site-association`
- position: 1, ahead of `native-app-access` (bypass). It has to sit
  ahead: bypass skips every later rule, so a log rule after it would
  never record anything.

Cost: this uses the **third and last Hobby custom-rule slot**. The A
escalation (§7 rate-limit rule, §8) needs a slot, so this rule should be
deleted once its count has been looked at.

## Recommended next step for Option B

Nothing to approve for B itself — it is deployed and verified on all
three axes (native paths pass, root page still challenged, AASA served).
The decisions left are housekeeping:

1. Glance at the Firewall overview grouped by rule: `recon-log-native-paths`
   and `native-app-access` should show identical match counts; the
   per-path split shows whether `/api/graphql` volume looks scripted
   (§9 step 6 — the trigger for A).
2. Delete `recon-log-native-paths` after that glance to free the slot.
   Or keep it if the duplicate counter is useful, accepting that A then
   needs a deletion first.
3. Update `booqs/native-api-access` and `axis/vercel-waf-recon`: the
   rollout is complete; both were still `in-progress` because the
   2026-07-29 completion was recorded in the design doc but not in the
   tracker.

## Follow-up 2026-09-18 — match counts not readable outside the dashboard

Asked to read the match counts on `recon-log-native-paths` and then
delete it. The counts could not be read, so **the rule was left in
place** (config still v9, all three rules unchanged on read-back):

- `GET /v1/security/firewall/events` (with a two-day window, with and
  without `ruleId`) returns `{"actions":[]}`; `attack-status` returns no
  anomalies. Neither carries custom-rule match counts.
- `vercel firewall overview --json` (CLI 59.14.0 does expose the
  firewall; the earlier note that it does not was wrong) returns
  `stats`, `series` and `topRules` as null with
  `activityUnavailable: { reason: "plan", message: "Traffic and alerts
  need Observability Plus." }`. `vercel firewall rules inspect` shows
  configuration only.

So on the Hobby plan, per-rule, per-path and per-user-agent counts exist
only in the dashboard (Firewall overview, grouped by rule / path / user
agent). Deleting the rule was sequenced after reading the counts, and
the counts are the rule's only purpose, so deletion waits on either a
dashboard glance or an explicit "delete without counts". The delete
itself is one call: `vercel firewall rules remove recon-log-native-paths`
then `vercel firewall publish`.
