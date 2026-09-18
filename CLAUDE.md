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

## Critical: the HTML is minified

**The entire body of each sheet — every section, card, template, and field — is on a single
very long line** (typically line 222). The `<style>` block occupies lines 8–220, then
`</head><body>` on line 221, then the entire `<main class="sheet">...</main>` on one line,
then the `__SHEET__` JSON on the next line, then the runtime `<script>` starts on the line
after that.

This has two consequences for editing:

1. **You must use short, unique substrings as `old_string` in the Edit tool.** You cannot
   match "the whole line" — it is 50–200 KB long. Instead, match just enough surrounding
   context to be unique. Always include an adjacent attribute or tag boundary.

2. **Use `replace_all: true` for label changes.** A label like `>Taken by</label>` appears
   once in the `<template>` and once in each pre-printed print-only card (typically 3). Using
   `replace_all` changes all occurrences. If the string is not unique to the target field
   (e.g. `>Plot</label>` appears in multiple sections), include more context — the preceding
   `name=` or `data-field=` attribute — to disambiguate.

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
  <header class="masthead">         <- title, scope, version stamp
  <div class="context">             <- pre-printed masthead (site, date, campaign, plot)
  <section id="...">                <- one per activity block
    <h2>                            <- section title
    <p class="hint">                <- instruction text (grey, optional)
    <div class="fieldrow">          <- one per field (label + input)
    <article class="card" data-row="N" data-section="..." data-key="{...}">
                                    <- one per repeating record (e.g. one soil core, one well)
  <section id="signoff">            <- always last content section; two-slot sign-off
  <div class="actions screen-only"> <- Export CSV / Print buttons
  <footer>                          <- retention instruction + build provenance
</main>
<script>window.__SHEET__ = {...}</script>  <- export contract (DO NOT HAND-EDIT)
<script>...</script>                       <- runtime (inline, same on every sheet)
```

**Key CSS classes:**
- `.screen-only` — hidden when printed; `.print-only` — hidden on screen
- `.pre` on a `<td>` — pre-printed (greyed, not editable)
- `.card` — a repeating record card; `.cardhead` — its pre-printed identity header
- `.ticks` — radio/checkbox group rendered as tick boxes
- `.unit` — the fixed unit label next to a numeric input (never an editable field)
- `.conv` — the live-converted readout (e.g. "= 0.1 m" next to a cm input)
- `.mdl` — the `< MDL` checkbox that disables its paired numeric
- `.photo-attach` — inline photo attachment block (V11; see photo-attach section below)

### Repeating elements: templates and print-only cards

Every repeating section (cards you can add) has this structure:

```
<template id="tmpl-SECTIONID">
  <article class="card" data-row="__N__" data-section="SECTIONID" ...>
    ...fields with __N__ in name attributes...
  </article>
</template>
<button class="add-card-btn screen-only" data-add-section="SECTIONID">+ Add ...</button>
<article class="card print-only" data-skip-if-empty>...fields with cP0 prefix...</article>
<article class="card print-only" data-skip-if-empty>...fields with cP1 prefix...</article>
<article class="card print-only" data-skip-if-empty>...fields with cP2 prefix...</article>
```

- The `<template>` is what the "+ Add" button clones on screen. Its field names use `__N__`
  as a placeholder that gets replaced with a counter.
- The `print-only` cards are blank cards visible only when printed (paper backup). Their field
  names use `cP0`, `cP1`, `cP2` prefixes.
- **Both template and print-only cards have identical field labels and structure.** Any label
  or hint change must hit both. Using `replace_all: true` handles this automatically when the
  search string is unique.

## Precise edit recipes

These are the exact HTML patterns for each kind of safe edit. Every example uses the Edit tool
with `replace_all: true` unless stated otherwise.

### 1. Change a field label

The pattern is: `>OLD LABEL</label>` or `>OLD LABEL<span class="req"` (if the field is required).

**Simple label (not required):**
```
old_string: >Siting notes</label>
new_string: >Installation notes</label>
replace_all: true
```

**Required field label (has a `*` indicator):**
```
old_string: >Device ID<span class="req"
new_string: >Instrument ID<span class="req"
replace_all: true
```

**Ambiguous label** (e.g. "Plot" appears in multiple sections): include the preceding
`data-field` attribute to disambiguate:
```
old_string: data-field="bicrn_plot_id"></textarea></div></div><div class="fieldrow"><label for="devices
new_string: (don't change this — it's load-bearing)
```
Instead, include enough of the fieldrow to be unique:
```
old_string: bicrn_replaces_device_id">Replaces (if a swap)</label>
new_string: bicrn_replaces_device_id">Replaces (swap only)</label>
replace_all: true
```

### 2. Change a hint (grey instruction text below a section heading)

```
old_string: <p class="hint">One card per device. Record the device ID, type, and plot.
new_string: <p class="hint">One card per installed device. Record the device ID, type, and plot.
```
Hints appear once per section, so `replace_all` is not needed.

### 3. Change a section title

```
old_string: <h2>Devices installed</h2>
new_string: <h2>Instruments installed</h2>
```
Each `<h2>` appears once. No `replace_all` needed.

### 4. Change the scope text (masthead subtitle)

```
old_string: <p class="scope">One sheet per installation day. Record what you put in the ground
new_string: <p class="scope">One sheet per installation visit. Record what you put in the ground
```

### 5. Change footer text

```
old_string: <footer>Retain this sheet. Scan the signed original
new_string: <footer>Keep this sheet. Scan the signed original
```

### 6. Change a radio/select option's display text

The pattern is: `value="TERM"> Display text</label>`. Change only the display text, never
the `value`:

```
old_string: value="soil_moisture_probe"> Soil-moisture probe</label>
new_string: value="soil_moisture_probe"> Soil moisture probe</label>
replace_all: true
```

### 7. Change the sheet title (H1)

```
old_string: <h1>Device installation</h1>
new_string: <h1>Instrument installation</h1>
```
Also update the `<title>` tag near line 6:
```
old_string: <title>V11 - Device installation</title>
new_string: <title>V11 - Instrument installation</title>
```

### 8. Change help text on a field (tooltip or inline help)

Help text appears as `title="..."` on inputs or as `<small>` within table cells. Search for
the field name and its surrounding context.

## Photo-attach system (V11)

V11's "Devices installed" section has an **inline photo attachment** feature. Each device card
contains a nested photo block that lets technicians attach a photo without switching to the
dedicated Photographs section.

The structure inside each device card (both template and print-only):
```
<div class="photo-attach screen-only" data-photo-attach>
  <button class="photo-attach-toggle" data-photo-toggle="...">Attach photo</button>
  <div class="photo-attach-area" id="pa_..." hidden>
    <article class="card photo-inline" data-section="photos_inline_devices" ...>
      <div class="drop-zone">Drop a photo here</div>
      ...Photo ID, Taken by, Notes fields...
      ...hidden inputs for subject_type, subject_id, etc...
    </article>
  </div>
</div>
```

Key points:
- The photo-attach block is `screen-only` — it does not appear on printed sheets (print-only
  cards have no photo-attach block).
- `data-subject-from="bicrn_device_id"` on the inner card tells the runtime to inherit the
  device ID as the photo's subject ID during export.
- Subject type is pre-filled as `instrument_installation` via a hidden input.
- Because `instrument_installation` has `requires_geometry: FALSE`, the inline block omits
  geometry fields (orientation, mount, camera height, scale reference) — those are hidden
  inputs instead, exported as the absent token.
- The dedicated Photographs section still exists at the bottom for photos that don't belong
  to a specific device card.

**Do not edit the photo-attach block's data attributes, hidden inputs, or section bindings.**
The button text ("Attach photo"), drop zone text, and visible field labels (Photo ID, Taken by,
Notes) are safe to change with the same label-edit pattern above.

## Edit ledger — making changes durable

**Every edit to a sheet must be recorded in `CHANGELOG.md`.** This is not optional. The
changelog is the only mechanism for carrying changes back to the main project — without an
entry, a change made here will be silently overwritten by the next build.

Update the changelog **in the same response** as the edit, not as a separate step. If you
make an edit and forget the changelog entry, add it immediately when you notice.

### Format

Entries go under a date heading, newest first:

```markdown
### YYYY-MM-DD

- **V4** `section#bracket .hint` — Reworded to clarify span gas procedure: "Fill this block
  TWICE" → "Record TWO readings — one before and one after the round".
  *Why: site manager feedback that "TWICE" was ambiguous.*

- **V1** `.masthead h1` — "Site visit" → "Soil coring visit".
  *Why: distinguish from V3/V11 which are also site visits.*
```

### What to include in each entry

1. **Which sheet** — the V-number.
2. **Where in the sheet** — section id, CSS selector, or element description, specific enough
   that someone reading this can find the spot in the layout YAML.
3. **What changed** — the old text/value and the new text/value. For renames, use `→`. For
   rewrites, quote enough of the old and new to be unambiguous.
4. **Why** — the motivation. "Site manager feedback", "reviewer request", "consistency with
   V3" — whatever prompted the change. This is what the person merging back needs to judge
   whether the change still applies.

When a single edit touches the same element across multiple sheets, list every affected sheet
in one entry rather than repeating the entry thirteen times.

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

## Checklist for every edit

1. **Read** the file before editing (Edit tool requirement).
2. **Grep** for the string you plan to change to confirm how many times it appears and in
   which contexts. If it appears in both template and print-only cards, use `replace_all`.
3. **Edit** with a short, unique `old_string`. Include an adjacent attribute or tag to
   anchor it. Never try to match a whole line — the body is one line.
4. **Grep** again after editing to confirm the change landed everywhere and nothing was
   missed or broken.
5. **Record** the change in `CHANGELOG.md` — same response, not later. Include the sheet,
   location, old → new text, and why. See the edit ledger section for the format.
6. **Mark** the version stamp with `(edited)` if this is the first edit to this sheet.

## Git

This directory should be a git repository. If it is not yet, run `git init` before making
edits, so every change is recoverable.

Commit after each logically complete edit. A good commit message names the sheet, the section,
and what changed — e.g. `V4: reword bracket hint for clarity`.
