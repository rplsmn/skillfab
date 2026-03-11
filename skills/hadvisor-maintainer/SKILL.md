---
name: hadvisor-maintainer
description: Use when working on the hadvisor R Shiny application - adding features, fixing bugs, extending to new sectors (MCO, SSR), modifying groupeR integration, or understanding the golem/R6 architecture. Triggers on hadvisor, PMSI, DRG grouping, HAD, MCO, SSR, groupeR, autogrouper, bslib, golem module work.
---

# Hadvisor Maintainer

## Overview

Hadvisor is a golem-framework R Shiny app for simulating French DRG grouping (PMSI). Users fill in stay characteristics per sector (HAD, MCO, future SSR) and get grouping results via the internal `groupeR` package. State is managed through composed R6 classes (`AppState` → per-sector `PatientState` + `GroupageState`) with reactive slots.

## When to Use

- Adding or modifying features in any hadvisor module
- Fixing bugs in data assembly, groupeR integration, or UI rendering
- Adding a new hospital sector (SSR, PSY, etc.)
- Working with the experimental severity pipeline (`sev_mult_*`)
- Understanding the module/state/data flow architecture

## Architecture at a Glance

```
app_ui.R (page_sidebar + navset_hidden per sector)
    ↓
app_server.R (creates AppState, wires sector switch + "Grouper" button)
    ↓
┌─── HAD ───────────────────────┐  ┌─── MCO ───────────────────────┐
│ mod_patients → mod_seq_inputs │  │ mod_mco_patients              │
│                               │  │  ├─ mod_mco_sejour_inputs     │
│                               │  │  └─ mod_mco_rum_inputs        │
│                               │  │      └─ mod_mco_actes_modal   │
│ mod_groupage                  │  │ mod_groupage_mco              │
└───────────────────────────────┘  └───────────────────────────────┘
    ↓                                  ↓
utils_generateurs.R                utils_generateurs_mco.R
    ↓                                  ↓
groupeR::autogrouper(type="had")   groupeR::autogrouper(type="mco")
```

**State flows one way:** UI inputs → `state$patient$*_inputs` → builder functions → `state$groupage$input_data()` → groupeR → `state$groupage$result()` → render outputs.

See architecture.md for the full R6 hierarchy, module map, observer priorities, data flow diagram, and test structure.

## Quick Reference: Key Files

| What | Where |
|------|-------|
| R6 state classes | `R/fct_app_state.R` |
| App shell | `R/app_ui.R`, `R/app_server.R` |
| HAD modules | `R/mod_patients.R`, `R/mod_sequence_inputs.R`, `R/mod_groupage.R` |
| MCO modules | `R/mod_mco_patients.R`, `R/mod_mco_sejour_inputs.R`, `R/mod_mco_rum_inputs.R`, `R/mod_mco_actes_modal.R`, `R/mod_groupage_mco.R` |
| HAD data builders | `R/utils_generateurs.R` |
| MCO data builders | `R/utils_generateurs_mco.R` |
| Severity pipeline | `R/utils_severite_mult.R` |
| CSS | `inst/app/www/css/monzhad.css` |
| Bundled data | `data/sysdata.rda` (built by `data-raw/sources_default.R`) |
| Parquet refs | `inst/extdata/mco/` |
| Config | `inst/golem-config.yml`, `DESCRIPTION` |

## Common Tasks

### Adding a field to an existing sector

1. Add the input widget in the sector's `mod_*_inputs_ui()`
2. Store it in the module's server: `state$patient$*_inputs[[module_name]]$new_field <- input$new_field`
3. Include it in the relevant `make_*_tbl_*()` builder (check groupeR expects it)
4. Update `init_*_default_vals()` with a sensible default
5. Update copy/paste: add to the updater function in `utils_ui_updaters*.R`
6. Update tests

### Adding a new sector

Follow the detailed checklist in adding-a-sector.md — covers R6 state, builders, modules, app shell wiring, CSS, tests, and common pitfalls.

### Modifying groupeR integration

Always use the `grouper-consumer` skill. Key points:
- groupeR is called via `groupeR::autogrouper(.data = list(...), type = "<sector>", version = "...", sources = groupeR::referentiels$<sector>$<version>)`
- Table schemas (column names, types) must match exactly
- Result paths differ by sector — check `result$data$output$*`

### Working with severity pipeline

`utils_severite_mult.R` is pure functions (no Shiny). Steps documented in patterns.md. Unit tests are extensive in `test-utils_severite_mult.R`. Reference tables from `sysdata.rda`.

## Critical Conventions

- **CSS:** `grp-<block>--<modifier>` prefix, single file `monzhad.css`, no inline `style=`
- **Observer priorities:** Panel insert (9999) > ident generation (99) > data assembly (97)
- **Module naming:** `"{sector}_{unit}_inputs_{n}"` as keys in reactiveValues
- **Tidyverse style:** purrr over base apply, dplyr/tidyr, cli for messages, rlang for conditions
- **Tests:** Write tests first. Guard groupeR calls with `skip_if_not_installed("groupeR")`

## Supplementary Files

- **architecture.md** — Full R6 hierarchy, module map, data flow diagram, observer priorities, test structure, groupeR result paths
- **adding-a-sector.md** — Step-by-step checklist for adding SSR or any new sector
- **patterns.md** — Dynamic module insertion, copy/paste, data assembly, CSS, severity pipeline, error handling, test patterns
