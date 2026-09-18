# CRN Field Sheets — editing workspace

This directory holds **copies** of the CRN field sheets for review and editing. The
authoritative source — schemas, build tooling, dispositions, the validator — lives at
`C:\Users\nic\crn_data`. Changes made here must eventually be carried back there by a human;
nothing in this directory feeds the build pipeline or the validator.

## What you are looking at

Each `.html` file in `sheets/` is a **self-contained CRN field sheet** — a data-entry form
that is also a printable paper sheet (same DOM, `@media print` re-lays it out). A sheet
represents a **site visit**, not one activity: visit metadata at the top, then one or more
activity modules, then deviations and sign-off. On export, the form transcribes to per-table
CSVs inside a ZIP with no human deciding anything.

There are **thirteen sheets** (V1–V13). Each one covers a specific kind of visit:

| ID | Title | Streams |
|----|-------|---------|
| V1 | Site visit (soil coring) | soil |
| V2 | Feedstock lot receipt | feedstock |
| V3 | Equipment installation visit | device, porewater, collector |
| V4 | Soil-gas grab round | device |
| V5 | Aqueous collection and splitting | porewater |
| V6 | Integrating collector round | collector |
| V7 | Tracer test | tracer |
| V8 | Feedstock application | management |
| V9 | Biomass harvest visit | biomass |
| V10 | Monolith extraction | porewater |
| V11 | Device installation | device |
| V12 | Agricultural amendments | management |
| V13 | Plot delineation | site |

If `sheets/index.html` is present, it is a landing page linking to each sheet.

If `layouts/` and `build.py` are also present, edits go to the layout YAML and the HTML is
regenerated — never edit both. The HTML files are build output in that case.

## Scope of edits — what you can and cannot change here

### Safe to change (arrangement and wording)

- **Labels** — the human-readable text next to a box (e.g. "Arrived" → "Arrival time")
- **Hints and scope text** — the grey instruction text below section headers
- **Section order** — rearranging sections within a sheet (but **never rename a `<section
  id="...">`** — the export contract binds tables to sections by that id; renaming it silently
  drops every field in that section from the export)
- **Footer text** — the instruction text at the bottom
- **Print layout CSS** — `@media print` rules, page breaks, column widths
- **Screen-only CSS** — colors, spacing, font sizes for the digital form
- **Controlled-value labels** — the displayed text of a radio/select option, when the
  underlying `value` attribute stays the same (e.g. displaying "Consumed on site" while the
  value remains `consumed_in_field`)

### Stop and report — do NOT change without coordination

These are decisions owned by the metadata standard in `C:\Users\nic\crn_data`. Changing them
here silently forks the sheet from the standard, and the next build over there will overwrite
the change with no trace.

- **`data-field` attributes** — these are the column names in the exported CSV. Renaming one
  breaks the export and every downstream validator.
- **`value` attributes on radios/selects** — these are controlled vocabulary terms registered
  in `standard/vocab/*.csv`. Changing a value here means the export writes a term the standard
  does not recognize.
- **`data-key` attributes on cards** — these carry pre-filled column values as JSON. Changing
  them alters the exported data.
- **`data-transform` attributes** — these declare unit conversions (e.g. `cm_to_m`,
  `hpa_to_kpa`). Removing or changing one silently alters the stored value.
- **`data-mdl-for` attributes** — these wire `< MDL` checkboxes to their paired numeric field.
  Rewiring one means censored values write to the wrong column.
- **`data-broadcast` attributes** — these stamp a field's value onto every row in a table.
  Changing the target table breaks the export.
- **`name` attributes on inputs** — `data-mdl-for` and `data-conv-for` reference inputs by
  name; autosave serializes on `el.name`. Renaming one silently unwires the `< MDL` tick or
  the unit-conversion readout.
- **`data-row` and `data-section` on cards** — the export routes rows through these; the
  photo store keys off them. Changing either silently drops rows or misroutes photos.
- **`required` attributes** — adding or removing `required` changes whether the export button
  will fire. This is a data-quality decision, not a layout decision.
- **The inline `<script>` runtime** — it is generated from a single source
  (`crn_data/sheets/runtime.js`) and duplicated into every sheet. Editing it in one sheet
  diverges that sheet's behavior from all others. Never edit the runtime here.
- **The `window.__SHEET__` JSON block** — this is the export contract. It names every table,
  section binding, column order, and the absent-value token. Editing it by hand is almost
  certainly wrong.
- **Adding or removing a `data-field` input** — this adds or removes a column from the
  exported CSV. If the field exists in the standard, it needs a disposition assignment. If it
  does not exist, it needs to be admitted to the standard first (admission log, axis-first
  lookup, the whole process).
- **The version stamp** in the masthead (e.g. "Sheet V4 v1.0.0 / Standard 0.3.0-dev"). The
  stamp exists so a stale printed sheet self-identifies. See the edit ledger rule below.

If you encounter a request that falls into this category, say so clearly: *"This changes a
field name / controlled vocabulary term / export binding. That decision belongs to the metadata
standard at `C:\Users\nic\crn_data` — I can describe what needs to change there, but I should
not make the change in this copy."*

## How to find a field

To locate where a specific field appears in a sheet:

1. **By field name** — search for `data-field="<name>"` in the HTML. Every input, select, and
   textarea that contributes to the export carries this attribute. Example:
   `data-field="bicrn_soil_air_pco2"` is the CO₂ reading box.
2. **By label** — search for the human-readable label text. Labels are `<label>` elements
   immediately before the input's wrapper `<div>`.
3. **By section** — sections are `<section id="...">` elements. The `id` matches the section
   name in the layout (e.g. `id="conditions"`, `id="wells"`, `id="signoff"`).
4. **By table binding** — look in the `__SHEET__` JSON at the bottom. `row_tables` maps each
   repeating section to its target table; `singletons` maps field-group sections to theirs.

## DOM structure — the load-bearing parts

```
<main class="sheet">
  <header class="masthead">         ← title, scope, version stamp
  <div class="context">             ← pre-printed masthead (site, date, campaign, plot)
  <section id="...">                ← one per activity block
    <h2>                            ← section title
    <p class="hint">                ← instruction text (grey, optional)
    <div class="fieldrow">          ← one per field (label + input)
    <article class="card" data-row="N" data-section="..." data-key="{...}">
                                    ← one per repeating record (e.g. one soil core, one well)
  <section id="signoff">            ← always last content section; two-slot sign-off
  <div class="actions screen-only"> ← Export CSV / Print buttons
  <footer>                          ← retention instruction + build provenance
</main>
<script>window.__SHEET__ = {...}</script>  ← export contract (DO NOT HAND-EDIT)
<script>...</script>                       ← runtime (inline, same on every sheet)
```

**Key CSS classes:**
- `.screen-only` — hidden when printed; `.print-only` — hidden on screen
- `.pre` on a `<td>` — pre-printed (greyed, not editable)
- `.card` — a repeating record card; `.cardhead` — its pre-printed identity header
- `.ticks` — radio/checkbox group rendered as tick boxes
- `.unit` — the fixed unit label next to a numeric input (never an editable field)
- `.conv` — the live-converted readout (e.g. "= 0.1 m" next to a cm input)
- `.mdl` — the `< MDL` checkbox that disables its paired numeric

## Edit ledger — making changes durable

Every edit must be recorded in `CHANGELOG.md` (create it on first edit). Each entry is one
line:

```
YYYY-MM-DD | V<N> | <section or element> | <what changed> | <why>
```

Example:
```
2026-09-18 | V4 | section#bracket hint | Reworded to clarify span gas procedure | Site manager feedback
2026-09-18 | V1 | .masthead h1 | "Site visit" → "Soil coring visit" | Distinguish from V3/V11
```

This ledger exists so that changes can be **replayed** into the layout files at
`C:\Users\nic\crn_data\sheets\layouts/` when it is time to merge back. Without it, a change
made here is a one-way fork.

**Version stamp rule:** when you edit a sheet, append `(edited)` to the **visible masthead
text only** — the `<div class="stamp">` element. For example, `Sheet V4 v1.0.0` becomes
`Sheet V4 v1.0.0 (edited)`. Do **not** change `sheet_version` inside the `__SHEET__` JSON
block — the autosave restore gates on an exact match, and bumping it means a technician's
in-progress work will not restore. The `(edited)` marker is a human-readable signal on
printed copies, not a data value.

## The invariants — rules that must hold on every sheet

These are the error-reduction design decisions. They are not conventions; violating any one of
them reintroduces a class of error the sheets were built to prevent.

1. **Units are printed, never typed.** A numeric input carries its unit as fixed `<span
   class="unit">` text. There is no input a technician can type a unit into.
2. **Depth is entered in cm, stored in m.** A `data-transform="cm_to_m"` attribute on the
   input handles the conversion; the `.conv` span shows the converted value live. An
   undeclared unit conversion is a defect (CRN-FSH09) — the cm/m confusion in G10 was a real
   error that shipped.
3. **A censored value gets a checkbox, not a `<`.** Any field paired with a `_flag` column has
   a `< MDL` tick (`data-mdl-for`). Ticking it disables the numeric input and writes
   `below_detection` to the flag column, keeping the numeric column absent. "Numeric columns
   hold numerics" dies the moment someone writes `<0.05`.
4. **No IGSN is ever hand-copied.** IGSNs are pre-printed labels. The technician sticks the
   label on the bag. A single wrong character silently detaches a sample from its chemistry.
5. **Two-slot sign-off on every sheet.** Two different people must sign. The runtime warns (but
   does not block) when the same person ID appears in both slots.
6. **A deviation box on every sheet.** If anything departed from the SOP, there is always a
   place to say so.
7. **Never ask for a derivable quantity.** If it can be computed from other recorded fields,
   compute it. Asking a technician to do arithmetic in a field notebook is how two values of
   the same quantity come to disagree.
8. **Screen and print are the same DOM.** `@media print` re-lays it out but never selects a
   different field set. If the two diverge, the paper copy drifts — and the paper copy is used
   precisely when nobody can notice.

## Git

This directory should be a git repository. If it is not yet, run `git init` before making
edits, so every change is recoverable.

Commit after each logically complete edit. A good commit message names the sheet, the section,
and what changed — e.g. `V4: reword bracket hint for clarity`.
