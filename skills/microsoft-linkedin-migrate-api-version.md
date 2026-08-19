---
name: microsoft-linkedin-migrate-api-version
description: >-
  Move a LinkedIn Marketing API integration from a sunset or soon-to-sunset monthly version to a
  supported one, and put a recurring check in place so the twelve-month clock never runs out again.
api: microsoft-linkedin:marketing-api
generated: '2026-08-13'
method: generated
source: https://learn.microsoft.com/en-us/linkedin/marketing/versioning
operations:
  - adAccounts_search
  - adAccounts_get
---

# Migrate a LinkedIn API version

LinkedIn ships a new Marketing API version every month, supports each for a minimum of twelve
months, and then sunsets it hard. There is no grace period and no fallback: a sunset version returns
HTTP 426 on every call. Because LinkedIn emits no `Sunset` or `Deprecation` response headers, an
integration gets no runtime warning — the first signal is total failure.

## Step 1 — find out what version you are actually sending

Grep the codebase for `Linkedin-Version`. The value is a `YYYYMM` string, and it is frequently
hard-coded once and forgotten.

```
Linkedin-Version: 202511
```

Current version at the time of writing: **202607** (July 2026). Anything twelve or more months older
than the current release is at or past sunset.

## Step 2 — check it against the migration status table

Read <https://learn.microsoft.com/en-us/linkedin/marketing/integrations/migrations#api-migration-status>
for supported versions and their sunset dates. Recently sunset: 202507 (2026-07-15), 202506
(2026-06-15), 202505 (2026-05-15), 202504 (2026-04-15), 202503 (2026-03-16).

## Step 3 — read the changelog for every version you are skipping

This is the step people skip and regret. Migrating 202511 → 202607 crosses eight releases, and
breaking changes are per-release. From `changelog/microsoft-linkedin-changelog.yml`:

- **202603** — Lead Sync webhook validation became mandatory. Unvalidated webhook endpoints stop
  receiving new lead notifications entirely.
- **202605** — Events Management `endsAt` became required; the Conversions API changed which user
  identifiers it accepts as a sole identifier.
- **202607** — `SPONSORED_INMAILS` campaigns with a `LEAD_GENERATION` objective now default
  `creativeSelection` to `OPTIMIZED` instead of `ROUND_ROBIN`. Behaviour changes silently if you
  relied on the old default.

Each API resource also has its own migration guide; check the ones you call.

## Step 4 — test against a test ad account

Point a non-production build at your test ad account (`AdAccount.test = true`) with the new version
header and exercise every call path. Nothing under a test account serves or bills, so this is free.

Verify with a cheap read first:

```
GET https://api.linkedin.com/rest/adAccounts?q=search&search.test=true
Linkedin-Version: 202607
X-Restli-Protocol-Version: 2.0.0
```

A 200 proves the version is accepted. A 426 means you mistyped the version or picked one that is
already sunset.

Remember `search.test=false` for anything that touches production reporting — finders return test
entities by default.

## Step 5 — cut over and instrument

Change the header value in one place. LinkedIn's own guidance is that migration is usually nothing
more than updating the `YYYYMM` value — the resource paths rarely change; the behaviour behind them
does.

Then add the alarm that prevents a repeat:

- Assert on **426** specifically and page on it. It is the only signal LinkedIn gives.
- Diary a review at ten months from the version you pinned, not twelve.
- Subscribe to <https://www.linkedin-apistatus.com/> (email, Slack, webhook, RSS) for platform
  incidents — it will not tell you about sunsets, but it is the only operational feed LinkedIn runs.
- Watch the changelog: <https://learn.microsoft.com/en-us/linkedin/marketing/integrations/recent-changes>

## A note on the client libraries

The official `linkedin-api-client` packages on npm (0.3.0, 2023-02-07) and PyPI (0.3.0, 2023-06-29)
have not been released since 2023. They are thin Rest.li wrappers, so they do not encode version
behaviour and will not break on a version bump — but they also will not help you find what changed.
The version header is yours to manage.

Policy detail: `lifecycle/microsoft-linkedin-lifecycle.yml`.
