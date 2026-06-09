---
name: grid-queue-map-improved
description: Use this skill when the user wants an improved interactive Great Britain electricity grid connection queue map using live NESO TEC Register data. It visualises battery/storage, solar PV, onshore wind, offshore wind, or generic demand connections across GB, starts with honest HOST TO regional buckets, and upgrades to T1-T9 project/site mapping when the user provides a trusted site_zone_lookup.csv. Trigger phrases include "improved grid queue map", "interactive TEC queue map", "map NESO queue", "show battery/solar/wind queue on a map", "GB connection queue map", "T1-T9 queue map", "upload site zone lookup", "map grid capacity", "connection headroom map", and "where is grid capacity constrained". Great Britain only.
---

# Grid Queue Map Improved — Great Britain NESO TEC Queue

Build a standalone, browser-based HTML map that visualises the Great Britain electricity connection queue from the NESO TEC Register. The tool must be credible first and visually useful second: it must never invent precise project locations, T1-T9 zones, capacities, or queue counts.

The improved version has five key upgrades over the basic grid-queue-map skill:

1. A technology switcher inside the HTML, so the user can switch between battery/storage, solar PV, onshore wind, offshore wind, and demand connections without regenerating the file.
2. A user-uploaded `site_zone_lookup.csv` workflow, so coarse HOST TO-only data can be upgraded into matched T1-T9 zones and map markers.
3. Confidence labels for every project row: High, Medium, Low, or Unknown.
4. A searchable/filterable project table with CSV export.
5. Safer rendering, including escaping live API values before injecting them into HTML.

---

## Core honesty rule

The NESO TEC Register does **not** provide latitude/longitude and does **not** provide a direct T1-T9 transmission zone field.

The relevant geography fields in the TEC Register are:

- `Connection Site` — best available project-level location clue.
- `HOST TO` — broad transmission owner only:
  - `SHET` = Scottish Hydro Electric Transmission / north Scotland transmission area.
  - `SPT` = SP Transmission / central and southern Scotland transmission area.
  - `NGET` = National Grid Electricity Transmission / England and Wales transmission area.

Therefore:

- Never claim `HOST TO` gives an exact project location.
- Never claim the TEC Register directly provides T1-T9.
- Never allocate NGET projects across T3-T9 proportionally.
- Never place a precise project marker unless `Connection Site` has been matched to a trusted lookup.
- If no reliable match exists, keep the project in a coarse HOST TO bucket.

Always include this user-facing caveat in the HTML:

> The TEC Register does not provide coordinates or a direct T1-T9 field. T1-T9 is derived by matching `Connection Site` to a reference site/zone lookup. Records that cannot be matched are grouped only by `HOST TO` — SHET, SPT or NGET — so they are regional/coarse, not exact project locations.

---

## Technology mapping

The HTML must include a technology selector using these mappings:

| UI label | TEC Register `Plant Type` ILIKE pattern |
|---|---|
| Battery / storage | `%Energy Storage System%` |
| Solar PV | `%PV Array%` |
| Onshore wind | `%Wind Onshore%` |
| Offshore wind | `%Wind Offshore%` |
| Demand connections | `%Demand%` |

Default to battery/storage.

`Plant Type` is semicolon-delimited and multi-valued, so use `ILIKE` to catch co-located and hybrid projects.

### Demand caveat

If the selected technology is Demand, show this warning clearly:

> Demand is the TEC Register’s generic demand/load plant type. It is not a data-centre count. It may include power-station auxiliary load, hydrogen/electrolyser schemes, green energy hubs, and other transmission-connected demand.

Do not label Demand as data centres.

---

## NESO API resource

Resource ID:

```text
17becbab-e3e8-473f-b303-3806f43a6a10
```

Use the CKAN SQL endpoint:

```text
https://api.neso.energy/api/3/action/datastore_search_sql?sql=<URL_ENCODED_SQL>
```

Verified columns:

```text
Project Name
Customer Name
Connection Site
Stage
MW Connected
MW Increase / Decrease
Cumulative Total Capacity (MW)
MW Effective From
Project Status
Agreement Type
HOST TO
Plant Type
Project ID
Project Number
Gate
```

Do not probe columns unless NESO changes the dataset and the query fails.

---

## API efficiency contract

Use no more than two NESO API calls per technology refresh:

1. One aggregate SQL call for headline totals.
2. One project-list SQL call for project rows and matching.

The standalone HTML should perform the live fetch itself when opened. Do not hard-code live counts or capacities.

A static snapshot fallback is only allowed if it is built at generation time from a successful live API response and stamped with the fetch date. If live fetch fails during generation, do not invent a snapshot. The HTML should still function as a live-fetching shell and display “live data unavailable” if the browser fetch fails.

---

## Aggregate SQL

Build this query dynamically from the selected pattern, without the surrounding `%` in the JavaScript variable if you re-add `%` inside SQL.

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
WHERE "Plant Type" ILIKE $$%<PATTERN_WITHOUT_PERCENT_SIGNS>%$$
```

Read values from `result.records[0]`.

---

## Project-list SQL

Use this as the optional but recommended second call:

```sql
SELECT "Project Name" pn,
       "Customer Name" cn,
       "Connection Site" cs,
       "HOST TO" h,
       "Project Status" ps,
       "Agreement Type" agr,
       "Cumulative Total Capacity (MW)" cap,
       "Plant Type" plant
FROM "17becbab-e3e8-473f-b303-3806f43a6a10"
WHERE "Plant Type" ILIKE $$%<PATTERN_WITHOUT_PERCENT_SIGNS>%$$
ORDER BY "Cumulative Total Capacity (MW)" DESC
```

Important SQL gotcha: do not alias `Agreement Type` as `at`; `AT` can behave as a reserved keyword. Use `agr`.

---

## `site_zone_lookup.csv` upgrade workflow

The HTML should include a file input allowing the user to upload a trusted CSV lookup.

Expected columns:

```csv
site_name,aliases,lat,lon,neso_zone,host_to,dno,voltage_kv,gen_headroom,source,confidence_base
```

Minimum useful columns:

```csv
site_name,aliases,lat,lon,neso_zone,host_to,dno,voltage_kv
```

### Matching procedure

For each TEC project row:

1. Normalise `Connection Site` and lookup names:
   - lowercase;
   - strip punctuation;
   - remove matching-only words such as `substation`, `gsp`, `grid supply point`, `400kv`, `275kv`, `132kv`, `kv`;
   - collapse whitespace.
2. Exact match `Connection Site` against `site_name`.
3. Exact match against each semicolon-separated alias.
4. Conservative fuzzy match only if score is high enough and unambiguous.
5. If multiple possible matches exist, mark ambiguous and do not place a precise marker.
6. If no match exists, use `HOST TO` fallback only.

### Confidence levels

| Level | Meaning | Map treatment |
|---|---|---|
| High | `Connection Site` exact or alias match to trusted lookup | Place marker at lookup lat/lon and assign T1-T9 |
| Medium | Strong inferred local clue, but not exact site | Optional approximate area only; label as inferred |
| Low | `HOST TO` only | Do not place exact marker; keep in SHET/SPT/NGET bucket |
| Unknown | No usable geography | Keep in table only |

The default improved HTML may start with High/Low/Unknown only. Medium inference is optional and should be conservative.

---

## Map requirements

Render a real GB map, not a schematic.

Use D3 and TopoJSON from CDN:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js"></script>
```

Use a GB topology file such as:

```text
https://cdn.jsdelivr.net/npm/datamaps@0.5.10/src/js/data/gbr.topo.json
```

Project with:

```js
const feats = topojson.feature(gb, gb.objects.gbr);
const proj = d3.geoMercator().fitSize([mapWidth, mapHeight], feats);
const [x, y] = proj([lon, lat]);
```

Only place project/site markers when lat/lon come from a trusted lookup match.

---

## UI requirements

The generated HTML should include:

1. Header with selected technology and live/snapshot/unavailable status.
2. Technology selector.
3. Optional `site_zone_lookup.csv` upload control.
4. Headline cards:
   - total projects;
   - total GW;
   - SHET count;
   - SPT count;
   - NGET count;
   - Direct Connection count;
   - Embedded count;
   - Scoping / Awaiting Consents / Consents Approved / Built.
5. Real GB map with:
   - coastline;
   - major cities for orientation;
   - coarse HOST TO regional bucket labels;
   - matched markers only where lookup confidence permits;
   - legend showing open to severe queue pressure.
6. Read-out panel showing:
   - matched to T-zone: project count and GW;
   - unresolved by T-zone: project count and GW;
   - HOST TO fallback counts;
   - greenest / least congested matched zones if available;
   - reddest / most congested matched zones if available;
   - caveat if no lookup is uploaded.
7. Searchable project table with columns:
   - Project;
   - Customer;
   - Connection Site;
   - HOST TO;
   - Derived Zone;
   - Zone Source;
   - Confidence;
   - Project Status;
   - Agreement Type;
   - MW;
   - DNO;
   - voltage.
8. Export visible/filtered table to CSV.

---

## Colour rules

Use this five-step queue pressure ramp:

```text
0 #1a9850  open / low pressure
1 #91cf60
2 #fee08b
3 #fc8d59
4 #d73027  severe / high pressure
```

Queue pressure can be calculated for matched T-zones using a simple rank or quantile score based on matched project count and/or matched capacity. Make clear this applies only to matched rows.

Do not colour T3-T9 from NGET counts unless those rows are individually matched to T-zones.

---

## Safe rendering requirements

When rendering live API data into HTML:

- escape project names, customer names, connection sites, statuses, and agreement types;
- do not assign raw API strings to `innerHTML` unless escaped;
- handle null/undefined values gracefully;
- parse numeric capacity safely;
- display `—` for missing values.

Recommended helper:

```js
function esc(value){
  return String(value ?? '—')
    .replaceAll('&','&amp;')
    .replaceAll('<','&lt;')
    .replaceAll('>','&gt;')
    .replaceAll('"','&quot;')
    .replaceAll("'",'&#039;');
}
```

---

## Plain-English read-out

Below the map, include one concise read-out section:

- If no lookup is uploaded: explain that the file is in coarse mode and needs `site_zone_lookup.csv` for T1-T9 mapping.
- If lookup is uploaded: list the least congested and most congested matched T-zones.
- State how many projects and GW remain unresolved by T-zone.
- Explain the Direct Connection vs Embedded mix.
- Explain the Project Status maturity, especially share in Scoping vs Built.
- Mention that projects at or below about 100 MW may be more relevant to DNO pre-application or distribution connection routes, depending on location and technology.
- Give two next steps:
  1. For transmission-scale projects, request a pre-application or connection enquiry with NGET, SHET, or SPT as appropriate.
  2. For distribution-scale projects, request a DNO pre-application and check local GSP/headroom data.

---

## Output requirement

Create a standalone `.html` file. It should work when opened locally in a normal browser. The page must fetch live NESO data on open. If the fetch is blocked, it must show a clear error and retain the layout.

Do not provide only a screenshot or static chart. The deliverable is an interactive browser file.

