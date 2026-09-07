---
name: fcs-institution-lookup
description: >-
  Look up Farm Credit System institutions, their branch offices and their chartered territories
  from the Farm Credit Administration's public ArcGIS map services — anonymously, with no API key.
api: FCA Farm Credit System Map Services
generated: '2026-09-07'
method: derived
source: >-
  arcgis/farm-credit-administration-*.json — verbatim ArcGIS service and layer descriptors fetched
  from https://wgis.fca.gov/arcgis/rest/services/FCA on 2026-09-07. FCA publishes no OpenAPI, so
  every endpoint and field name below comes from those descriptors and from live 200 responses,
  not from a specification.
base: https://wgis.fca.gov/arcgis/rest/services/FCA
auth: none
operations:
  - GET /arcgis/rest/services/FCA?f=json
  - GET /arcgis/rest/services/FCA/hq/MapServer/0?f=json
  - GET /arcgis/rest/services/FCA/hq/MapServer/0/query
  - GET /arcgis/rest/services/FCA/branches/MapServer/0/query
  - GET /arcgis/rest/services/FCA/regions/MapServer/0/query
---

# Look up Farm Credit System institutions from FCA

The Farm Credit Administration regulates the Farm Credit System (FCS). It publishes three public
map services — and nothing else machine-readable. There is no developer portal, no OpenAPI and no
API key. Everything here is anonymous HTTPS GET.

| Service | Layer 0 | What it holds |
|---|---|---|
| `hq` | `ACA_FLCA_HEADQUARTER_POINT` | 55 institution headquarters: charter address, phone, county, CEO/chair surname, institution website, coordinates |
| `branches` | `BRANCH_OFFICES_POINT` | Branch offices: institution, address, city/state/ZIP, county, FIPS, phone, coordinates |
| `regions` | `ACA_FLCA_Institutions_REGION` | Chartered-territory polygons |

`UNINUM` is the institution identifier and it joins all three layers to each other **and** to the
quarterly Call Report bulk files.

## Rules you must follow

1. **Always send `f=json` (or `f=geojson`).** Without it the server returns an HTML page with
   HTTP 200. This is the most common way to get a wrong answer here.
2. **Check the body for `error` before trusting a 200.** Errors come back as
   `{"error":{"code":404,"message":"Service not found","details":[]}}` — with HTTP 200.
3. **Check `exceededTransferLimit`.** A truncated result set says so at the top level and gives you
   no cursor. Page with `resultOffset` / `resultRecordCount`; the server cap is 1000 rows.
4. **Field names are case-sensitive and upper-case.** Read them from the layer descriptor first if
   you are unsure: `GET .../MapServer/0?f=json`.
5. **This surface is read-only.** Capabilities are `Map,Query,Data`. There is nothing to create,
   update or undo, and no idempotency key to send.
6. **No rate limit is published and none is signalled.** Be conservative.

## Step 1 — confirm what is published

```
GET https://wgis.fca.gov/arcgis/rest/services/FCA?f=json
```

Returns `services[]` with `FCA/branches`, `FCA/hq`, `FCA/regions`. Use those names verbatim.

## Step 2 — find institutions by state

```
GET https://wgis.fca.gov/arcgis/rest/services/FCA/hq/MapServer/0/query
    ?where=CHARTER_ST%3D%27TX%27
    &outFields=UNINUM,SHORTNAME,CHARTER_CI,CHARTER_ST,PHONE,WEB_URL
    &returnGeometry=false
    &f=json
```

To count first instead of fetching: `&where=1%3D1&returnCountOnly=true` returns `{"count":55}`.

## Step 3 — find that institution's branches

Take `UNINUM` from step 2 and query the branches layer:

```
GET https://wgis.fca.gov/arcgis/rest/services/FCA/branches/MapServer/0/query
    ?where=UNINUM%3D710812
    &outFields=SHORTNAME,NAME,ADDRESS,CITY,STATE,ZIP_5,COUNTY,FIPS,PHONE
    &returnGeometry=false
    &f=json
```

Geographic searches work the same way — `where=STATE%3D%27IA%27`, `where=FIPS%3D%2719153%27`.

## Step 4 — get the chartered territory as GeoJSON

```
GET https://wgis.fca.gov/arcgis/rest/services/FCA/regions/MapServer/0/query
    ?where=UNINUM%3D710812&outFields=*&returnGeometry=true&f=geojson
```

Returns `application/geo+json`, a standard `FeatureCollection`. Coordinates are Web Mercator
(`wkid` 102100 / `latestWkid` 3857); add `&outSR=4326` if you need WGS 84 lon/lat.

## Step 5 — join to the financial data (this is NOT an API)

The financial picture of an institution — the Call Report — is **not** on this API. It is a
quarterly bulk download of comma-delimited text:

<https://www.fca.gov/bank-oversight/call-report-data-for-download>

Each quarter ships ~72 files. `INST0600.TXT` carries `UNINUM` plus the institution's short name and
address; the `D_*` files are the layouts. Join on `UNINUM` client-side. There is no endpoint for
this and no incremental feed — you download a quarter at a time.

## What you cannot do here

- There is no search across institutions and financials in one call.
- There is no history: the map services keep only the current snapshot
  (`archivingInfo.supportsHistoricMoment` is `false`).
- There is no webhook, event or change feed. Re-poll after each quarterly release.
