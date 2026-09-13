---
name: agrology-map-fields-geojson
description: Retrieve Agrology field and sensor-node geometry as RFC 7946 GeoJSON and follow its templated links from mapped geometry through to telemetry.
api: Agrology Public API v2
base_url: https://api.agrology.ag/v2
operations:
  - GET /access
  - GET /geojson/sites
  - GET /geojson/customer/{customerID}
  - GET /geojson/site/{siteID}
  - GET /sites
  - GET /nodes
generated: '2026-09-13'
method: generated
source: openapi/agrology-public-api-openapi.yml + https://github.com/agrology/public-api-docs/blob/main/README.md
---

# Map Agrology fields with GeoJSON

Read-only. Gets field and sensor geometry in a form any GIS stack ingests without a
connector, and uses it as the index into everything else in the API.

> None of these operations carries an operationId in the published spec. Address them by
> METHOD + PATH as written.

## Credential

`Authorization: Bearer $ACCESS_TOKEN` or `x-api-key: $API_KEY`. Either works on every
operation here.

## Step 1 — scope

```
GET /access
```

Gives you the `{customerID}` and `{siteID}` values you are allowed to ask about.

## Step 2 — choose your scope and fetch

```
GET /geojson/sites                      # everything the credential can see
GET /geojson/customer/{customerID}      # one customer
GET /geojson/site/{siteID}              # one site
```

Start at `/geojson/sites` when you do not yet know the shape of the estate; drop to the
site scope once you do.

## Step 3 — read the FeatureCollection

The body is a conformant RFC 7946 `FeatureCollection`. Two kinds of feature, told apart by
`properties.elementType`:

**`elementType: "site"`** — the growing location. Carries `name`, `customer`, `site`,
`role`, `country`, `displayUnits`, `timezone` (an IANA identifier, e.g.
`America/Los_Angeles`) and `siteClassificaiton`.

> `siteClassificaiton` is misspelled in Agrology's own published payload. Match it exactly
> as written; do not "correct" it.

> Site features are published with an **empty geometry object** (`{}`) rather than `null`.
> RFC 7946 allows a Geometry object or `null`, and `{}` is neither — some strict parsers
> will reject it. Guard for it.

**`elementType: "node"`** — a sensor station. Carries `id` (UUID), `name`,
`elevationMeters`, `crops[]`, `deviceIDs[]` and a `devices{}` map, with real `Point`
geometry:

```json
"geometry": { "type": "Point", "coordinates": [-122.602875, 45.556985] }
```

Coordinates are **longitude first, then latitude**, WGS 84 decimal degrees, per RFC 7946.
Reversing them is the classic failure here.

Each entry in `devices{}` gives `make`, `model`, `lastMessage` (epoch **milliseconds**),
`lastPayload` (a raw hex uplink frame) and `isMonitored`. `lastMessage` is a genuinely
useful liveness signal: a node whose devices all have a stale `lastMessage` is not
reporting.

## Step 4 — follow the templated links

Alongside the standard GeoJSON members, the root carries foreign members that are this
API's **only link-following affordance**:

```json
"historicalURL":  "https://api.agrology.ag/v2/historical/ground-truth/{site}",
"predictionsURL": "https://api.agrology.ag/v2/predictions/{dataset}/{site}",
"syntheticsURL":  "https://api.agrology.ag/v2/synthetics/{dataset}/{site}"
```

Substitute `{site}` with the site id and `{dataset}` with `microclimate` or `weather`, then
append a `{timeRange}` path segment. This is the intended route from a mapped node to its
readings — prefer it over hand-assembling paths, because it is the provider's own statement
of where the data lives.

A strict GeoJSON parser will drop these foreign members. If you need them, read them off
the raw JSON before handing the body to a geometry library.

## Step 5 — putting it to work

- **Ingest**: the FeatureCollection loads directly into QGIS, PostGIS, Leaflet, Mapbox or
  anything else that speaks GeoJSON.
- **Join**: node `id` is the key into every telemetry response's `nodes{}` map, and
  `deviceIDs[]` are the keys into `devices{}` and the `dev` field on each sample.
- **Filter**: `crops[]` and `elevationMeters` let you segment an estate before pulling
  telemetry, which matters because there is no pagination — narrowing first is how you keep
  response bodies manageable.

`GET /sites` and `GET /nodes` return the same topology without geometry; use them when you
only need the ids.

## Failures

`403` with `MissingAuthenticationTokenException` means no credential was sent (the README
says 401 — the gateway actually returns 403). `401` means the credential was rejected.
Requesting a site outside your access list fails; re-read `GET /access` rather than
guessing ids. No operation declares any error response, so there is no typed error path.
