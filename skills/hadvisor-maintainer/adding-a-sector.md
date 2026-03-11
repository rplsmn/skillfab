# Adding a New Sector to Hadvisor

Step-by-step checklist derived from how MCO was added alongside HAD. Use SSR/SMR as the running example.

## Pre-requisites

- [ ] groupeR supports the sector: `groupeR::referentiels$ssr` exists with a version
- [ ] You know the sector's input schema: what tables `autogrouper(type = "ssr")` expects
- [ ] Load the `grouper-consumer` skill for groupeR API details

## Checklist

### 1. R6 State Classes (`R/fct_app_state.R`)

- [ ] Create `SsrPatientState` (R6, following `McoPatientState` pattern)
  - Determine the repeating unit (HAD = sequences, MCO = RUMs, SSR = ?)
  - Add reactiveVal slots: `n_unit_current`, `n_unit`, `add_n_unit`, `unitnames`, `unit_inputs`, `sejour_inputs` (if applicable), `copy`, `gogrp`, `ident_sej`
- [ ] Create `SsrGroupageState` (clone of `McoGroupageState`: `result`, `details`, `history`, `input_data`)
- [ ] Create `SsrState` composing `patient` + `groupage`
- [ ] Add `ssr: SsrState$new()` to `AppState$initialize()`

### 2. Data Builders (`R/utils_generateurs_ssr.R`)

- [ ] Create `make_ssr_tbl_*()` functions for each table groupeR expects
  - Use `grouper-consumer` skill to get exact column names and types
  - Follow pattern in `utils_generateurs_mco.R`: one function per table, one `assemble_ssr_grouper_data()` master
- [ ] Create `init_ssr_*_default_vals()` for sensible defaults
- [ ] Write tests first in `tests/testthat/test-utils_generateurs_ssr.R`

### 3. Input Modules

Create sector-specific input modules following existing patterns:

- [ ] `R/mod_ssr_patients.R` — Container with dynamic accordion (like `mod_mco_patients.R`)
- [ ] `R/mod_ssr_sejour_inputs.R` — Stay-level inputs (if the sector has them)
- [ ] `R/mod_ssr_unit_inputs.R` — Per-unit inputs (like `mod_mco_rum_inputs.R`)
- [ ] Any modals for complex inputs (like `mod_mco_actes_modal.R`)

**Each module must:**
- Accept `state` argument (the sector's `SsrState`)
- Store inputs into `state$patient$*_inputs` reactiveValues
- Implement copy/paste via `state$patient$copy()`
- Use `bttnCopyPaste()` from `utils_ui_generators.R`

### 4. Groupage Module (`R/mod_groupage_ssr.R`)

- [ ] Clone from `mod_groupage_mco.R` and adapt:
  - Change `autogrouper(type = "ssr", version = ..., sources = groupeR::referentiels$ssr$...)`
  - Adjust result extraction paths (check groupeR output structure for SSR)
  - Adapt value boxes and details panel to SSR output fields
- [ ] Write tests in `tests/testthat/test-mod_groupage_ssr.R`

### 5. UI Updaters (`R/utils_ui_updaters_ssr.R`)

- [ ] Create `update_all_ssr_inputs()` for copy/paste sync (if needed)

### 6. Wire Into App Shell

**`R/app_ui.R`:**
- [ ] Add SSR to `radioGroupButtons("sector", ...)` choices
- [ ] Add conditional SSR sidebar controls (n_unit input + "Generer" button)
- [ ] Add `nav_panel_hidden(value = "ssr", ...)` containing `mod_ssr_patients_ui()` + `mod_groupage_ssr_ui()`

**`R/app_server.R`:**
- [ ] Add SSR observers for n_unit / add_n_unit (forwarding sidebar inputs to state)
- [ ] Extend the `input$go` observer's `switch()` to include `ssr = state$ssr$patient$gogrp(input$go)`
- [ ] Call `mod_ssr_patients_server()` and `mod_groupage_ssr_server()` with `state = state$ssr`

### 7. CSS (`inst/app/www/css/monzhad.css`)

- [ ] Add `grp-badge--ssr` class following existing badge pattern
- [ ] Add `grp-radio--ssr` and `grp-switch--ssr` if needed
- [ ] Group new classes under a `/* SSR */` comment header

### 8. Package Metadata

- [ ] Add roxygen `@importFrom` for any new dependencies in `R/hadvisor-package.R`
- [ ] Run `devtools::document()` to update NAMESPACE
- [ ] Update DESCRIPTION if new packages needed

### 9. Input Choice Data (if new referentials)

- [ ] Update `data-raw/sources_default.R` to build SSR-specific choice vectors
- [ ] Rebuild `data/sysdata.rda` with `source("data-raw/sources_default.R")`

### 10. Integration Test

- [ ] Add `test-autogrouper_integration_ssr.R` (or extend existing)
- [ ] Test: build SSR tables from defaults → call `autogrouper(type="ssr")` → verify result structure
- [ ] Guard with `skip_if_not_installed("groupeR")`

## File Naming Convention

Follow the established pattern — sector name as infix:

```
R/mod_{sector}_patients.R
R/mod_{sector}_sejour_inputs.R    # if applicable
R/mod_{sector}_{unit}_inputs.R    # rum, sequence, etc.
R/mod_groupage_{sector}.R
R/utils_generateurs_{sector}.R
R/utils_ui_updaters_{sector}.R
tests/testthat/test-utils_generateurs_{sector}.R
tests/testthat/test-mod_{sector}.R
```

## Common Pitfalls

- **typ_diag codes differ by sector.** HAD uses "DP"/"MPP"/"MPA"/"DAS". MCO uses "1"/"2"/"5". SSR will have its own mapping — check groupeR docs.
- **RUM/sequence indexing is zero-padded** (`sprintf("%02d", i)`). Keep this convention.
- **Observer priorities matter.** Panel insert (9999) must run before data assembly (97). Copy the priority pattern from MCO modules.
- **The "Go" button is shared** across all sectors. The sector switch in `app_server.R` dispatches to the right state. Don't create a sector-specific button.
- **sysdata.rda is a single file** containing all bundled referentials. Rebuilding it replaces everything — make sure `sources_default.R` includes all sectors.
