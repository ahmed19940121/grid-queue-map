# GB Grid Connection Queue Map Skill

A reusable AI skill/prompt for creating a visual map of the Great Britain electricity grid connection queue.

This project is designed to help users explore grid connection pressure in a more visual and practical way. It combines NESO TEC Register data, transmission regions, GSP-style markers, project status, agreement type, and a developer-style readout into a single map view.

The aim is to make complex grid connection data easier to understand and easier to discuss.

## What it does

The skill helps generate a GB grid connection queue map that can show:

* NESO TEC Register queue data
* Transmission owner regions: SHET, SPT and NGET
* Indicative queue pressure across GB
* GSP / substation-style markers
* Project status breakdowns
* Agreement type breakdowns
* A developer-focused readout
* A project table showing relevant projects where available

It can be used for different technology types, including:

* Battery / energy storage
* Solar
* Onshore wind
* Offshore wind
* Demand connections

## Example use cases

You can use this skill to ask questions such as:

* Where are battery storage queues most constrained?
* Which parts of GB may have more grid connection opportunity?
* How does the queue look for solar, wind or demand connections?
* What is the split between direct and embedded connection projects?
* How mature is the queue based on project status?
* Which projects are driving the largest queue volumes?

## Important caveat

This is an indicative mapping tool, not a substitute for formal grid studies or engagement with NESO, transmission owners, or distribution network operators.

The NESO TEC Register resolves geography mainly by transmission owner:

* SHET: north Scotland
* SPT: central and south Scotland
* NGET: England and Wales

This means England and Wales cannot be split into live per-zone counts from the TEC Register alone. Any detailed regional shading for England and Wales should be treated as an indicative reference layer, not a live capacity result.

For demand connections, the skill uses the generic `Plant Type = Demand` field from the TEC Register. This should not be interpreted as a data-centre map or a count of data-centre connections.

## How to use in ChatGPT or Claude

Copy the skill/prompt into ChatGPT or Claude and ask it to generate a map.

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
Generate a standalone HTML map that fetches live NESO TEC Register data and includes a snapshot fallback.
```

## Suggested output

The ideal output is a standalone HTML file that:

* Attempts to fetch live NESO TEC Register data when opened
* Falls back to an embedded snapshot if live data is blocked
* Shows a real GB map rather than a schematic
* Includes a clear legend and data-source notes
* Separates live data from curated or indicative reference layers
* Includes project status and agreement type breakdowns
* Makes all caveats visible to the user

## Data source

The main dataset used by the skill is the NESO Transmission Entry Capacity Register.

The skill is structured to query the TEC Register using the `Plant Type` field, rather than relying on project names. This is important because many projects are hybrid or multi-technology.

## Technologies

The generated map can be built using:

* HTML
* JavaScript
* D3.js
* TopoJSON
* NESO Data Portal API

The skill itself is model-agnostic and can be used in ChatGPT, Claude, or other AI assistants that support code generation.

## Why I built this

Grid connection queue data is often difficult to understand in spreadsheet form. Developers, analysts and energy professionals usually want to answer practical questions quickly:

> Where is the queue most constrained, and where might there still be opportunity?

This skill is an experiment in using AI to turn complex energy infrastructure data into a more accessible visual tool.

## Limitations

This project does not provide formal connection advice.

It does not guarantee available capacity.

It does not replace:

* NESO connection studies
* Transmission owner engagement
* DNO pre-application checks
* Detailed network modelling
* Professional technical due diligence

Use it as a starting point for exploration, not as a final investment or connection decision tool.

## Licence

You may adapt and reuse this skill for learning, research, prototyping and non-commercial analysis.

Please credit the original concept if you build on it publicly.
