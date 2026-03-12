# Development Guide — groupeR

## Adding a New FG Version

When ATIH releases a new classification version (e.g., v2027):

### 1. Obtain Source Excel Files

Place Excel files in `inst/extdata/{sector}/` following the naming convention:
- MCO: `inst/extdata/mco/v2027/` (tree workbooks, exclusion tables, severity refs)
- SMR: `inst/extdata/smr/v2027/`
- HAD: `inst/extdata/had/v2027/`

### 2. Create Referentiel Build Script

Create `data-raw/{sector}_referentiel_v2027.R`:

```r
# Pattern from existing scripts (e.g., mco_referentiel_v2025.R):
# 1. Read Excel sheets with readxl
# 2. Parse decision trees via init_grouper_sources()
# 3. Parse exclusion tables
# 4. Parse severity reference tables
# 5. Build the nested list structure for this version
```

### 3. Register in Sector Referentiels

Update `data-raw/{sector}_referentiels.R` to include the new version:

```r
mco_referentiels <- list(
    v2023 = ...,
    v2025 = ...,
    v2026 = ...,
    v2027 = <new version output>  # Add here
)
```

### 4. Register in init_grouper.R

Update the `init_sources_grouper.grouper_{sector}()` method to recognize the new version string. The version is used to index into `referentiels$<sector>$<version>`.

### 5. Rebuild sysdata.rda (Human-Only)

```r
# Run in R console with source data available:
source("data-raw/referentiels.R")
# → Rebuilds R/sysdata.rda and data/referentiels.rda
```

**NEVER have an LLM agent run this.** It requires the source Excel files and specific R environment.

### 6. Generate Golden Fixtures

Run the pipeline on test data with the new version and generate golden fixtures:

```r
result <- autogrouper(.data, type = "mco", version = "v2027")
golden_generate(result, champ = "mco", golden_dir = "tests/testthat/golden/mco/v2027")
```

### 7. Update Tests

Add the new version to integration tests in `tests/testthat/test-grouper_auto_{sector}.R`.

### 8. Update Documentation

- `DESCRIPTION`: If new dependencies needed
- `NEWS.md`: Note the new version
- Consumer-facing docs

---

## Adding a New Pipeline Stage

### 1. Create the Stage File

`R/{sector}_{phase}_{name}.R` with S3 generic + two methods:

```r
#' @title Brief description
#' @param .grouper A grouper object
#' @return Modified .grouper
mco_new_stage <- function(.grouper) {
    UseMethod("mco_new_stage")
}

#' @export
mco_new_stage.grouper_data.frame <- function(.grouper) {
    # Read from .grouper
    input <- .grouper$data$staging$some_table
    refs <- .grouper$sources$some_ref

    # Transform
    result <- input |>
        dplyr::left_join(refs, by = "key") |>
        dplyr::mutate(new_col = ...)

    # Write back
    .grouper$data$output$new_result <- result
    .grouper
}

#' @export
mco_new_stage.grouper_tbl_sql <- function(.grouper) {
    con <- .grouper$opts$con
    input <- .grouper$data$staging$some_table

    # Same logic but ensure it works with dbplyr lazy evaluation
    result <- input |>
        dplyr::left_join(refs, by = "key") |>
        dplyr::mutate(new_col = ...)

    .grouper$data$output$new_result <- result
    .grouper
}
```

### 2. Insert in Pipeline

Add the call in `R/grouper_auto_{sector}.R` at the correct position:

```r
.grouper <- .grouper |>
    existing_stage() |>
    mco_new_stage() |>     # ← Insert here
    next_existing_stage()
```

### 3. Write Tests

Create `tests/testthat/test-{sector}_{phase}_impl.R`:

```r
test_that("mco_new_stage handles normal case", {
    # Build minimal .grouper with required inputs
    # Call stage
    # Assert outputs
})
```

### 4. Update Golden Fixtures

If the new stage changes any checkpoint output, regenerate golden fixtures.

---

## Adding a New Hospital Sector

This is a major effort. Follow the MCO/SMR/HAD pattern:

### 1. Referentiels

- Source Excel files in `inst/extdata/<sector>/`
- Build script: `data-raw/<sector>_referentiel_<version>.R`
- Register in `data-raw/referentiels.R`

### 2. Pipeline

- `R/grouper_auto_<sector>.R` — Pipeline orchestration
- `R/<sector>_data_clean_*.R` — Cleaning stages
- `R/<sector>_data_staging_*.R` — Staging stages
- `R/<sector>_groupe_*.R` — Grouping stages
- Each with S3 generics + data.frame/tbl_sql methods

### 3. Init

- Add `init_sources_grouper.grouper_<sector>()` in `R/init_grouper.R`
- Register valid versions

### 4. S3 Classes

The new sector gets class `grouper_<sector>` automatically via `new_grouper_type()`.

### 5. Tests

- `tests/testthat/test-grouper_auto_<sector>.R` — Integration
- `tests/testthat/test-<sector>_*_impl.R` — Unit tests per stage

### 6. Golden

- Add checkpoint registry: `golden_checkpoint_registry("<sector>")`
- Generate initial fixtures

---

## Modifying a Test Translator

Test translators live in `R/translate_tests.R`. Each converts a test type (from Excel tree columns) into an `rlang::expr()`.

### Adding a New Test Type

1. Write the translator function:

```r
isNewTest <- function(test_type, val, con = NULL) {
    # Parse val
    n <- as.numeric(val)
    
    if (is.null(con)) {
        # R path
        rlang::expr(any(new_var > !!n & !is.na(new_var), na.rm = TRUE))
    } else {
        # SQL path — must return valid SQL fragment
        glue::glue("(new_var > {n})")
    }
}
```

2. Register in `match_test_fun()`:

```r
match_test_fun <- function(test_val) {
    switch(test_val,
        "EXISTING" = isExisting,
        "NEW_TYPE" = isNewTest,    # ← Add here
        stop("Unknown test type: ", test_val)
    )
}
```

3. If the test uses a new data column, ensure it's available in the staging tables.

---

## Working with Referentiels

### Structure

```r
referentiels$<sector>$<version>$arbres       # Decision trees
referentiels$<sector>$<version>$listes       # Code lists (membership lookups)
referentiels$<sector>$<version>$exclusions   # Exclusion rule tables
referentiels$<sector>$<version>$severite     # Severity reference data
```

### Inspecting Trees

```r
# View CMD tree structure
str(referentiels$mco$v2025$arbres$cmd, max.level = 2)

# View racine trees (one per CMD)
names(referentiels$mco$v2025$arbres$racine)  # "01", "02", ..., "28"

# View code lists
head(referentiels$mco$v2025$listes$codes)
```

### Never Modify Referentiels at Runtime

Referentiels are pre-built and immutable. To change classification rules:
1. Modify source Excel files
2. Rebuild via `data-raw/` scripts
3. Regenerate `sysdata.rda`

The only exception is passing custom `sources` to `autogrouper()` for testing.

---

## Debugging Classification Issues

### Trace a Specific Stay

```r
result <- autogrouper(.data, "mco", "v2025")

ident <- "PROBLEM_STAY"

# CMD assignment
result$data$output$grp_cmd |> filter(ident == !!ident)

# Racine assignment
result$data$output$grp_racine_all |> filter(ident == !!ident)

# Staging data (what inputs were used)
result$data$staging$fixe_staging |> filter(ident == !!ident)
result$data$staging$diag_staging |> filter(ident == !!ident)
result$data$staging$acte_staging |> filter(ident == !!ident)

# Excluded diagnoses
result$data$output$exclusion$xclu_dpdas |> filter(ident == !!ident)

# Severity chain
result$data$output$sev_niv_das_init |> filter(ident == !!ident)
result$data$output$sev_niv_max |> filter(ident == !!ident)
result$data$output$sev_effet_age |> filter(ident == !!ident)
```

### Use Stop Points (MCO only)

```r
# Stop after CMD grouping
result <- autogrouper(.data, "mco", "v2025", stop = "grp_cmd")
# Inspect result$data$output$grp_cmd without running remaining stages
```

### Golden Divergence Tracing

```r
comparison <- golden_validate(result, "mco", golden_dir)
# Find where a specific stay first diverges:
golden_trace_divergence(comparison, "PROBLEM_STAY")
```
