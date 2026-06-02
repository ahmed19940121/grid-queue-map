Grid Queue Map — Great Britain
Render the GB electricity connection queue as a real, interactive map of the whole
of Great Britain — accurate coastline, major cities for orientation, and grid
supply points (GSPs) / NESO transmission zones colour-graded by how hard it is to
connect there. This is the visual companion to grid-queue-intelligence.
The map answers the question a real grid-connection developer asks first:
"Where can I connect my project without huge reinforcement, and how long will it
take?" — modelled on the tools developers actually use (National Grid's Network
Opportunity & Development Map, GB Grid Data, e-dno lookups). See What developers
want to see below.
Efficiency contract (read this first)
This skill runs LIVE but must be frugal with requests:

One aggregate HTTP call to NESO for the map numbers. A single
datastore_search_sql query with conditional aggregation (SUM(CASE WHEN …)) returns
every headline number — including the Agreement Type and Project Status breakdowns — in
one response. Do not make a column-probe call (the columns are listed in Step 2),
do not fire one call per keyword or per status.
One optional second call for the project-name table (Step 2b). That is the only
sanctioned second request; still do not loop.
Let the map fetch itself. The NESO API returns Access-Control-Allow-Origin: *,
so a standalone HTML file can run those queries client-side on every open — the
deliverable becomes self-updating with no further work from you. Build it that way
(Step 3).
Embedded snapshot fallback. Inline/sandboxed widgets have a CSP that blocks
api.neso.energy, so the same page must fall back to an embedded snapshot and label
it honestly ("snapshot", with its date). The standalone file is the live one.
The map geometry (GB coastline) is one small CDN topology file — fixed geography,
not grid data.


What developers want to see (research-informed)
From how UK developers actually evaluate sites, the map and its side panel should
surface, in priority order:

Whole of GB, real coastline — not a schematic. Developers orient by geography.
Cities / place names — so a user can find "near Cardiff" or "East Midlands"
at a glance, and tie a site to a postcode/region.
Connection headroom, colour-graded — green = likely to connect without major
reinforcement, red = heavily constrained. This is the single most-requested layer
(mirrors the Network Opportunity Map RAG colouring).
GSP / substation markers — developers connect at a specific grid supply point,
not a zone. Show the relevant named GSPs as points.
Which DNO / network operator to apply to (SSEN, SP Energy Networks, Northern
Powergrid, Electricity North West, NGED, UK Power Networks) — so they know who to
contact. Held in locations.csv.
Distribution vs transmission — for projects <= 100 MW a DNO distribution
connection is usually faster/cheaper and sits in a separate queue. Always flag it.
Indicative timeline — realistic first-energisation window (see realism rules).
Voltage level — connection needs 33 kV+ for grid-scale; note this when relevant.
Agreement Type & Project Status — how real the queue is (early Scoping vs Built,
transmission Direct Connection vs distribution Embedded). Always show.
Named projects — the actual project + customer names behind the counts.

Surface 3-10 in compact side panels, a project table, or tooltips — not all on the
map face.

Step 1 — read the technology
Map the requested technology to a Plant Type value (see Step 2 for why). Defaults:
requestPlant Type ILIKE patternbattery (default), battery_hybrid, BESS, storage%Energy Storage System%solar%PV Array%wind_onshore%Wind Onshore%wind_offshore%Wind Offshore%demand (incl. requests for data centres, EV hubs, electrolysers)%Demand%
Plant Type is semicolon-delimited and multi-valued (e.g.
Energy Storage System;PV Array (Photo Voltaic/solar)), so an ILIKE '%…%' pattern
correctly catches co-located/hybrid projects. If unstated, default to battery and say
so in one line.

Honesty rule for demand — do NOT label this "data centres". The TEC Register
has no data-centre (or EV-charging) classification; Demand is a single generic
plant type covering any transmission-connected load. In practice the %Demand%
rows are dominated by power-station auxiliary/parasitic load (CCGT, BECCS, biomass),
~1 GW co-located "Green Energy Centre" storage+solar+demand hubs, and
hydrogen/electrolyser projects — genuine standalone data centres are not separately
identifiable here. So when a user asks for data centres, run %Demand% but title
the map and read-out "Demand connections", state up front that it is the generic
Demand plant type and not a count of data centres, and point the user to true
data-centre sources for that need: UK Power Networks Open Data ("Data Centres by Local
Authority", London/SE/East only) and the DSIT "Estimate of Data Centre Capacity: Great
Britain" publication (national, aggregated). The other rows (battery, solar,
wind_onshore, wind_offshore) are verified against the register's canonical plant
types and need no caveat.

Step 2 — fetch LIVE queue data (ONE call, conditional aggregation)
Pull fresh data from the NESO TEC Register. Resource id:
17becbab-e3e8-473f-b303-3806f43a6a10.
Verified columns (June 2026) — do not probe, they are fixed:
Project Name, Customer Name, Connection Site, Stage, MW Connected,
MW Increase / Decrease, Cumulative Total Capacity (MW), MW Effective From,
Project Status, Agreement Type, HOST TO, Plant Type, Project ID,
Project Number, Gate.

Important correction: this resource has no per-zone column (there is no
"Transmission Entry Capacity Zone" field) and the technology is in Plant Type,
not in the project name — do not text-search Project Name for "battery". The only
geographic field is HOST TO, the transmission owner, with three values:
SHET (north Scotland → zone T1), SPT (central & south Scotland → zone T2),
NGET (England & Wales → zones T3–T9).

The single query — fire exactly this (URL-encode the sql param; the $$…$$
dollar-quoting avoids escaping inner quotes), substituting the Step-1 pattern. It now
always includes Agreement Type and Project Status breakdowns (free — same call):
GET https://api.neso.energy/api/3/action/datastore_search_sql?sql=
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
  WHERE "Plant Type" ILIKE $$%Energy Storage System%$$
result.records[0] returns every headline number in one round-trip (~1.5 s). (Verified
1 Jun 2026 for battery: total 1497, gw 466.0, nget 998, spt 290, shet 209.)
Verified value sets (June 2026): Agreement Type = Direct Connection, Embedded
(Embedded = connects via the distribution network, not directly to transmission).
Project Status = Scoping, Awaiting Consents, Consents Approved, Built. Render
both as small breakdown cards in the side panel. (1 Jun 2026 demand: Direct 115 /
Embedded 2; Scoping 79 / Awaiting Consents 30 / Consents Approved 3 / Built 5 — i.e. the
queue is overwhelmingly early-stage.)
Turn live counts into colours. Colour ramp (0 = most open, 4 = most contested):
0 #1a9850 / 1 #91cf60 / 2 #fee08b / 3 #fc8d59 / 4 #d73027.
Because the register only resolves geography to the three transmission owners, derive
queue_score honestly:

T1 from the live SHET count, T2 from the live SPT count (rank/scale them).
T3–T9 all sit under one NGET total, so the register cannot split them. Grade these
English/Welsh zones using the curated network-constraint reference in
assets/locations.csv (gen_headroom: open→severe ⇒ 0→4). State this split clearly
in the caption — it is the truthful description of what the live data can and cannot
resolve.

Snapshot fallback. If the live query is blocked (sandboxed widget) or errors, render
from an embedded snapshot of the figures above and label it "snapshot (date)". The
standalone file should always attempt the live fetch first.
assets/locations.csv holds cities + key GSPs with lat,lon,neso_zone,dno,gen_headroom
for orientation, marker placement and the T3–T9 grading above.
Step 2b — fetch the project list (one optional second call → names table)
The aggregate above cannot name projects. To populate a names table, fire ONE more call
pulling the records (the only sanctioned second request):
SELECT "Project Name" pn,"Customer Name" cn,"HOST TO" h,"Project Status" ps,
       "Agreement Type" agr,"Cumulative Total Capacity (MW)" cap
FROM "17becbab-e3e8-473f-b303-3806f43a6a10"
WHERE "Plant Type" ILIKE $$%Demand%$$
ORDER BY "Cumulative Total Capacity (MW)" DESC

SQL gotcha: never alias a column at — AT is a reserved word and the query
returns HTTP 409 (syntax error). Use agr (or any non-reserved alias). Also
ORDER BY the full column name, not the alias.

Render the result as a scrollable, client-side-filterable table (Project, Customer,
Region = HOST TO, Status, Agreement, MW), sorted by capacity. Embed a snapshot of the
rows so sandboxed previews still show the table, and have the standalone file re-fetch
them live. Colour the Status cell (Built green → Scoping grey).
Step 3 — render the real GB map (D3, self-fetching)
Never hand-draw the coastline. Fetch real GB topology and render with D3. Produce a
standalone HTML file (so the user can reopen it and it re-fetches live) — and,
when an inline preview is useful, also an inline widget that uses the snapshot. Only
D3 + topojson from the CDN; inline everything else.
html<div id="map" style="width:100%"></div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.8.5/d3.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/topojson/3.0.2/topojson.min.js"></script>
<script>
const RID='17becbab-e3e8-473f-b303-3806f43a6a10';
const LIKE='Energy Storage System';                 // from Step 1
const SNAPSHOT={total:1497,shet:209,spt:290,nget:998,gw:466.0,ts:'snapshot'};
const SQL=`SELECT COUNT(*) total,`+
 `SUM(CASE WHEN "HOST TO"=$$SHET$$ THEN 1 ELSE 0 END) shet,`+
 `SUM(CASE WHEN "HOST TO"=$$SPT$$ THEN 1 ELSE 0 END) spt,`+
 `SUM(CASE WHEN "HOST TO"=$$NGET$$ THEN 1 ELSE 0 END) nget,`+
 `ROUND(SUM("Cumulative Total Capacity (MW)")/1000.0,1) gw `+
 // + the ag_* and st_* SUM(CASE WHEN ...) columns from Step 2
 `FROM "${RID}" WHERE "Plant Type" ILIKE $$%${LIKE}%$$`;
const URL='https://api.neso.energy/api/3/action/datastore_search_sql?sql='+encodeURIComponent(SQL);

function draw(D, live){
  d3.json('https://cdn.jsdelivr.net/npm/datamaps@0.5.10/src/js/data/gbr.topo.json').then(gb=>{
    const feats = topojson.feature(gb, gb.objects.gbr);             // object key "gbr", properties.name = county
    const proj = d3.geoMercator().fitSize([520,760], feats);
    const path = d3.geoPath(proj);
    const svg = d3.select('#map').append('svg').attr('viewBox','0 0 560 800').attr('width','100%');
    svg.append('g').selectAll('path').data(feats.features).join('path')
       .attr('d',path).attr('fill','#dfe6ec').attr('stroke','#fff').attr('stroke-width',0.4);
    // cities + GSP markers placed by real lat/lon via proj([lon,lat]); colour by queue_score (Step 2).
    // T1 score from D.shet, T2 from D.spt, T3-T9 from locations.csv gen_headroom. Caption: live vs snapshot.
  });
}
fetch(URL).then(r=>r.json()).then(j=>{ if(!j.success) throw 0; draw(j.result.records[0], true); })
          .catch(()=>draw(SNAPSHOT, false));
</script>
Markers: const [x,y] = proj([lon,lat]); Colour each marker with the ramp value for its
zone's queue_score from Step 2. Cities get a small dot + label for orientation; GSPs
get a larger ringed marker and a click/hover that fills the side panel (zone, live
status, DNO, distribution flag, indicative timeline). Keep labels legible (dark text,
white halo if overlapping land).
Add a legend (5-colour ramp open->severe), a status badge / caption stating
whether the figures are live (with timestamp) or snapshot, the honest "live counts
resolve to transmission owner; Scotland set from SHET/SPT, England & Wales graded by
reference" note, a side panel carrying the developer fields (#3-#8 above), the
Agreement Type and Project Status breakdown cards (#9), and the projects table
(#10, Step 2b).
Step 4 — plain-English read-out (below the map)
Name the 2-3 greenest zones and 2-3 reddest for the technology, one line each, then
the <=100 MW distribution flag if relevant, then 2 concrete next steps (e.g. "Submit a
Grid Connection Enquiry to NGET for a pre-application study"; "Request a
pre-application from your local DNO"). Add one line on queue maturity from the status
breakdown (e.g. "X% still in Scoping, only Y Built") and one on agreement mix (Direct
Connection vs Embedded).
State the coverage caveats honestly: (a) Plant Type is the technology field and is
multi-valued, so hybrids are counted via ILIKE; (b) the register resolves geography
only to transmission owner, so English/Welsh zone shading is a curated reference, not a
live per-zone count; (c) for a demand request, add the Step-1 honesty rule — it is the
generic Demand plant type, not a data-centre count.
