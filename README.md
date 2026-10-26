# Agrology

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **Not from the company, and here with a question?** You are welcome here — we would rather be the
> front line and point you the right way than have a good report go nowhere. What this repository
> can answer is narrow, though, so it is worth knowing who you are actually looking for:
>
> - **A question about how the API works, an account, billing, or a bug in the service** — that is
>   the company's own support, not us. We profile this API; we do not operate it and cannot see
>   your account.
> - **A bug in an open-source project we only catalog** — file it on that project's own repository.
>   This has happened with a real and correct bug report that reached us instead of the people who
>   could fix it, which helped nobody.
> - **Anything about this listing itself** — the description, the tags, the rating, a missing or
>   wrong artifact — is ours. Open an issue here.
> - **Not sure, or something general about API Evangelist or APIs.io** — open an issue on the
>   [APIs.io Inbox](https://github.com/api-search/inbox) and we will route it.
>
> **This repository contains no software, and we will never ask you to download anything.** There is
> no build, release, installer, or binary here — only text and machine-readable API descriptions, so
> there is nothing here that can be "corrupt" or need "repairing". Any issue, comment, or email
> claiming otherwise and offering a download link is not from us and is hostile. Do not follow the
> link; it is a lure. Report it to GitHub and, if you like, tell us at
> [info@apievangelist.com](mailto:info@apievangelist.com) so we can take it down.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Agrology is a Delaware Public Benefit Corporation building a predictive agriculture
platform for specialty crops, vineyards and regenerative row-crop operations. In-field
sensor nodes capture ground-truth agronomic telemetry — soil moisture, tension,
conductivity and temperature, air temperature, humidity, vapor pressure deficit,
barometric pressure, total VOCs, CO2 and nitrous-oxide flux — and machine-learning models
turn it into microclimate predictions, synthetic metrics and threshold alerts for frost,
extreme heat, irrigation and smoke taint.

## The API

Agrology publishes a first-party **Agrology Public API v2** as OpenAPI 3.0.1 — 66 paths,
90 operations, 22 component schemas — from its own GitHub organization under Apache-2.0.

- Base URL: <https://api.agrology.ag/v2>
- Contract: <https://github.com/agrology/public-api-docs>
- Auth: bearer JWT (Cognito, one-hour lifetime) or a staff-issued `x-api-key`

The surface covers site and node topology, field geometry as RFC 7946 GeoJSON, historical
ground-truth sensor telemetry, historical and forecast weather data (Tomorrow.io),
ML-synthesized microclimate metrics, microclimate predictions, threshold alerts, reports,
dashboards, charts, customer inputs and field experiments.

## Notable findings from this profile

- **The contract is published openly under Apache-2.0**, which is uncommon and makes the
  spec forkable and diffable — it is also the only dated record of contract change, since
  there is no changelog.
- **Historical data is not immutable.** Agrology publishes a standing policy
  (<https://agrology.ag/model-updates>) that retraining its models restates values for
  *past* dates, with no version identifier, no as-of query and no header to detect it by.
  This is the single most consequential fact for any consumer building a ledger or a
  carbon claim on this data.
- **No operation declares a 4xx or 5xx response**, so a generated client has no typed
  error path. The docs also state a missing credential returns 401; the deployed gateway
  returns 403.
- **One of seven delete surfaces is reversible** — dashboards can be undeleted; everything
  else is permanent, and no retention window is published.
- **No idempotency mechanism exists** on any of the 31 mutating operations.
- **GeoJSON (RFC 7946) is the domain standard this API speaks**, and the templated
  `historicalURL` / `predictionsURL` / `syntheticsURL` links inside the GeoJSON body are
  its only link-following affordance.
- Agrology publishes no SDKs, no CLI, no sandbox, no pricing, no status page, no MCP
  server and no well-known documents on any of its hosts.

## Links

- Website: <https://agrology.ag>
- Grower's Portal: <https://grower.agrology.ag/>
- Agrology AI: <https://chat.agrology.ag>
- GitHub: <https://github.com/agrology>
- Contact: <https://agrology.ag/contact>
