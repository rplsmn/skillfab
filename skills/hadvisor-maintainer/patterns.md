# Hadvisor Patterns & Conventions

## Dynamic Module Insertion (Accordion Pattern)

Sequences (HAD) and RUMs (MCO) are added/removed dynamically at runtime. The pattern:

**Trigger chain:**
1. User changes sidebar `numericInput("n_rum")` or clicks `actionBttn("add_n_rum")`
2. `app_server.R` observer forwards to `state$mco$patient$n_rum()` or `add_n_rum()`
3. `mod_mco_patients_server` observer (priority 9999) fires:
   - Computes delta between desired count and `n_rum_current()`
   - If adding: `bslib::accordion_panel_insert()` + `mod_mco_rum_inputs_server(newname, state)`
   - If removing: `bslib::accordion_panel_remove()`
   - Updates `rumnames()` list and `n_rum_current()`

**Initial values propagation:** When adding a new unit, the previous unit's inputs are read and passed as `init_*` arguments to the new module's UI. This gives sensible defaults (same age, same type_rum, etc.).

**Module naming:** `"rum_inputs_1"`, `"rum_inputs_2"`, etc. Constructed with `paste0("rum_inputs_", i)`. Used as keys in `state$patient$rum_inputs`.

## Copy/Paste Between Units

Each sector has a clipboard slot (`state$patient$copy()`):

1. **Copy:** User clicks copy button → observer saves current unit's inputs as a named list to `copy()`
2. **Paste:** User clicks paste button on another unit → observer reads `copy()` and calls `update_all_*_inputs(session, inputs_copy)` to sync UI widgets

The updater functions (`utils_ui_updaters*.R`) handle widget-specific update calls (`updateVirtualSelect`, `updateNumericInput`, etc.).

## groupeR Data Assembly

Each sector has a set of `make_*_tbl_*()` builder functions and one master assembler.

**HAD:** 4 tables assembled in `mod_patients_server`:
```r
list(fixe, fixe_sej, diag, acte)
```

**MCO:** 6 tables assembled via `assemble_mco_grouper_data()`:
```r
list(fixe, fixe_2, diag, acte, um, autor_pgv)
```

**Rules for builders:**
- Each function is pure (takes inputs, returns tibble)
- Column names and types must match groupeR expectations exactly (use `grouper-consumer` skill)
- `ident` column links all tables for a stay (12-char random from `make_random_ident()`)
- Diagnosis `typ_diag` codes are sector-specific
- RUM/sequence indices are zero-padded: `sprintf("%02d", i)`

## CSS Conventions

**Prefix:** `grp-` (short for "grouper")
**Pattern:** `grp-<block>--<modifier>` (BEM-lite)
**Single file:** `inst/app/www/css/monzhad.css`
**Auto-loaded** by golem's `bundle_resources()`

**Blocks:** `card`, `badge`, `label`, `img`, `btn`, `radio`, `switch`
**Modifiers:** sector names (`had`, `mco`), semantic (`success`, `warning`, `sevlourd`), size (`small`), shape (`rounded`), action (`copy`, `paste`)

**Rules:**
1. No inline `style=` for static CSS — use a `grp-*` class
2. Leave `style = "unite"` untouched — it's a shinyWidgets argument, not CSS
3. Group classes by block with comment headers in monzhad.css
4. New sectors get `grp-badge--<sector>` following existing pattern

## Input Choice Vectors

Dropdown choices (virtualSelectInput) come from bundled data:

- `cim_input` — ICD-10 codes (from `la_cim` referential)
- `mpp_input`, `mpa_input` — HAD-specific mode codes
- `um_input` — Medical unit types
- `mode_entree_input`, `mode_sortie_input` — Entry/exit modes

Built in `data-raw/sources_default.R`, stored in `data/sysdata.rda`. Use `shinyWidgets::prepare_choices()` for virtualSelectInput compatibility.

## Experimental Severity Pipeline (MCO)

`utils_severite_mult.R` implements the v2025+ multi-dimension severity algorithm as **pure functions** (no Shiny reactivity):

```
sev_mult_pipeline()
  1. sev_mult_enrich_das()         → join DAS with prio/dimension tables
  2. sev_mult_exclusion_dasdas()   → apply DAS-DAS exclusion pairs
  3. sev_mult_max_par_dimension()  → highest severity per dimension
  4. sev_mult_combinaison()        → combine dimensions via combi table
  5. sev_mult_effet_fse()          → FSE code boost
  6. sev_mult_effet_age()          → age-based modification
  7. sev_mult_effet_deces()        → death boost (modesortie == 9)
  8. sev_mult_borne_duree()        → duration bounds
```

**Activated** by a toggle switch (`sev_mult_toggle`) in `mod_groupage_mco`'s details panel. Returns a named list with all intermediate and final severity levels.

**Reference tables** come from `sysdata.rda`: `sev_mult_prio`, `sev_mult_dimension`, `sev_mult_combi`, `sev_mult_fse`. DAS-DAS exclusions from `inst/extdata/mco/` parquet.

## Error Handling

- groupeR calls wrapped in `tryCatch` with `waiter$show()`/`waiter$hide()` for UI feedback
- Errors shown via `shiny::showNotification()`
- Failed grouping does NOT update `state$groupage$result()` — it stays `NULL` or at previous value

## Test Patterns

- **Unit tests** for pure builder functions: construct inputs → call builder → assert column names, types, row counts
- **Integration tests** (guarded by `skip_if_not_installed("groupeR")`): full pipeline from defaults → autogrouper → result structure
- **Module UI tests**: `mod_*_ui()` returns valid Shiny tag
- **State tests**: create R6 objects, verify reactiveVal defaults via `shiny::isolate()`
- Test files mirror source files: `test-utils_generateurs_mco.R` tests `utils_generateurs_mco.R`
