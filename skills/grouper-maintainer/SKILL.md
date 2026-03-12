---
name: grouper-maintainer
description: Use when maintaining, extending, or integrating the groupeR R package — adding FG versions, modifying classification pipelines, fixing decision tree logic, working with the metaprogramming or IR core, adding hospital sectors, running golden tests, or calling groupeR as a dependency from another R project. Triggers on groupeR, autogrouper, GHM, GHS, GME, GMT, GHPC, PMSI, DRG grouping, MCO, SMR, HAD, referentiels, ATIH, sysdata.rda, decision tree, CASE WHEN, rlang expr, build_grouper, translate_tests, group_tree.
---

# groupeR Maintainer

## Overview

groupeR is an R package that classifies French hospital stays into reimbursement groups (GHM/GHS for acute care, GME/GMT for rehab, GHPC for home care). It converts ATIH-authored classification algorithms (Excel decision trees) into executable R expressions or DuckDB SQL via **metaprogramming**, then applies them to patient data.

**Core pattern:** Excel trees → `rlang::expr()` / SQL `CASE WHEN` → `dplyr::case_when(!!!exprs)` or raw SQL execution.

## When to Use

**As maintainer:**
- Adding a new FG version (v2027, etc.)
- Modifying pipeline stages (clean, flag, stage, group, exclude, severity, GHS)
- Adding or extending a hospital sector (MCO, SMR, HAD)
- Working with the metaprogramming core (`build_grouper.R`, `translate_tests.R`)
- Working with the IR compilation system (`ir_compile.R`, `ir_execute.R`)
- Running or updating golden regression tests
- Debugging unexpected classification results

**As consumer (dependency):**
- Calling `autogrouper()` from another R project
- Building input tables matching groupeR's data contracts
- Extracting and interpreting results from the `.grouper` object

## Architecture at a Glance

```
autogrouper(.data, type, version)
    │
    ├─ init_grouper() → creates .grouper object
    │   ├─ Detects backend: data.frame | tbl_sql | file paths
    │   ├─ Loads referentiels (pre-parsed decision trees from sysdata.rda)
    │   ├─ Optionally compiles IR (compile_ir())
    │   └─ Assigns S3 classes: grouper_{mco|smr|had} + grouper_{data.frame|tbl_sql}
    │
    ├─ grouper_auto(.grouper) → S3 dispatch by sector
    │   ├─ grouper_auto.grouper_mco → 35-stage pipeline
    │   ├─ grouper_auto.grouper_smr → 28-stage pipeline
    │   └─ grouper_auto.grouper_had → 20-stage pipeline
    │
    └─ Returns .grouper with populated $data$output
```

**Each pipeline stage** is an S3 generic dispatching on backend class (`grouper_data.frame` vs `grouper_tbl_sql`), enabling the same logic to run as in-memory R or pushed-down SQL.

## Key Files

| Area | Files | Purpose |
|------|-------|---------|
| **Entry point** | `R/autogrouper.R` | `autogrouper()`, `grouper_auto()` S3 generic |
| **Init** | `R/init_grouper.R` | `.grouper` constructor, source loading, backend detection |
| **Metaprogramming** | `R/build_grouper.R` | `build_case_when()` — trees → R expr or SQL string |
| **Test translation** | `R/translate_tests.R` | `match_test_fun()` → 15+ translators (isListDiag, isVarVal, etc.) |
| **Tree execution** | `R/group_tree.R` | `group_tree()` S3 — applies case_when to data |
| **IR system** | `R/ir_compile.R`, `ir_schema.R`, `ir_execute.R`, `ir_validate.R` | Alternative compilation with deduplication |
| **Referentiel parsing** | `R/init_referentiels.R` | Excel → R data structures |
| **Data contracts** | `R/checks_data_contracts.R` | Input validation and column derivation |
| **Golden tests** | `R/golden_harness.R`, `golden_compare.R` | Regression test framework |
| **MCO pipeline** | `R/mco_*.R` (35 files) | Clean → Flag → Stage → Group → Exclude → Severity → GHS |
| **SMR pipeline** | `R/smr_*.R` (21 files) | Clean → Stage → Flag → Group → Severity → GME/GMT |
| **HAD pipeline** | `R/had_*.R` (15 files) | Clean → Stage → Group → Exclude → Severity → GPSL |
| **Bundled data** | `R/sysdata.rda` | Pre-built referentiels object |
| **Build scripts** | `data-raw/referentiels.R` | Builds sysdata.rda from Excel sources |

## The .grouper Object

```r
.grouper <- structure(list(
    sources = <referentiels for this sector+version>,
    data = list(
        input   = <original .data (list of tables)>,
        refs    = <cleaned + flagged intermediate tables>,
        staging = <prepared tables ready for grouping>,
        output  = <final results (grp_cmd, grp_racine, grp_ghm, grp_ghs, etc.)>
    ),
    opts = list(
        type    = "mco",          # sector
        version = "v2025",        # FG version
        usr_opts = list(...),     # user options (stop points, paths)
        con     = <DBI connection or NULL>
    ),
    ir = <compiled IR or NULL>
), class = c("grouper_mco_tbl_sql", "grouper_tbl_sql", "grouper_mco", "list"))
```

**Accumulator pattern:** Each pipeline stage reads from `.grouper`, mutates it, and returns it. Data flows through `$data$refs` → `$data$staging` → `$data$output`.

## Consumer API

### Entry Point

```r
result <- autogrouper(.data, type, version, sources = NULL, ...)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `.data` | list | Named list of input tables (see data-contracts.md) |
| `type` | character | `"mco"`, `"smr"`, or `"had"` |
| `version` | character | `"v2023"`, `"v2025"`, `"v2026"` (MCO); `"v2022"`, `"v2025"` (SMR); `"vExp"` (HAD) |
| `sources` | list | Custom referentiels (default: bundled `referentiels$<type>$<version>`) |

### Supported Backends

| Backend | Class | When |
|---------|-------|------|
| In-memory | `data.frame` / `tibble` | Small data (<10K stays) |
| DuckDB | `tbl_sql` | Large data, pushed-down SQL |
| File paths | `character` | Parquet files (auto-creates DuckDB; requires `path_db_results`) |

**All tables in `.data` must share the same backend.** No mixing.

### Result Extraction

```r
# MCO
result$data$output$grp_ghs_tot_all  # Main: ident, ghm, ghs, cmd, racine, niv_sev
result$data$output$grp_cmd          # CMD assignment
result$data$output$grp_racine_all   # Root GHM before severity

# SMR
result$data$output$grp_gmt_all      # GMT (financing)
result$data$output$grp_gme_all      # GME (activity)

# HAD
result$data$output$grp_gpsl          # GPSL (group/procedure/severity/burden)
```

### Gotchas

1. **Column names must match exactly** — lowercase PMSI format (`ident`, `age`, `modeentree`, not `IDENT`, `mode_entree`)
2. **Diagnosis position codes are sector-specific** — MCO: `"DP"/"DR"/"DAS"`; SMR: `"FPP"/"MMP"/"AE"/"DAS"`; HAD: `"MPP"/"MPA"/"DAS"`
3. **Version must be ≥ data year** — v2025 rules can group 2020–2025 data, but v2023 cannot properly group 2025 data
4. **DuckDB memory** for >5M stays: `dbConnect(duckdb(config = list(memory_limit = "40GB")))`
5. **Connection cleanup** — always use `withr::defer(dbDisconnect(con))` or `on.exit()`

## Development Commands

```bash
air format .                                     # Format code
Rscript -e "jarl::lint_package()"                # Lint
Rscript -e "testthat::test_local()"              # Test
Rscript -e "devtools::document()"                # Rebuild docs/NAMESPACE
Rscript -e "devtools::check()"                   # Full R CMD check
```

## Critical Conventions

- **S3 dispatch on two axes:** sector (`grouper_mco`) × backend (`grouper_data.frame` / `grouper_tbl_sql`)
- **French domain names** in data columns (`ident`, `modeentree`, `modesortie`, `duree`, `racine`)
- **English function/file names** (`mco_data_clean_fixe`, `build_case_when`)
- **Air formatter** enforced; 4-space indent
- **Never rebuild `sysdata.rda` or `data/`** — human-only task (requires `data-raw/` scripts with source Excel files)
- **Pipeline stage functions** follow pattern: `{sector}_{phase}_{detail}()` → reads/writes `.grouper`
- **Each stage has two S3 methods:** `.grouper_data.frame` and `.grouper_tbl_sql`

## Common Maintainer Tasks

### Adding a new FG version

See development-guide.md § "Adding a Version" — involves Excel parsing, referentiel build script, version registration in init_grouper, golden fixture generation.

### Adding a pipeline stage

1. Create `R/{sector}_{phase}_{name}.R` with S3 generic + two methods (data.frame, tbl_sql)
2. Add call in `R/grouper_auto_{sector}.R` at correct position
3. Write tests in `tests/testthat/test-{sector}_{phase}_impl.R`
4. Update golden fixtures if output changes

### Working with the metaprogramming core

See architecture.md § "Metaprogramming System" — covers `build_case_when()`, `match_test_fun()`, test translators, R vs SQL paths.

### Working with the IR system

See architecture.md § "IR System" — the IR deduplicates test predicates and normalizes trees. Opt-in via `getOption("groupeR.use_ir")`.

### Running golden tests

See architecture.md § "Golden Testing" — `golden_validate()` compares pipeline checkpoints against baseline fixtures.

## Supplementary Files

- **architecture.md** — Full pipeline details, metaprogramming core, IR system, golden tests, severity calculation, S3 dispatch patterns
- **data-contracts.md** — Complete column specs for every input table (MCO, SMR, HAD)
- **development-guide.md** — Step-by-step: adding versions, sectors, pipeline stages, working with referentiels
