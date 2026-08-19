---
name: microsoft-linkedin-audit-ad-account-access
description: >-
  Enumerate the LinkedIn ad accounts an application can reach, separate test data from production,
  and audit who holds which role on each account.
api: microsoft-linkedin:marketing-api
generated: '2026-08-13'
method: generated
source: openapi/ + https://learn.microsoft.com/en-us/linkedin/marketing/
operations:
  - adAccounts_search
  - adAccounts_get
  - adAccountUsers_find
---

# Audit LinkedIn ad account access

An access audit answers two questions: which accounts can this token reach, and who can act on each
one. Both are finder calls, and both have a trap.

## Headers

```
Authorization: Bearer {access_token}
Linkedin-Version: 202607
X-Restli-Protocol-Version: 2.0.0
```

Scope `r_ads` is sufficient for the read path. `X-Restli-Protocol-Version: 2.0.0` is required for
most `/adAccounts` calls — omitting it produces a syntax exception on path variables rather than a
clear error.

## Step 1 — enumerate accounts (`adAccounts_search`)

```
GET https://api.linkedin.com/rest/adAccounts?q=search&search=(type:(values:List(BUSINESS)),status:(values:List(ACTIVE)))&sortOrder=DESCENDING
```

With no search criteria the finder returns every account the caller can access.

**The trap:** finders return test *and* non-test accounts unless you filter. For a production audit
append `search.test=false`. Note that `test` is the one search field that takes a scalar
(`test:true` / `test:false`), not the `(values:List(...))` document every other field uses.

**Pagination:** from version 202401 the search method uses cursor pagination — `pageSize` (max
1,000) and `pageToken`, with `nextPageToken` returned in the response `metadata` object. The older
`start`/`count` offset style still applies to other resources. Loop until `nextPageToken` is absent.

## Step 2 — read the serving state (`adAccounts_get`)

```
GET https://api.linkedin.com/rest/adAccounts/{adAccountID}
```

`servingStatuses` is the field that explains why an account is not delivering. A single-element
array of `RUNNABLE` means healthy. Anything else is a list of blockers: `STOPPED`, `BILLING_HOLD`,
`ACCOUNT_TOTAL_BUDGET_HOLD`, `ACCOUNT_END_DATE_HOLD`, `RESTRICTED_HOLD`, `INTERNAL_HOLD`. An audit
should report these, because `status: ACTIVE` with `servingStatuses: [BILLING_HOLD]` is an account
that looks fine and is delivering nothing.

`changeAuditStamps.created.actor` and `.lastModified.actor` give you the URNs and epoch-millisecond
timestamps for provenance.

## Step 3 — enumerate users and roles (`adAccountUsers_find`)

```
GET https://api.linkedin.com/rest/adAccountUsers?q=accounts&accounts=urn:li:sponsoredAccount:{adAccountID}
```

Roles, from most to least privileged:

| Role | Can |
|---|---|
| `ACCOUNT_BILLING_ADMIN` | everything, including deleting the account |
| `ACCOUNT_MANAGER` | manage the account and everything under it |
| `CAMPAIGN_MANAGER` | manage campaigns and campaign groups |
| `CREATIVE_MANAGER` | manage creatives |
| `VIEWER` | read only, even when the token carries `rw_ads` |

Flag any account whose billing-admin count is zero or greater than two — only the billing admin can
delete an account, so both extremes are a governance finding.

## Interpreting failures

- **404 on a path you know is right** — on the Ads APIs a 404 is often an entitlement problem, not a
  missing resource. Check that the app has the Advertising product approved.
- **403** — the token lacks the scope, or the member was never asked to grant it.
- **429** — throttled. There is no `Retry-After`; back off on your own schedule and check the app's
  Analytics tab in the Developer Portal for the endpoint's limit.
- **426** — the `Linkedin-Version` you sent has been sunset.

URNs in path segments must be URL-encoded. Full conventions:
`conventions/microsoft-linkedin-conventions.yml`.
