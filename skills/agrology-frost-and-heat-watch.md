---
name: agrology-frost-and-heat-watch
description: Watch Agrology microclimate and weather forecasts for frost, extreme heat and irrigation stress, and reconcile them against the alerts Agrology raises on its own thresholds.
api: Agrology Public API v2
base_url: https://api.agrology.ag/v2
operations:
  - GET /access
  - GET /predictions/microclimate/metrics
  - GET /predictions/weather/metrics
  - GET /predictions/microclimate/{siteID}
  - GET /predictions/microclimate/{siteID}/{timeRange}
  - GET /predictions/microclimate/{siteID}/node/{nodeID}
  - GET /predictions/weather/{siteID}
  - GET /predictions/weather/{siteID}/{timeRange}
  - GET /alerts/customer/{customerID}
  - GET /historical/ground-truth/{siteID}/{timeRange}
generated: '2026-09-13'
method: generated
source: openapi/agrology-public-api-openapi.yml + https://github.com/agrology/public-api-docs/blob/main/README.md
---

# Watch for frost, heat and irrigation stress

Read-only. Combines Agrology's own threshold alerts with the underlying forecasts, so a
warning can be explained rather than just relayed.

> No operationIds exist for any of these operations in the published spec. Use METHOD +
> PATH.

## Credential

`Authorization: Bearer $ACCESS_TOKEN` or `x-api-key: $API_KEY`. For a watch loop that runs
unattended you need the API key — the bearer token expires after one hour and there is no
refresh flow.

## Step 1 — scope

```
GET /access
```

Collect `{customerID}` and the `{siteID}` / `{nodeID}` values under it.

## Step 2 — read what Agrology already flagged

```
GET /alerts/customer/{customerID}
```

Agrology raises these when a customer's configured thresholds are met. Each alert carries:

- `insight` — the alert class (e.g. `temperature`)
- `startTime` / `updateTime` — **ISO-8601 strings**, unlike every other timestamp in this
  API, which are epoch integers. Handle both.
- `sites[]` — the affected sites
- `eventValues.title` and `eventValues.headline` — human-readable, e.g. "Temperature is
  forecast to drop below 34 °F starting at 5am PST today at …"
- `eventValues.points[]` — `[latitude, longitude]` pairs. **Note the order is the reverse
  of the GeoJSON endpoints**, which use RFC 7946 longitude-first. Do not pass one straight
  into the other.

The envelope also carries an `aps` block, because this endpoint doubles as the mobile push
feed. Ignore it unless you are building a notifier.

**There is no webhook and no event stream.** Alerts must be polled. No polling interval is
published; no rate limits are published either, so poll conservatively.

## Step 3 — get the forecast behind the alert

Two independent forecast sources, both up to **four days ahead**:

```
GET /predictions/microclimate/{siteID}     # Agrology's own model: forecast + ground truth
GET /predictions/weather/{siteID}          # Tomorrow.io weather service
```

Omit the time range and the API assumes "now" and returns all future predictions — which
is what you want for a watch loop. Add `/{timeRange}` only to inspect a specific window,
and `/node/{nodeID}` to drop from site to individual sensor station.

The microclimate prediction is the one that matters for frost and canopy decisions: it is
Agrology's model over that *specific* microclimate, not a regional forecast. The weather
prediction is the regional third-party view. Disagreement between them is signal, not
error.

Resolve metric ids first from `GET /predictions/microclimate/metrics` and
`GET /predictions/weather/metrics` — the metric vocabulary is runtime data. Metrics
relevant to this flow include `airTemp`, `humidity`, `vpd` (vapor pressure deficit),
`dewPoint` and `windSpeed`.

## Step 4 — ground the forecast in what was actually measured

```
GET /historical/ground-truth/{siteID}/{timeRange}?metrics=airTemp,humidity,vpd
```

Use a short relative range such as `24h-`. Comparing the last measured readings against the
forecast is how you judge whether a prediction is tracking. Time-range grammar:
`{startTime}-{endTime}` as a path segment, the trailing hyphen required even when the end
is omitted; relative units are s/m/h/d/w with no compound forms; epoch forms must be
exactly 10 or 13 digits.

## Step 5 — interpret

- **Frost**: falling `airTemp` in the microclimate prediction, cross-checked against
  `dewPoint`. Agrology's own alert headline states the threshold it fired on — quote it
  rather than re-deriving one.
- **Extreme heat**: rising `airTemp` with rising `vpd`. VPD is the better stress signal
  than temperature alone.
- **Irrigation**: `vpd` plus soil metrics from ground truth (`soilMoisture`, `soilTension`,
  `waterPotential`), and `vwc` from `GET /synthetics/microclimate/{siteID}/{timeRange}`
  where the site has it.

Always report the units the metrics endpoint gives you — site `displayUnits` (observed
value `imperial`) is a display preference and does **not** change the payload, which is
metric (`˚C`, `%`, `m/s`).

## Two things that will bite a watch loop

1. **Restated history.** Agrology retrains its models and publishes
   (https://agrology.ag/model-updates, last reviewed 2026-05-27) that this changes values
   for **past** dates as well as present ones. A reading you stored yesterday can legitimately
   differ on refetch, with no version change and no header to detect it. Store the fetch
   time with every value, and never raise a "data changed" anomaly on that basis alone.
2. **Errors are untyped.** No operation declares a 4xx or 5xx. A missing credential returns
   `403` with `MissingAuthenticationTokenException` (the README says 401); an invalid one
   returns `401`. No rate limits are published and no `Retry-After` is returned — back off
   exponentially on any 429 or 5xx.
