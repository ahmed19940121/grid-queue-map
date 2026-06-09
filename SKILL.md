---
name: grid-queue-map- 
description: Use this skill whenever the user wants an interactive Great Britain electricity grid connection queue map built from live NESO TEC Register data — including any request to "run", "build", "regenerate", "update" or "improve" the grid queue map, change its technology focus (battery/storage, solar PV, onshore wind, offshore wind, demand), add or adjust the site_zone_lookup.csv workflow, or modify the map's filters, table, or styling. Trigger phrases include "grid queue map", "TEC queue map", "map the NESO queue", "show battery/solar/wind queue on a map", "GB connection queue map", "T1-T9 queue map", "connection headroom map", "where is grid capacity constrained", and "upload site zone lookup". Also trigger when the user asks to tweak or extend a previously generated gb-grid-queue-map.html file. Great Britain only.
---

# Grid Queue Map Improved — GB NESO TEC Queue (v2)

Deliver a standalone, browser-based HTML map of the Great Britain electricity connection
queue from the NESO TEC Register. The tool is credible first, visual second: it must never
invent project locations, T1–T9 zones, capacities, or queue counts.

## Fast path (default — use this unless the user asks for modifications)

A fully verified reference implementation is bundled with this skill. Do not regenerate it
from scratch; regeneration risks reintroducing bugs and breaking the honesty rules.

1. Copy `assets/gb-grid-queue-map.html` (relative to this SKILL.md) to the outputs
   directory, optionally renaming per the user's request.
2. Optionally verify the NESO API is reachable with ONE test call (the aggregate SQL below
   for battery). If it succeeds, mention the live headline numbers to the user. If it
   fails, still deliver the file — it is a live-fetching shell with its own error state —
   but tell the user the API was unreachable at generation time.
3. Present the file. Tell the user it fetches live data on open, starts in coarse
   HOST TO mode, and upgrades to T1–T9 site markers when they upload a trusted
   `site_zone_lookup.csv` (a template is downloadable inside the page).

## Modification path

When the user asks for changes, copy the asset to the working directory, edit it, then
re-verify before delivering:

- Extract the embedded `<script>` block and run `node --check` on it.
- Check for duplicate HTML `id` attributes.
- Keep every rule in "Core honesty rules" intact — they are non-negotiable regardless of
  what is being changed.

Map of the file for targeted edits:
- CSS design tokens: `:root` block at the top (`--p0`…`--p4` are the pressure ramp).
- Constants: `RESOURCE`, `API`, `TOPO_URLS`, `TECHS`, `RAMP`, `HOST_ANCHORS`, `CITIES`.
- SQL builders: `sqlAggregate()`, `sqlProjects()`.
- Lookup matching: `parseCSV`, `normName`, `diceCoeff`, `buildLookupIndex`, `matchRows`.
- Rendering: `renderCards`, `renderMap` (zoom + bubbles + markers), `renderReadout`,
  `visibleRows`/`renderTable` (filters + chunking), `exportCSV`.
- Orchestration: `refresh(force)` with per-tech in-memory `cache`.

## Core honesty rules (non-negotiable)

The TEC Register provides NO latitude/longitude and NO direct T1–T9 field. Geography comes
only from:
- `Connection Site` — best project-level location clue.
- `HOST TO` — broad transmission owner only: SHET (north Scotland), SPT (central &
  southern Scotland), NGET (England & Wales).

Therefore:
- Never claim HOST TO gives an exact project location.
- Never claim the TEC Register directly provides T1–T9.
- Never allocate NGET projects across T3–T9 proportionally.
- Never place a precise project marker unless Connection Site matched a trusted lookup.
- Unmatched records stay in coarse HOST TO buckets, clearly labelled as regional/coarse.
- Never colour T3–T9 from NGET counts unless rows are individually matched to T-zones.
- The HTML must always show this caveat: "The TEC Register does not provide coordinates
  or a direct T1–T9 field. T1–T9 is derived by matching Connection Site to a reference
  site/zone lookup. Records that cannot be matched are grouped only by HOST TO — SHET,
  SPT or NGET — so they are regional/coarse, not exact project locations."
- A static snapshot is only allowed if built at generation time from a successful live
  API response and stamped with the fetch date. Never invent a snapshot. The default
  deliverable performs the live fetch itself when opened and shows "live data
  unavailable" if the browser fetch fails.

## NESO API contract

Resource ID: `17becbab-e3e8-473f-b303-3806f43a6a10`
Endpoint: `https://api.neso.energy/api/3/action/datastore_search_sql?sql=<URL_ENCODED_SQL>`

Maximum TWO API calls per technology refresh: one aggregate, one project list. The asset
also caches results per technology in memory so toggling technologies does not refetch;
the "Refresh live data" button forces a new fetch.

Verified columns: Project Name, Customer Name, Connection Site, Stage, MW Connected,
MW Increase / Decrease, Cumulative Total Capacity (MW), MW Effective From, Project Status,
Agreement Type, HOST TO, Plant Type, Project ID, Project Number, Gate. Do not probe other
columns unless a query fails because NESO changed the dataset.

Aggregate SQL (substitute the pattern without surrounding % signs):

```sql
SELECT COUNT(*) total,
  SUM(CASE WHEN "HOST TO"=$$SHET$$ THEN 1 ELSE 0 END) shet,
  SUM(CASE WHEN "HOST TO"=$$SPT$$  THEN 1 ELSE 0 END) spt,
  SUM(CASE WHEN "HOST TO"=$$NGET$$ THEN 1 ELSE 0 END) nget,
  ROUND(SUM("Cumulative Total Capacity (MW)")/1000.0,1) gw,
  SUM(CASE WHEN "Agreement Type"=$$Direct Connection$$ THEN 1 ELSE 0 END) ag_direct,
  SUM(CASE WHEN "Agreement Type"=$$Embedded$$ THEN 1 ELSE 0 END) ag_embedded,
  SUM(CASE WHEN "Project Status"=$$Scoping$$ THEN 1 ELSE 0 END) st_scoping,
  SUM(CASE WHEN "Project Status"=$$Awaiting Consents$$ THEN 1 ELSE 0 END) st_awaiting,
  SUM(CASE WHEN "Project Status"=$$Consents Approved$$ THEN 1 ELSE 0 END) st_approved,
  SUM(CASE WHEN "Project Status"=$$Built$$ THEN 1 ELSE 0 END) st_built
FROM "17becbab-e3e8-473f-b303-3806f43a6a10"
WHERE "Plant Type" ILIKE $$%<PATTERN>%$$
```

Project-list SQL:

```sql
SELECT "Project Name" pn, "Customer Name" cn, "Connection Site" cs, "HOST TO" h,
       "Project Status" ps, "Agreement Type" agr,
       "Cumulative Total Capacity (MW)" cap, "Plant Type" plant
FROM "17becbab-e3e8-473f-b303-3806f43a6a10"
WHERE "Plant Type" ILIKE $$%<PATTERN>%$$
ORDER BY "Cumulative Total Capacity (MW)" DESC
```

SQL gotcha: never alias `Agreement Type` as `at` — it can behave as a reserved keyword.
Use `agr`.

## Technology mapping

| UI label | Plant Type ILIKE pattern |
|---|---|
| Battery / storage (default) | %Energy Storage System% |
| Solar PV | %PV Array% |
| Onshore wind | %Wind Onshore% |
| Offshore wind | %Wind Offshore% |
| Demand connections | %Demand% |

Plant Type is semicolon-delimited and multi-valued; ILIKE catches co-located/hybrid
projects. When Demand is selected, the page must show: "Demand is the TEC Register's
generic demand/load plant type. It is not a data-centre count. It may include
power-station auxiliary load, hydrogen/electrolyser schemes, green energy hubs, and other
transmission-connected demand." Never label Demand as data centres.

## Coarse mode (default before lookup upload)

Show one dashed pressure bubble per HOST TO area with live projects, at representative
regional anchor points — explicitly labelled "(coarse)" and tooltipped as HOST TO coarse
pressure only, not project locations or T1–T9 pressure:
- SHET ≈ 57.6, −4.4 (north Scotland)
- SPT ≈ 55.6, −3.9 (central & southern Scotland)
- NGET ≈ 52.6, −1.6 (England & Wales)

Bubble size reflects a transparent metric (GW); colour uses the five-step ramp ranked
across the three buckets. After a lookup upload, unmatched records stay in these buckets
while matched records get site markers.

## site_zone_lookup.csv workflow

Expected columns:
`site_name,aliases,lat,lon,neso_zone,host_to,dno,voltage_kv,gen_headroom,source,confidence_base`
(minimum useful: site_name, aliases, lat, lon, neso_zone, host_to, dno, voltage_kv).
The page includes a "Download template" button producing this header plus one worked
example row.

Matching procedure per TEC row:
1. Normalise both sides: lowercase; strip punctuation; remove matching-only words
   (substation, gsp, grid supply point, 400kv, 275kv, 132kv, kv); collapse whitespace.
2. Exact match against site_name, then against each semicolon-separated alias.
3. Conservative fuzzy match only if unambiguous (asset uses Dice coefficient ≥ 0.92 with
   a clear margin over the second-best candidate).
4. Multiple possible matches → mark ambiguous, do NOT place a marker.
5. No match → HOST TO fallback bucket only.

Confidence levels:

| Level | Meaning | Map treatment |
|---|---|---|
| High | Exact/alias/unambiguous-fuzzy lookup match | Marker at lookup lat/lon, T-zone assigned |
| Medium | Strong inferred local clue (optional, conservative — NOT in default asset) | Approximate area only, labelled inferred |
| Low | HOST TO only (incl. ambiguous matches) | Stay in SHET/SPT/NGET bucket |
| Unknown | No usable geography | Table only |

After upload, report the match outcome (matched / ambiguous / unmatched counts) next to
the upload control.

## Map and UI requirements

Real GB map (not a schematic) via CDN:
- d3 7.8.5 and topojson 3.0.2 from cdnjs.
- GB topology: `https://cdn.jsdelivr.net/npm/datamaps@0.5.10/src/js/data/gbr.topo.json`
  with unpkg as a fallback URL.
- `d3.geoMercator().fitSize(...)` over `topojson.feature(gb, gb.objects.gbr)`.

The page must include:
- Header with selected technology and a live / fetching / unavailable status pill.
- Technology selector + "Refresh live data" (forces refetch past the per-tech cache).
- Lookup upload control + template download + match-outcome status line.
- Headline cards: total projects, total GW, SHET/SPT/NGET counts, Direct Connection,
  Embedded, and the Scoping/Awaiting/Approved/Built status mix.
- Zoomable, pannable map (d3.zoom, scale 1–9, "Reset view" button) with coastline,
  orientation cities, coarse HOST TO bubbles, matched markers, and a legend covering the
  pressure ramp, the dashed coarse-bubble style, and matched markers.
- Click-to-filter: clicking a HOST TO bubble filters the table to that area (click again
  to clear); clicking a matched marker filters the table to that connection site.
- Plain-English read-out: coarse-mode explanation OR matched-zone league (least/most
  congested among matched rows only); HOST TO fallback counts and GW; unresolved
  projects/GW; Direct vs Embedded mix; status maturity (Scoping share vs Built); note
  that ≤~100 MW projects may suit DNO/distribution routes; two next steps
  (transmission pre-application with NGET/SHET/SPT; DNO pre-application + GSP headroom
  check).
- Project table with columns: Project, Customer, Connection Site, HOST TO, Derived Zone,
  Zone Source, Confidence, Project Status, Agreement Type, MW, DNO, Voltage. Filters:
  free-text search (debounced), HOST TO, Project Status, Confidence, minimum MW.
  Sortable headers. Chunked rendering (first 400 rows + "Show all"). CSV export covers
  the full filtered set, not just the rendered chunk.

## Colour ramp (queue pressure)

```
0 #1a9850  open / low
1 #91cf60
2 #fee08b
3 #fc8d59
4 #d73027  severe / high
```

Pressure is a rank/quantile score from matched project count and/or capacity — applied
to matched T-zones, and separately (clearly labelled coarse) to the three HOST TO
buckets. Make explicit which rows feed which score.

## Safe rendering and resilience

- Escape ALL live API strings before inserting into HTML (the asset's `esc()` helper);
  never assign raw API values to innerHTML.
- Parse numerics defensively; display "—" for missing values.
- 15-second fetch timeout via AbortController on every network call.
- Fetch failure → clear error notice, layout retained, no invented figures.

## Output requirement

Deliver a standalone `.html` file that works opened locally, fetches live NESO data on
open, and degrades gracefully if blocked. Never deliver only a screenshot or static
chart. Place the file in the outputs directory and present it. 
