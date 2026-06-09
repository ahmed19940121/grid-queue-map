# GB Grid Connection Queue Map Skill

A reusable AI skill/prompt for creating an interactive visual map of the Great Britain electricity grid connection queue using NESO TEC Register data.

This project is designed to help users explore grid connection pressure in a more visual and practical way. It combines NESO TEC Register data, transmission owner regions, project status, agreement type, technology filtering, optional site-to-zone matching, confidence labels, and a developer-style readout into a single browser-based map view.

The aim is to make complex grid connection data easier to understand, easier to interrogate, and easier to discuss with energy developers, analysts, investors, policy teams, and grid professionals.

---

## What it does

The skill helps generate a standalone GB grid connection queue map that can show:

- NESO TEC Register queue data
- Transmission owner regions: SHET, SPT and NGET
- Indicative queue pressure across Great Britain
- Technology-specific queue views
- Project status breakdowns
- Agreement type breakdowns
- A developer-focused readout
- A searchable project table
- CSV export of visible or filtered project rows
- Optional trusted `site_zone_lookup.csv` upload for improved site, marker, and T1-T9 zone matching
- Confidence labels for mapped projects: High, Medium, Low, or Unknown

It can be used for different technology types, including:

- Battery / energy storage
- Solar PV
- Onshore wind
- Offshore wind
- Demand connections

The generated HTML should include a technology selector so users can switch between these queue views without regenerating the file.

---

## Example use cases

You can use this skill to ask questions such as:

- Where are battery storage queues most constrained?
- Which parts of Great Britain may have more grid connection opportunity?
- How does the queue look for solar, wind, storage, or demand connections?
- What is the split between direct and embedded connection projects?
- How mature is the queue based on project status?
- Which projects are driving the largest queue volumes?
- Which projects can be mapped confidently to a trusted site or transmission zone lookup?
- How much of the queue remains unresolved by T1-T9 zone?

---

## Important caveat

This is an indicative mapping tool, not a substitute for formal grid studies or engagement with NESO, transmission owners, or distribution network operators.

The NESO TEC Register does **not** provide latitude/longitude and does **not** provide a direct T1-T9 transmission zone field.

The relevant geography fields in the TEC Register are mainly:

- `Connection Site` — the best available project-level location clue
- `HOST TO` — a broad transmission owner field only

`HOST TO` should be interpreted as:

- `SHET`: Scottish Hydro Electric Transmission / north Scotland transmission area
- `SPT`: SP Transmission / central and southern Scotland transmission area
- `NGET`: National Grid Electricity Transmission / England and Wales transmission area

This means the TEC Register alone cannot precisely place projects on a map or split England and Wales into live T3-T9 counts. Any precise marker or T1-T9 allocation should only be used when `Connection Site` has been matched to a trusted reference lookup.

If no reliable site match exists, the project should remain in a coarse HOST TO bucket. Detailed regional shading for England and Wales should be treated as an indicative reference layer, not a live capacity result.

For demand connections, the skill uses the generic `Plant Type = Demand` field from the TEC Register. This should **not** be interpreted as a data-centre map or a count of data-centre connections. Demand may include power-station auxiliary load, hydrogen or electrolyser schemes, green energy hubs, and other transmission-connected demand.

---

## Data source

The main dataset used by the skill is the NESO Transmission Entry Capacity Register.

The skill is structured to query the TEC Register using the `Plant Type` field, rather than relying on project names. This is important because many projects are hybrid or multi-technology.

The skill uses the NESO Data Portal CKAN SQL API and the TEC Register resource ID:

```text
17becbab-e3e8-473f-b303-3806f43a6a10
```

The generated HTML should attempt to fetch live NESO TEC Register data when opened. If the browser blocks the fetch or the API is unavailable, the page should show a clear error and retain the layout.

---

## Technology mapping

The skill uses the TEC Register `Plant Type` field with broad matching patterns:

| Map view | TEC Register `Plant Type` match |
|---|---|
| Battery / storage | `Energy Storage System` |
| Solar PV | `PV Array` |
| Onshore wind | `Wind Onshore` |
| Offshore wind | `Wind Offshore` |
| Demand connections | `Demand` |

Because `Plant Type` can be semicolon-delimited and multi-valued, the skill should use flexible matching so that hybrid and co-located projects are captured.

---

## Optional site-zone lookup workflow

The improved skill supports an optional trusted CSV upload called:

```text
site_zone_lookup.csv
```

This allows coarse HOST TO-only data to be upgraded into more useful site markers and T1-T9 style mapping where the user has a reliable lookup.

Expected columns:

```csv
site_name,aliases,lat,lon,neso_zone,host_to,dno,voltage_kv,gen_headroom,source,confidence_base
```

Minimum useful columns:

```csv
site_name,aliases,lat,lon,neso_zone,host_to,dno,voltage_kv
```

The tool should match TEC Register `Connection Site` values against this lookup conservatively. It should only place precise project markers when the match is trusted and unambiguous.

---

## Confidence labels

Every project row should include a confidence label:

| Confidence | Meaning | Map treatment |
|---|---|---|
| High | Exact or alias match to a trusted lookup | Place marker and assign derived zone |
| Medium | Strong inferred local clue, but not exact | Optional approximate area only, clearly labelled as inferred |
| Low | HOST TO only | Do not place an exact marker; keep in SHET/SPT/NGET bucket |
| Unknown | No usable geography | Keep in table only |

The default output should be conservative. It is better to leave a project unresolved than to invent a precise location or zone.

---

## Suggested output

The ideal output is a standalone `.html` file that:

- Fetches live NESO TEC Register data when opened
- Includes a technology selector
- Shows a real Great Britain map rather than a schematic
- Uses D3.js and TopoJSON for map rendering
- Includes clear headline cards for project count, GW, HOST TO split, agreement type, and project status
- Shows only trusted matched project/site markers
- Keeps unmatched projects in coarse SHET, SPT, or NGET buckets
- Includes a visible caveat explaining TEC Register geography limitations
- Supports optional `site_zone_lookup.csv` upload
- Shows matched versus unresolved projects by count and GW
- Includes a searchable project table
- Allows export of visible or filtered table rows to CSV
- Separates live data from curated or indicative reference layers
- Makes all caveats visible to the user

---

## How to use in ChatGPT or Claude

Copy the skill/prompt into ChatGPT, Claude, or another AI assistant and ask it to generate a map.

Example prompts:

```text
Use this skill to show me the GB battery storage grid connection queue on a map.
```

```text
Show me the solar connection queue across Great Britain.
```

```text
Create a demand connections map, but make the caveat clear that this is not a data-centre count.
```

```text
Generate a standalone HTML map that fetches live NESO TEC Register data and includes a technology switcher, project table, CSV export, and optional site_zone_lookup.csv upload.
```

```text
Create an improved interactive TEC queue map for battery, solar, wind and demand connections using live NESO data. Do not invent precise project locations unless a trusted site-zone lookup is uploaded.
```

---

## Technologies

The generated map can be built using:

- HTML
- CSS
- JavaScript
- D3.js
- TopoJSON
- NESO Data Portal API

The skill itself is model-agnostic and can be used in ChatGPT, Claude, or other AI assistants that support code generation.

---

## Why I built this

Grid connection queue data is often difficult to understand in spreadsheet form. Developers, analysts, investors and energy professionals usually want to answer practical questions quickly:

> Where is the queue most constrained, and where might there still be opportunity?

This skill is an experiment in using AI to turn complex energy infrastructure data into a more accessible visual tool.

It is intended to help users move from static spreadsheet analysis toward interactive, transparent, and caveated exploration of the GB grid connection queue.

---

## Limitations

This project does not provide formal connection advice.

It does not guarantee available capacity.

It does not provide precise project coordinates from NESO data alone.

It does not create official T1-T9 zone allocations unless the user supplies a trusted lookup.

It does not replace:

- NESO connection studies
- Transmission owner engagement
- DNO pre-application checks
- Detailed network modelling
- Professional technical due diligence

Use it as a starting point for exploration, not as a final investment or connection decision tool.

---

## Licence

You may adapt and reuse this skill for learning, research, prototyping and non-commercial analysis.

Please credit the original concept if you build on it publicly.
