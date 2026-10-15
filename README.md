# CRN Field Sheets — editing workspace

This is a working copy of the field sheets for the Cascade Climate Bedrock Initiative
Coordinated Research Network (CRN). The sheets are self-contained HTML files — each one is
a data-entry form on screen and a paper field sheet under print, both from the same page.

## Purpose

The field sheets are built by a generator in the main data project (`crn_data`). This
directory exists so people can review and edit the **semi-final** sheets — adjusting labels,
hints, section order, and wording — without risking the larger project's build pipeline,
validator, or schemas.

Changes made here are arrangement and wording only. Structural changes (field names,
controlled vocabularies, export bindings) belong in the main project.

## What is in `sheets/`

Fifteen HTML files, one per visit type:

| Sheet | What the crew does |
|-------|--------------------|
| V1 | Soil coring visit — baseline and seasonal |
| V2 | Feedstock lot receipt — incoming rock characterization |
| V3 | Equipment installation — lysimeters, resins, gas wells |
| V4 | Soil-gas grab round — biweekly CO₂/CH₄/N₂O profile |
| V5 | Aqueous collection — porewater sampling and splitting |
| V6 | Integrating collector round — resin/tea-bag retrieval |
| V7 | Tracer test — dye or isotope injection and monitoring |
| V8 | Feedstock application — rock spreading event |
| V9 | Biomass harvest — aboveground plant tissue sampling |
| V10 | Monolith extraction — intact soil column removal |
| V11 | Device installation — sensor and logger emplacement |
| V12 | Agricultural amendments — fertilizer, lime, tillage |
| V13 | Plot delineation — initial plot geometry and monuments |
| V14 | FIZ soil coring — feedstock incorporation zone push cores |
| V15 | AIZ soil augering — agronomic + deep zone bucket auger |

Open any `.html` file in a browser to see the digital form. Print it (or save as PDF) to
see the paper version.

## Using Claude to edit

This project includes a `CLAUDE.md` that tells Claude Code which parts of the HTML are safe
to change and which parts are load-bearing for the data export. Key points:

- **Labels, hints, section order, and CSS are safe.** These are the things worth editing here.
- **`data-field`, `value`, `data-key`, `data-transform`, and the `__SHEET__` block are not.**
  These control what the exported CSV contains. Changing them here silently forks the sheet
  from the metadata standard.
- Every edit is logged in `CHANGELOG.md` so it can be replayed into the main project later.
- Edited sheets get an `(edited)` marker in their version stamp so they are never mistaken
  for clean build output.

## Merging changes back

When edits are ready to go back into the main project:

1. Read `CHANGELOG.md` for the list of changes.
2. Apply each change to the corresponding layout file in `crn_data/sheets/layouts/*.yml`.
3. Rebuild with `python sheets/build.py all` in the main project.
4. Run `crn-standard check-sheets` to verify nothing broke.

The HTML files in this directory are never copied back directly — the build always regenerates
from layouts.
