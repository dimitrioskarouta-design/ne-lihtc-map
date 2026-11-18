# ne-lihtc-map — project context

A single-file interactive map for screening LIHTC sites in **Connecticut, Rhode Island
and Massachusetts**. Deployed at https://dimitrioskarouta-design.github.io/ne-lihtc-map/

Everything lives in `index.html` — no build step, no dependencies beyond CDN scripts.
Leaflet from unpkg; all data fetched at runtime from state GIS services.

---

## The one rule that must not be broken

**The three states' scores are never shown together, and are never comparable.**

Each state's QAP scores geography completely differently:

| state | what the colour means | denominator |
|---|---|---|
| CT | Housing Needs Index quintile + Affordable Housing Appeals List points | 10 of 10 — the *entire* town-level Housing Needs & Location score |
| RI | Community criterion A only | 13 of 147 total QAP points |
| MA | jobs access (B-6) + rural (B-7) | ~10 of 186 total points |

A CT "10" and an MA "10" mean nothing like the same thing. Only the **active** state is
coloured; the others render neutral grey. The legend always states the denominator.
Hovering an inactive state says "switch Active scoring to see points" rather than
showing a number. **Do not** add a combined view, a shared colour ramp, or a
cross-state ranking.

---

## Architecture

### Per-state config (`STATES` object, near the top of the script)

Adding a state = adding one config block. Nothing below the config should know which
state it is looking at. Each block supplies:

- `src` — boundary GeoJSON URL (an ArcGIS `/query?f=geojson` URL, or a static file)
- `nameOf(props)` — extract the municipality name from a feature
- `lookup(name)` — case-insensitive match into that state's baked data
- `list()`, `columns`, `row(n)` — sidebar table
- `color(n)`, `tooltip(n)`, `legend` — map rendering
- `score(name, utilResults)` — address-panel payload
- `utils[]` — utility overlay services. Each entry is `{id, label, color, url}` plus
  three optional hooks read by `loadUtil`/`drawUtil`: `extraQS` (raw query-string
  fragment appended to every page of `fetchAllPages`, e.g. a spatial envelope filter),
  `postProcess(rawFeatures)` (runs once on the raw fetch, before caching — the
  *processed* result is what gets cached and drawn), and `style` (a per-feature Leaflet
  style function instead of the default flat `color`/`fillColor`).
- `screenUtils` — which util ids are the sewer / water for this state
- `ingest(features)` *(optional)* — fold live attributes into the baked data
- `extra(latlng)` *(optional)* — extra async checks for the address panel
- `vintage`, `sources` — provenance strings shown in the footer

MA uses `ingest` to compute the B-7 rural test live from MassGIS `POP2020` /
`AREA_SQMI` rather than baking it in. MA uses `extra` for the tract poverty gate.
CT uses `extra` for the Location Index tract lookup, and a `utils[]` entry
(`postProcess`/`style`) for the matching map layer — see "CT Location Index" below.

### Data: "frozen" vs "live"

This distinction is load-bearing and is surfaced in the About panel.

- **Frozen** — transcribed from a PDF on the build date, will never self-update:
  CT scoring table, CT Location Index tract table, RI LMIH + growth categories, MA SHI
  + jobs chart, Gateway Cities.
- **Live** — refetched from the publisher on every load: all boundaries, all utility
  layers (including the CT Location Index's tract *geometry*, though its CHFA
  rank/quintile per tract is frozen), MA rural inputs, ACS tract poverty, HUD QCT.

### CT Location Index (tract-level, High/Very High only)

Drives the **Location Set-aside** (CHFA's target of 50% of New Construction awards in
high/very-high resource tracts) — it does **not** add points to the Housing Needs &
Location score, and must never be presented as if it does.

CHFA's own Guideline gives tract rank/quintile by town name + a `"NNNN.NN"`-style tract
code (baked into `CT_LOC`, keyed by town then code), but no public CT-specific tract
*geometry* service was available to build against from this project's sandbox. The map
instead reuses the tract boundaries already proven working for MA's poverty gate (Esri
Living Atlas `ACS_Poverty_by_Age_Boundaries/FeatureServer/2`, nationwide), bbox-filtered
to Connecticut via `extraQS` (a spatial envelope, not an attribute `where` — the exact
state-FIPS field name on that layer is unverified).

The join has no town field to key on, so for each fetched tract: average its outer
ring's coordinates as a cheap centroid, run it through the existing `locate()` against
the already-loaded CT town polygons to get the town, then search that tract's
properties for *any* value that is 6+ digits once non-digits are stripped (covers an
11-digit GEOID or a 6-digit TRACTCE regardless of the field's actual name), take the
last 6 digits as `TTTT.TT`, and look it up in `CT_LOC[town]`. Matching is scoped to the
one town found by geometry first, specifically so a coincidental digit collision in an
unrelated attribute (an OBJECTID, a population count) can't match a real code in the
*wrong* town. A tract that can't be matched this way is dropped, not mis-colored — if
Esri ever changes that layer's schema such that no property survives the digit-strip
heuristic, the layer will quietly render fewer tracts rather than wrong ones; the
`extra()` point-lookup used by the address panel shares the same logic and the same
failure mode.

If you add data, classify it and put it in the About panel. A future reader must be
able to tell what needs revisiting.

### Utility layer cache (IndexedDB, stale-while-revalidate)

CT DEEP's sewer service takes **12–19 seconds** on every request — it generalises very
complex geometry server-side. (Asking for ungeneralised geometry returns 28 MB.)
So utility layers are cached in IndexedDB:

- a cached copy renders **immediately**, regardless of age
- if older than 7 days it refreshes in the background and swaps in
- if a background refresh fails, the stale copy stays — the layer never vanishes
- "refresh now" link clears the cache and refetches

Measured: CT sewer 12,978 ms → **529 ms** cached.

Trade-off: layers can be up to a week stale. This is stated next to the toggles and in
the About panel. Do not silently extend the window.

### Defaults

Only the **active state's sewer layer** auto-enables, and it swaps on state change.
Water service is manual — its polygons blanket large areas and wash out the choropleth.
Manual toggles are remembered (`userTouchedUtil`) and never overridden.
Auto-loading is skipped when `navigator.connection.saveData` is set.

---

## Gotchas that cost real time

**ArcGIS layer indices are not 0.** MassDEP sewer is layer **2** (POTW), layer 1 is
non-POTW. MassDEP water is layer **2** (COM), layer 1 is NTNC. CT DPH ESA is layer
**1**. Esri ACS poverty is layer **2** (tract). Always fetch `FeatureServer?f=pjson`
and read the layer list before assuming.

**Some services only speak Esri JSON.** MassDEP and the Esri ACS layers advertise
`supportedQueryFormats: "JSON"` — `f=geojson` fails, sometimes with a bare HTTP 400
rather than a JSON error. `fetchAllPages()` falls back to `f=json` and converts,
using ring winding order to distinguish outer boundaries from holes.

**Paging is mandatory.** ArcGIS caps each query at `maxRecordCount`. A single
un-paged request silently returns a truncated set that looks complete. `fetchAllPages()`
pages, de-dupes by object id, and bails if a service ignores `resultOffset`.

**Temporal dead zone.** `applyBasemap()` runs at load and calls `restyle()`, which
reads `const G` declared further down. The call is wrapped in try/catch. `node --check`
will **not** catch this class of bug — it only parses. There's a `window.onerror`
handler that prints script errors into the sidebar; keep it near the top.

**Boundary name fields differ and are often uppercase.** RIGIS is `NAME` uppercase,
MassGIS is `TOWN` uppercase, CT is `TOWN`. All `lookup()` functions are
case-insensitive. Check actual *values*, not just field names.

**Map sizing.** `invalidateSize()` must complete before `fitBounds()` or it fits to
the old container and crops the region. `refitMap()` sequences them with a delay.

**Touch has no hover.** Tooltips are bound only when `TOUCH` is false; on touch,
tapping opens a dismissible popup with the same content. `TOUCH` is
`(hover: none), (pointer: coarse)`.

**Verify commits.** GitHub's web upload form fails silently and returns you to the
repo either way — a failed commit looks identical to a successful one. Using
`git push` from Claude Code avoids this entirely.

---

## Known gaps (deliberate, documented in the About panel)

- **MA MCAS school quality (8 pts)** — the QAP scores "Advanced or Proficient" 10th
  grade MCAS, a scale DESE no longer reports. Reference chart linked from EOHLC's own
  page is from 2013. Unresolved; a question for EOHLC, not a coding problem.
- **MA Housing Production Plan communities (2 pts)** — excluded on purpose. EOHLC
  publishes status only via a dashboard, and HPP status is volatile (5-year plans,
  annual re-certification). A frozen list would go stale quietly.
- **MA site-level B-6** — transit 0.5 mi (4), higher ed 2 mi (2), health care 1 mi (2).
  Not mapped. These belong in the address panel as buffer queries.
- **RI criterion B** — graduation rates (2 pts) and the CCRP half of the QCT test.
- **CT boundaries depend on a third-party GitHub repo** via jsDelivr
  (`HandsOnDataViz/ct-boundaries`). If it moves, CT goes blank while RI and MA keep
  working. Embedding the polygons would remove the risk (~1 MB).

## Data caveats that must survive

- **CT water ESA** is the weakest layer in the map: compiled 2017, updated 2018, and it
  maps who is *entitled* to serve, not where mains exist. Never present it as evidence
  of available water.
- **All three states** built utility layers the same two ways: a 300-foot buffer around
  supplied pipe locations, or the municipal boundary where a town is 97%+ served. A
  polygon edge is often a cartographic artifact. MassDEP says outright the data "is not
  appropriate for site-specific or parcel-specific analysis."
- **RI's conditional 5-point tier** is the one place a service-area boundary decides a
  score. Treat it accordingly.
- **MA Priority 5 uses SHI < 12%**, not the statutory 10% everyone quotes.
- **Gateway Cities**: the QAP text says 24; the statute now lists 26. The map uses 26.
- **CT scoring is the 2027–2028 adopted QAP/Guideline** (Board 23 Jul 2026, Governor
  28 Jul 2026). The per-town Housing Needs Index + Appeals List figures are byte-identical
  to the public-hearing draft this map used before — only the adoption status changed.
  Don't assume that holds next cycle.
- **CT Location Index is tract-level, not town-level**, and is an *eligibility* input
  for the Location Set-aside (CHFA's target of 50% of New Construction awards in
  high/very-high resource tracts) — it does **not** add points to the Housing Needs &
  Location score. Keep that distinction in any UI copy.
- **MA rural** uses MassGIS `AREA_SQMI`, which appears to include water. Bourne (498)
  and Halifax (446) sit right on the 500/sq mi line and could flip on land-only area.

## Watch list

- RI 2027 QAP — as of 1 Jan 2026 the Executive Office of Housing writes it, not
  RIHousing; a draft was in public comment in Aug 2026
- Newer EOHLC QAP and a newer SHI (updated ~every 2 years)
- RIHousing's annual LMIH chart
- HUD redesignates QCTs annually — the year is in the service URL
- Next CHFA Housing Needs & Location Guideline cycle — re-verify the Location Index
  join still holds (see "CT Location Index" under Architecture) if CHFA changes the
  geometry source's schema.

## Possible next work

Discussed and deliberately **not** built here — a parcel/zoning discovery tool is a
separate application, not a feature of this map. Different question ("where should I
look?" vs "how does this site score?"), different scale (MA has 2.56M parcels, CT
1.29M), and different privacy posture (parcel data carries owner names; this map is a
public URL with no auth).

Verified working: MA `Massachusetts_Property_Tax_Parcels` and CT
`Connecticut_CAMA_and_Parcel_Layer_2024` both answer attribute queries over millions of
records in 200–300 ms. CT's is richer — direct `Land_Acres`, assessed *and* appraised
values, and two sales deep. Neither has usable normalised zoning; CT's `State_Use` field
contains raw municipal zoning codes in some towns and junk in others.
