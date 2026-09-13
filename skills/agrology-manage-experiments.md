---
name: agrology-manage-experiments
description: Create and run Agrology field experiments — trial versus control groups of sensor nodes with tracked metrics — and read back the computed time series, with the irreversibility of every delete made explicit.
api: Agrology Public API v2
base_url: https://api.agrology.ag/v2
operations:
  - GET /access
  - GET /experiments/customer/{customerID}
  - POST /experiments/customer/{customerID}
  - GET /experiments/{customerID}/{id}
  - PUT /experiments/{customerID}/{id}
  - DELETE /experiments/{customerID}/{id}
  - POST /experiments/groups/{customerID}/{experimentID}
  - GET /experiments/groups/{customerID}/{experimentID}
  - POST /experiments/members/{customerID}/{experimentID}/{groupID}/{nodeID}
  - GET /experiments/members/{customerID}/experiment-id/{experimentID}
  - POST /experiments/metrics/{customerID}/experiment-id/{experimentID}
  - GET /experiments/data/{customerID}/e/{experimentID}/{resolution}/{timeRange}
  - GET /experiments/data/{customerID}/e/{experimentID}/g/{groupID}/{resolution}/{timeRange}
  - POST /experiments/data/{customerID}/e/{experimentID}/run
  - POST /experiments/data/{customerID}/e/{experimentID}/regenerate
generated: '2026-09-13'
method: generated
source: openapi/agrology-public-api-openapi.yml
---

# Run an Agrology field experiment

**This skill writes.** Experiments are the largest surface in the API — 26 of its 90
operations — and the one where a mistake is least recoverable.

> Read `## Consequences` before the first POST.

## Consequences

- **No idempotency, anywhere.** There is no `Idempotency-Key` header in the spec or the
  docs, and none of the 31 mutating operations declares replay protection. A retried
  `POST /experiments/customer/{customerID}` creates a **second experiment** with a new
  server-assigned id, and there is no documented way to detect or collapse the duplicate.
  Never blind-retry a create. On a timeout, LIST first and check before retrying.
- **No delete in this flow is reversible.** Experiments, groups, members and metrics all
  delete permanently. Nothing here has an undelete (the only undelete in the whole API is
  for dashboards), and no retention window is published. Treat every DELETE as final and
  confirm with a human.
- **No dry run.** Nothing supports preview, validate or simulate.
- **`regenerate` is destructive to computed data.** It discards the existing computed
  series and rebuilds it. Prefer `run`.
- **`PUT` is naturally safe to retry.** The caller supplies the full composite key in the
  path, so a replay overwrites rather than duplicates. This is a property of the URL
  design, not a documented guarantee — but it is why updates are safer than creates here.

## Credential

`Authorization: Bearer $ACCESS_TOKEN` or `x-api-key: $API_KEY`. A write session that takes
longer than an hour needs the API key, because the bearer token expires and cannot be
refreshed.

## Step 1 — scope, and check for an existing experiment

```
GET /access
GET /experiments/customer/{customerID}
```

Always list before creating. Without idempotency, listing is the only duplicate guard you
have.

## Step 2 — create the experiment

```
POST /experiments/customer/{customerID}
```

Body is an `Experiment`. `customerID` and `id` are required; the useful optional fields are
`description`, `status`, `notes`, `startTime`, `endTime` (epoch, int64), `positionFilter`,
and the `notice` / `noticeLevel` / `noticeExpires` trio for surfacing a banner to portal
users.

Returns 201. Capture the id — **if the call times out, LIST rather than retry.**

## Step 3 — create the groups

```
POST /experiments/groups/{customerID}/{experimentID}
```

An `ExperimentGroup` requires `experimentID` and `id`, and takes `description`,
`groupType`, `notes` and `icon`. A trial design is normally at least two: treatment and
control.

Read them back with `GET /experiments/groups/{customerID}/{experimentID}`.

## Step 4 — assign nodes to groups

```
POST /experiments/members/{customerID}/{experimentID}/{groupID}/{nodeID}
```

`ExperimentMember` is the join between a sensor node and a group. **This create is
naturally idempotent** — the whole composite key is in the path, so a replay is safe. That
makes it the one write in this flow you may retry freely.

Use node UUIDs from `GET /access` or from the GeoJSON endpoints. Verify assignments with:

```
GET /experiments/members/{customerID}/experiment-id/{experimentID}
GET /experiments/members/{customerID}/group-id/{groupID}
GET /experiments/members/{customerID}/node-id/{nodeID}
```

The third answers the reverse question — which experiments a given node is enrolled in —
which matters before you re-use a node in a second trial.

## Step 5 — choose the tracked metrics

```
POST /experiments/metrics/{customerID}/experiment-id/{experimentID}
```

Metric ids come from the runtime vocabulary endpoints, not from a fixed enum — resolve
them from `GET /historical/ground-truth/metrics` (and the synthetics or predictions metrics
endpoints where the trial uses those datasets) before posting.

## Step 6 — compute and read the series

```
POST /experiments/data/{customerID}/e/{experimentID}/run          # 204, incremental
POST /experiments/data/{customerID}/e/{experimentID}/regenerate   # 204, DESTRUCTIVE rebuild
```

Then read:

```
GET /experiments/data/{customerID}/e/{experimentID}/{resolution}/{timeRange}
GET /experiments/data/{customerID}/e/{experimentID}/g/{groupID}/{resolution}/{timeRange}
```

The second is the one that answers the actual research question — per-group series is how
treatment is compared against control.

`{timeRange}` uses the standard grammar: `{startTime}-{endTime}` as a path segment, the
hyphen required even when the end is omitted; relative units s/m/h/d/w with no compound
forms; epoch forms exactly 10 or 13 digits. `{resolution}` is a required path segment whose
accepted values are **not published** — read an existing experiment's working URL from the
Grower's Portal to learn them rather than guessing.

## Step 7 — amend rather than delete

```
PUT /experiments/{customerID}/{id}
PUT /experiments/groups/{customerID}/{experimentID}/{id}
PUT /experiments/members/{customerID}/{experimentID}/{groupID}/{nodeID}
PUT /experiments/metrics/{customerID}/{experimentID}/{id}
```

Setting `status` or `endTime` on the experiment closes it out while preserving the record.
Prefer this to deleting. The DELETE equivalents exist at the same paths and are permanent.

## Failures

No operation declares a 4xx or 5xx response, so there is no typed error path. `403` with
`MissingAuthenticationTokenException` means no credential was sent (despite the README
saying 401); `401` means the credential was rejected. No rate limits and no `Retry-After`
are published — back off exponentially, and remember that a retry on a create is not safe.

## Reporting a defect

The API carries its own feedback channel — `POST /feedback` with `feedbackType` of
`feature`, `bug` or `other`, plus `customer`, `client`, `clientVersion` and `remarks`. Set
`isTestMessage: true` for anything that is not a genuine report.
