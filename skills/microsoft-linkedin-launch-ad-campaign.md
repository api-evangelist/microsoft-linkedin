---
name: microsoft-linkedin-launch-ad-campaign
description: >-
  Stand up a complete LinkedIn advertising structure — ad account, campaign group, campaign,
  creative — against the versioned Marketing API, using a test ad account so nothing serves or
  bills until you deliberately move to production.
api: microsoft-linkedin:marketing-api
generated: '2026-08-13'
method: generated
source: openapi/ + https://learn.microsoft.com/en-us/linkedin/marketing/
operations:
  - adAccounts_create
  - adAccounts_get
  - adCampaignGroups_create
  - adCampaigns_create
  - adCreatives_create
---

# Launch a LinkedIn ad campaign

LinkedIn advertising is a four-level hierarchy: **ad account → campaign group → campaign →
creative**. Each level is created under the one above it, and the parent's id appears in the child's
path. Get the order wrong and you get a 404, not a helpful error.

## Before the first call

Every request to the versioned surface needs three headers. LinkedIn applies no default version, so
omitting `Linkedin-Version` is an error, not a fallback.

```
Authorization: Bearer {access_token}
Linkedin-Version: 202607
X-Restli-Protocol-Version: 2.0.0
Content-Type: application/json
```

Required scope: `rw_ads`. The authenticated member must hold `ACCOUNT_BILLING_ADMIN`,
`ACCOUNT_MANAGER` or `CAMPAIGN_MANAGER` on the account — `VIEWER` is read-only even when the token
carries `rw_ads`.

## Step 1 — create a test ad account (`adAccounts_create`)

Do this first, once. LinkedIn has no sandbox environment; isolation comes from an immutable `test`
flag on a production account. Everything created underneath inherits it, never serves, and never
bills.

```
POST https://api.linkedin.com/rest/adAccounts
{
  "name": "Integration test account",
  "type": "BUSINESS",
  "currency": "USD",
  "reference": "urn:li:organization:{organizationId}",
  "test": true
}
```

Constraints that will bite you:

- `type` must be `BUSINESS`. `test: true` with `type: ENTERPRISE` fails with `CONDITIONAL_INVALID_VALUE`.
- One test account per developer application. A second attempt returns `TEST_ACCOUNT_LIMIT_REACHED`.
- `test` is set on create only. You cannot convert an existing account — that returns `FIELD_NOT_EDITABLE`.
- The new account id comes back in the `x-restli-id` response header, not in a body.

Read it back with `adAccounts_get` at `/rest/adAccounts/{adAccountID}` and confirm `"test": true`
before creating anything under it.

## Step 2 — create a campaign group (`adCampaignGroups_create`)

```
POST https://api.linkedin.com/rest/adAccounts/{adAccountID}/adCampaignGroups
```

The campaign group is the budget and scheduling envelope. Do not set `test` on it — the flag is
inherited, and passing it explicitly returns `FIELD_NOT_ALLOWED`.

## Step 3 — create a campaign (`adCampaigns_create`)

```
POST https://api.linkedin.com/rest/adAccounts/{adAccountID}/adCampaigns
{
  "account": "urn:li:sponsoredAccount:{adAccountID}",
  "campaignGroup": "urn:li:sponsoredCampaignGroup:{campaignGroupID}",
  "associatedEntity": "urn:li:organization:{organizationId}",
  "name": "Q3 demand gen",
  "type": "TEXT_AD",
  "costType": "CPC",
  "dailyBudget": { "amount": "25", "currencyCode": "USD" },
  "status": "DRAFT",
  "locale": { "country": "US", "language": "en" },
  "targetingCriteria": { }
}
```

Create it in `DRAFT` and promote to `ACTIVE` with `adCampaigns_update` once the creative is attached
and reviewed. Outside a test account an `ACTIVE` campaign begins spending real money — treat the
status transition as the point of no return, not the create call.

## Step 4 — create a creative (`adCreatives_create`)

```
POST https://api.linkedin.com/rest/adAccounts/{adAccountID}/adCreatives
```

Under a test account every creative is auto-rejected in review by design, and no analytics are
produced because nothing is served. That is expected, not a failure — do not build retry logic
around it.

## Rules that apply to every step

- **No idempotency key exists.** A retried `POST` creates a second entity. Before retrying a create
  that timed out, run the matching `*_search` operation filtered by name to see whether the first
  attempt landed.
- **Partial updates are POSTs.** Use `X-RestLi-Method: PARTIAL_UPDATE` and a body of
  `{"patch":{"$set":{"field":"value"}}}`. A successful update returns `204 No Content`.
- **Finders return test entities by default.** Any search must pass `search.test=false` to exclude
  test data from production reporting, or `search.test=true` to look only at test data.
- **429 carries no `Retry-After`.** Back off with your own exponential schedule; there is no header
  telling you how long to wait or how much quota remains.
- **426 means your version was sunset.** Bump `Linkedin-Version` to a supported `YYYYMM` value; see
  `lifecycle/microsoft-linkedin-lifecycle.yml`.
- Errors arrive as `{"message": …, "serviceErrorCode": …, "status": …}`. Full catalog:
  `errors/microsoft-linkedin-problem-types.yml`.
