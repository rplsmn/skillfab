# groupeR Architecture Reference

## Pipeline Overview by Sector

### MCO (Acute Care) — 35 stages

```
Phase 1: CLEAN (3)        Phase 2: FLAGS (5+1)       Phase 3: STAGING (3)
─────────────────────     ─────────────────────      ─────────────────────
mco_data_clean_fixe       mco_transcodage            mco_fixe_staging
mco_data_clean_diag       mco_flag_gnn               mco_diag_staging
mco_data_clean_acte       mco_flag_inv_dp_dr         mco_acte_staging
                          mco_flag_ag                    ↓ [stop=staging]
                          mco_flag_acte_classant
                          mco_flag_ghm_erreur

Phase 4: GROUPING (2)     Phase 5: EXCLUSIONS (4)    Phase 6: SEVERITY (9)
─────────────────────     ─────────────────────      ─────────────────────
mco_grp_cmd               mco_exclusion_racine       mco_diag_exclu_staging
   ↓ [stop=grp_cmd]       mco_exclusion_age          mco_severite_init_niv_das
mco_grp_racine_all        mco_exclusion_dpdas        mco_severite_niv_max
   ↓ [stop=grp_racine]    mco_data_exclusion_all     mco_severite_effet_age
                                                      mco_severite_effet_dc
                                                      mco_severite_effet_age_gest
                                                      mco_severite_borne_duree
                                                      mco_severite_borne_duree_lettre
                                                      mco_severite_flags_court_sejour
                                                         ↓ [stop=grp_ghm]

Phase 7: GHS (7+)
─────────────────────
mco_decoration
mco_fixe_ghs_staging     ↓ [stop=fixe_ghs]
mco_grp_ghs_uhcd         ↓ [stop=grp_ghs_uhcd]
mco_grp_ghs_sans_test    ↓ [stop=grp_ghs_sans_test]
mco_data_staging_pregrad  ↓ [stop=df_pregrad]
mco_grp_pregrad           ↓ [stop=grp_ghs_pregrad]
mco_grp_ghs_restants_all  ↓ [stop=grp_ghs_grad]
mco_output_grp_ghs
```

**Stop points:** MCO supports early exit via `stop` option in `usr_opts`. Useful for debugging specific pipeline stages.

### SMR (Rehabilitation) — 28 stages

```
Phase 1: CLEAN (5)         Phase 2: STAGING (4)        Phase 3: FLAGS (4)
──────────────────────    ──────────────────────      ──────────────────────
smr_data_base_views        smr_data_transcode_diags    smr_data_flag_gmth
smr_data_clean_fixe        smr_data_staging_fixe       smr_data_flag_nbjpres_sej
smr_data_clean_fixe_sej    smr_data_staging_diag       smr_data_flag_ponderation_actes
smr_data_clean_acte_ccam   smr_data_staging_actes      smr_flag_vars_lourd
smr_data_clean_acte_readaptation

Phase 4: GROUPING (3)     Phase 5: RR SCORE (4)       Phase 6: SEVERITY (6)
──────────────────────    ──────────────────────      ──────────────────────
smr_groupe_cm              smr_data_flag_score_rr_glob smr_diag_cma_init
smr_groupe_gn              smr_data_flag_score_rr_spe  smr_exclusion_dpdas
smr_groupe_gr              smr_data_flag_score_rr_tot  smr_diag_cma
                           smr_groupe_lourdeur         smr_data_niv_codes_sej
                                                       smr_groupe_severite

Phase 7: OUTPUT (2)
──────────────────────
smr_gme → GME result
smr_gmt → GMT result
```

### HAD (Home Care) — 20 stages

```
Phase 1: CLEAN (5)         Phase 2: STAGING (2)        Phase 3: GROUPING (5)
──────────────────────    ──────────────────────      ──────────────────────
had_data_base_views        had_data_diag_seq           had_grp_gp
had_data_clean_fixe        had_data_acte_seq           had_grp_gs
had_data_fixe_seq                                      exclu_gs_gs
had_data_lourd                                         had_set_gp_sej
had_data_mp_seq                                        had_grp_metagp

Phase 4: EXCLUSIONS (3)   Phase 5: SEVERITY (4)       Phase 6: OUTPUT (1)
──────────────────────    ──────────────────────      ──────────────────────
exclu_gp_gs                set_sev_tot                 had_set_gpsl → GPSL
exclu_gs_gs_sej            set_sev_sej
had_update_gp_mgp          set_lourd_tot
                           set_lourd_sej
```

### Sector Comparison

| Aspect | MCO | SMR | HAD |
|--------|-----|-----|-----|
| **Output** | GHM (6 chars) + GHS | GME + GMT | GPSL |
| **Grouping dims** | CMD → Racine (5 chars) + severity (1-4) | CM + GR + GN | GP + GS |
| **Severity** | Numeric 1-4 with age/death/gest majorations | Lookup (CM,GR,GN)→sev | Text (LEGER/MOYEN/LOURD) |
| **Duration path** | Integer (1-4) vs Letter (A-D) fork | Single | Single |
| **Exclusions** | 3 types: racine, age, DP/DAS | DP/DAS only | GP/GS, GS/GS |
| **Pipeline files** | 35 | 21 | 15 |

## S3 Dispatch System

groupeR uses **dual-axis S3 dispatch**:

**Axis 1 — Sector type:** `grouper_mco`, `grouper_smr`, `grouper_had`
- Controls which pipeline runs (via `grouper_auto()`)
- Controls which referentiels are loaded (via `init_sources_grouper()`)

**Axis 2 — Data backend:** `grouper_data.frame`, `grouper_tbl_sql`
- Controls HOW each stage executes (R in-memory vs SQL push-down)
- Every pipeline stage function dispatches on this

**Class vector example:**
```r
c("grouper_mco_tbl_sql", "grouper_tbl_sql", "grouper_mco", "list")
```

**When adding a new stage:** Write a generic + two methods:
```r
mco_new_stage <- function(.grouper) UseMethod("mco_new_stage")

mco_new_stage.grouper_data.frame <- function(.grouper) {
    # In-memory R logic using dplyr
    .grouper$data$output$new_result <- ...
    .grouper
}

mco_new_stage.grouper_tbl_sql <- function(.grouper) {
    # SQL logic using dbplyr or raw SQL via .grouper$opts$con
    .grouper$data$output$new_result <- ...
    .grouper
}
```

## Metaprogramming System

### Overview

The metaprogramming system converts tabular decision trees into executable code. Three layers:

```
Excel trees (data-raw)
    ↓ init_referentiels.R
Referentiels (nested R lists in sysdata.rda)
    ↓ build_grouper.R + translate_tests.R
R expressions (rlang::expr) or SQL strings
    ↓ group_tree.R
Execution via case_when() or CASE WHEN
```

### build_grouper.R — Tree → Expressions

**`build_case_when(tree_params, con = NULL)`** is the core function.

- `tree_params`: List with `tree` (data frame of paths), `default` value, metadata
- `con = NULL` → R path: returns `expr(case_when(!!!cases))`
- `con = <DBI>` → SQL path: returns SQL `CASE WHEN ... END` string

For each row in the tree (= one path to a leaf):
1. Collect all tests on that path
2. For each test: call `match_test_fun(test_type)` to get translator
3. Translator returns an `rlang::expr()`
4. Combine tests with `&` (AND)
5. Create `condition ~ leaf_value` formula
6. Wrap all formulas: `case_when(!!!all_formulas)`

### translate_tests.R — Test Type → Expression

**`match_test_fun(test_val)`** dispatches to 15+ translator functions:

| Function | Test types | Generates |
|----------|-----------|-----------|
| `isListDiag` | DP, DR, DD, DIAG, Fpp, Mmp, Ae, DAS | `any(liste == !!val & pos_code == !!pos, na.rm = TRUE)` |
| `isListActe` | ACTE, AG, AA, AO | `any(liste == !!val & is_acte, na.rm = TRUE)` |
| `isListMP` | MP, MPP, MPA | `any(liste == !!val & pos_code == !!pos, na.rm = TRUE)` |
| `isVarVal` | AGE_SUP/INF/EGAL, PDS_*, DUREE_*, SEANC_* | `any(age > !!n & !is.na(age), na.rm = TRUE)` |
| `isVarVals` | SEXE, ME, MS | `any(var %in% c(!!vals), na.rm = TRUE)` |
| `isVarChar` | GN, GR, CMD, LOURD | `any(var == !!val, na.rm = TRUE)` |
| `isNotA` | NOT_A | `!any(liste == !!val & is_acte, na.rm = TRUE)` |
| `isNotDPA` | NOT_DP_A | `!any(liste == !!val & pos_code == "DP", na.rm = TRUE)` |
| `isDIAB` | DIAB | HAD-specific diabetes test |
| `isUNITENBJ` | UNITENBJ | SMR presence days test |
| `isTypeRR` | TYPE_RR | SMR rehabilitation type |
| `isSRRSupOrEqual` | SRR>=val | SMR RR score comparison |
| `isGrpError` | GRP_ERREUR | Error group detection |

### group_tree.R — Execution

**`group_tree(data, fn_grouper, var_id, var_result)`** — S3 generic:

- `.data.frame` method: `dplyr::summarise(result := case_when(!!!fn_grouper), .by = var_id)`
- `.tbl_sql` method: Constructs SQL `SELECT ... CASE WHEN ... END` and returns lazy query

### R vs SQL Divergence

```
Same tree definition
    │
    ├─ R path (con = NULL):
    │   build_case_when() → rlang::expr(case_when(...))
    │   group_tree.data.frame() → dplyr::summarise(case_when(!!!exprs))
    │
    └─ SQL path (con = DBI):
        build_case_when() → "CASE WHEN ... THEN ... END" string
        group_tree.tbl_sql() → SELECT ... CASE WHEN ... (lazy dbplyr query)
```

## IR System (Intermediate Representation)

The IR is an **optional optimization layer** that deduplicates test predicates and normalizes tree structures. Opt-in via `getOption("groupeR.use_ir", default = FALSE)`.

### Compilation Pipeline

```
compile_ir(sources, type, version)
    ├─ collect_tree_params() → uniform tree format
    ├─ compile_test_catalog() → deduplicated test_id → test/val pairs
    ├─ compile_decision_graph() → path-based trees referencing test_ids
    ├─ compile_outputs() → leaf value registry
    ├─ compile_metadata() → stats, hashes, version info
    └─ validate_ir() → compile-time integrity checks
```

### IR Schema

```r
grouper_ir {
    test_catalog: tibble(test_id, test_type, val_type, val, domain)
    decision_graph: list(
        cmd = ir_tree { paths, default },
        racine_CMD01 = ir_tree { paths, default },
        gs = ir_tree_nested { subtrees: list(ir_tree), nest_var }
    )
    outputs: tibble(tree_name, leaf_value, label)
    metadata: list(fg_version, champ, source_hash, stats, ...)
}
```

### Execution

`execute_tree()` S3 generic dispatches on data class:
- **data.frame**: Rebuilds case_when expressions from test_catalog
- **tbl_sql**: Generates SQL CASE statements from test_catalog

The IR does NOT replace the main pipeline — it replaces `build_case_when()` with a normalized, deduplicated equivalent.

## Golden Testing System

A **regression testing framework** that captures intermediate outputs ("checkpoints") and compares against baseline fixtures.

### Core Functions

| Function | Purpose |
|----------|---------|
| `golden_validate(.grouper, champ, golden_dir, ...)` | Main entry: run comparison, return pass/fail |
| `golden_generate(.grouper, champ, golden_dir)` | Create baseline fixtures (parquet + metadata) |
| `golden_extract_checkpoints(.grouper, champ)` | Extract named outputs from result |
| `golden_compare_all(current, golden)` | Multi-checkpoint comparison |
| `golden_compare_checkpoint(current, golden, ...)` | Row-by-row diff with NA-safe equality |
| `golden_trace_divergence(comparison, ident)` | Find first pipeline divergence for one stay |

### Checkpoint Registry (per sector)

```r
golden_checkpoint_registry("mco")  # → grp_cmd, grp_racine_all, grp_ghm_all, grp_ghs_tot_all
```

### Allowlist System

YAML-format allowlist for known acceptable diffs:
```yaml
- checkpoint: grp_ghs_tot_all
  idents: ["SEJ001", "SEJ002"]
  columns: ["ghs"]
```

Golden fixtures are stored as parquet files with `_metadata.parquet` containing MD5 checksums, row counts, and timestamps.

## MCO Severity Calculation (niv_sev 1-4)

```
DAS codes (after exclusions)
    ↓
init_niv_das: join with niv_das_ref → initial niv per DAS (1-4)
    ↓
niv_max: max(niv) per stay, combine with racine 6th char → niv_base
    ↓
effet_age:
  • age < 2 & racine allows → bump 1→2
  • age > 69/79 & racine allows → bump +1 (capped)
    ↓
effet_dc: modesortie == 9 & niv == 1 → niv = 2
    ↓
effet_age_gest: gestational age majorations (if applicable)
    ↓
FORK:
  ├─ INTEGER path (no 6th char, no age_gest):
  │   duree < 3 → niv=1; duree < 4 & niv>2 → niv=2; duree < 5 & niv>3 → niv=3
  │   → niv_ds (integer 1-4)
  │
  └─ LETTER path (6th char OR age_gest):
      B/C/D → A if duree<3; C/D → B if duree<4; D → C if duree<5
      → pre_niv (letter A-D), pre_niv_ds (capped)
    ↓
DECORATION: racine + (flag_j | flag_t | pre_niv_ds | niv_ds) → final GHM (6 chars)
```

**Key severity variables:**
- `niv_base` — foundation from DAS levels or racine 6th char
- `niv_age` — after age majorations
- `niv_dc` — after death effect (used for GHM_RAAC)
- `niv_ds` — final integer path (after duration caps)
- `pre_niv_ds` — final letter path (after duration caps)

## MCO Exclusion Rules

### Racine exclusions (`mco_exclusion_racine`)
DAS codes excluded based on GHM root. Inner join (racine_5char + DAS code) with `exclusions$racine` table.

### Age exclusions (`mco_exclusion_age`)
Perinatal codes (P*) excluded when patient age > 1 year.

### DP/DAS exclusions (`mco_exclusion_dpdas`)
Removes DAS codes based on primary diagnosis pairs:
1. Extract DP/DR per stay
2. Extract all DAS per stay
3. Cartesian product: DP × DAS
4. Semi-join with `exclusions$das` reference table
5. Matching DAS codes marked as excluded

### Finalization (`mco_data_exclusion_all`)
Union all exclusion types → anti-join diag_staging on `(ident, code, pos_code)` → only non-excluded diagnoses proceed to severity.

## Referentiels Structure

```r
referentiels$mco$v2025$arbres$cmd        # CMD decision tree
referentiels$mco$v2025$arbres$racine     # Per-CMD racine trees (list)
referentiels$mco$v2025$arbres$ghs        # GHS trees
referentiels$mco$v2025$listes$codes      # Code → list membership
referentiels$mco$v2025$exclusions$das    # DP/DAS exclusion pairs
referentiels$mco$v2025$exclusions$racine # Racine/DAS exclusion pairs
referentiels$mco$v2025$severite$niv_das  # DAS → severity level mapping
referentiels$mco$v2025$severite$eff_age  # Racine age effect flags
```

Source Excel files: `inst/extdata/{mco,smr,had}/`

Built by: `data-raw/referentiels.R` → calls sector-specific scripts → `usethis::use_data()` → `R/sysdata.rda` (internal) + `data/referentiels.rda` (public)
