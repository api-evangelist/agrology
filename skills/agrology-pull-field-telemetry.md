---
name: agrology-pull-field-telemetry
description: Pull measured sensor telemetry for an Agrology site or node over a time range, with the metric vocabulary resolved first and the time-range grammar applied correctly.
api: Agrology Public API v2
base_url: https://api.agrology.ag/v2
operations:
  - GET /access
  - GET /historical/ground-truth/metrics
  - GET /historical/ground-truth/device-types
  - GET /historical/ground-truth/{siteID}/{timeRange}
  - GET /historical/ground-truth/{siteID}/node/{nodeID}/{timeRange}
  - GET /historical/ground-truth/{siteID}/device/{deviceID}/{timeRange}
  - GET /historical/ground-truth/{siteID}/devices
generated: '2026-09-13'
method: generated
source: openapi/agrology-public-api-openapi.yml + https://github.com/agrology/public-api-docs/blob/main/README.md
---

# Pull Agrology field telemetry

Read-only. Retrieves measured ground-truth readings from Agrology's in-field sensors.

> The Agrology spec assigns operationIds to only 5 of its 90 operations, and none of them
> is in this flow. Every step below is therefore addressed by METHOD + PATH exactly as
> published in `openapi/agrology-public-api-openapi.yml`. Do not invent operationIds.

## Credential

Send exactly one of:

- `Authorization: Bearer $ACCESS_TOKEN` — a Cognito JWT from the Grower's Portal footer at
  https://grower.agrology.ag/. **Expires one hour after issue, with no refresh flow.**
- `x-api-key: $API_KEY` — issued by Agrology staff. Use this for anything unattended.

## Step 1 — resolve what you may read

```
GET /access
```

Never skip this. Authorization is an entity access list, not OAuth scopes, so this is the
only way to learn which customers, sites and nodes the credential can address.

Take `siteAccess[].sites` keys as your `{siteID}` values and `sites[].nodes[].id` as your
`{nodeID}` values. Identifier shapes are **not** uniform — older sites are slugs
(`austin-research-station`), newer ones are UUIDs, all nodes are UUIDs, devices are
hardware ids (`70B3D57BA000163E`). Do not validate any of them as a UUID.

## Step 2 — resolve the vocabulary before you filter

```
GET /historical/ground-truth/metrics
GET /historical/ground-truth/device-types
```

Metric ids are runtime data, not a fixed schema. Read `id`, `displayName` and `units` from
the first; `id` from the second. Filtering with a value that is not in these lists is
undefined behaviour — the docs do not say whether it errors, returns empty, or is ignored.

## Step 3 — build the time range

The range is a **path segment**, `{startTime}-{endTime}`, not a query parameter. The end
may be omitted (defaults to now) but **the hyphen is still required**.

| Form | Rule | Example |
|---|---|---|
| relative | `<int><unit>`, unit ∈ s m h d w | `6d`, `36h` |
| epoch seconds | exactly 10 digits | `1618203722` |
| epoch ms | exactly 13 digits | `1618203722000` |
| minute | exactly 12 digits `yyyyMMddhhmm`, starts `20`, UTC | `202108010000` |
| second | exactly 14 digits `yyyyMMddhhmmss`, starts `20`, UTC | `20210801052530` |

Digit count is how the API tells the formats apart, so **zero-pad and never trim**.
Compound relative times (`2h5m`) are not supported. ISO-8601 is not accepted as input.

Common ranges: `4h-` (last four hours), `6h-4h` (a two-hour window ending four hours ago),
`1618203722-1618290149` (explicit window).

## Step 4 — fetch

Pick the narrowest scope that answers the question:

```
GET /historical/ground-truth/{siteID}/{timeRange}
GET /historical/ground-truth/{siteID}/node/{nodeID}/{timeRange}
GET /historical/ground-truth/{siteID}/device/{deviceID}/{timeRange}
```

Always filter. There is **no pagination** in this API, so the time range and the filters
are your only volume control — a wide unfiltered range returns one very large body:

```
GET /historical/ground-truth/{siteID}/{timeRange}?deviceType=co2&metrics=co2Concentration
```

To enumerate a site's devices first: `GET /historical/ground-truth/{siteID}/devices`.

## Step 5 — read the response correctly

```
{ "status": "ok",
  "request": { "requestTime": ..., "source": "historical/ground-truth", "responseVersion": "2.1" },
  "sites":   { "<siteId>": {...} },
  "devices": { "<deviceId>": { "deviceType": ..., "position": ... } },
  "nodes":   { "<nodeUuid>": { "name": ..., "samples": [ { "ts": <epoch s>, "dev": ..., "d": {...} } ] } } }
```

- Samples are grouped by node and chronological **within** a node. **Node ordering is
  explicitly not guaranteed** and can differ between two identical requests — key on node
  id, never on array position.
- `d` is an **open map** of metric id to value. Its keys vary by device type. Join to the
  metrics list from step 2 for display names and units; do not hard-code keys.
- `ts` is epoch seconds.

## Failure handling

| What you see | What it means | What to do |
|---|---|---|
| `403 {"message":"Forbidden"}`, `x-amzn-errortype: MissingAuthenticationTokenException` | No credential was sent | Attach the header. The README says this is a 401; the deployed gateway returns 403. Branch on both. |
| `401` | Credential present but rejected — invalid key, or a bearer token past its one-hour life | Re-collect the token from the Grower's Portal, or check the key |
| `403` on a specific site | Valid credential, entity outside the access list | Re-read `GET /access` and only address what it returns |

No operation in this spec declares a 4xx or 5xx response, so there is no typed error path.
No rate limits are published and no `Retry-After` is returned — use conservative
concurrency and exponential backoff on any 429 or 5xx.

## Before you cache any of this

Agrology publishes a standing policy (https://agrology.ag/model-updates, last reviewed
2026-05-27) that retraining its models **changes values for dates in the past**. Derived
readings, recent history and aggregates all shift, with no version change and no header to
detect it by. Record the fetch time alongside every value, and never treat Agrology history
as immutable — there is no way to pin a model version or query as-of a prior model.
