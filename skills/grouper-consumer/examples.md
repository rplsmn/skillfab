# Code Examples - groupeR Usage Patterns

Complete, runnable examples for common groupeR use cases.

## Example 1: Basic In-Memory Classification

Classify a small sample of stays loaded in memory.

```r
library(groupeR)
library(dplyr)

# ─────────────────────────────────────────────────────────────
# Prepare sample data (normally you'd load from files)
# ─────────────────────────────────────────────────────────────

fixe <- tibble(
  ident = c("SEJ001", "SEJ002", "SEJ003"),
  finess = "750000001",
  annee = 2025L,
  mois = 1L,
  age = c(45L, 72L, 0L),
  agejour = c(0L, 0L, 15L),  # 15 days old for neonate
  sexe = c(1L, 2L, 1L),
  poids = c(0L, 0L, 3200L),  # Birth weight for neonate
  modeentree = c(8L, 8L, 7L),
  modesortie = c(8L, 9L, 8L),  # 9 = death
  provenance = c(1L, 1L, 5L),
  destination = c(1L, 0L, 1L),
  duree = c(3L, 12L, 5L),
  nbseance = c(0L, 0L, 0L)
)

fixe_2 <- tibble(
  ident = c("SEJ001", "SEJ002", "SEJ003"),
  lit_palliatif = c(0L, 1L, 0L),
  rescrit_tarif = NA_character_,
  raac = c(0L, 0L, 0L),
  contexte_pat = NA_character_,
  admin_rh = NA_character_,
  cat_nb_inter = NA_integer_
)

diag <- tibble(
  ident = c("SEJ001", "SEJ001", "SEJ002", "SEJ002", "SEJ003"),
  code = c("I2510", "E119", "C349", "J189", "P073"),
  pos_code = c("DP", "DAS", "DP", "DAS", "DP")
)

acte <- tibble(
  ident = c("SEJ001", "SEJ002"),
  code = c("DBLA001", "GFFA004"),
  activite = c(1L, 1L)
)

um <- tibble(
  ident = c("SEJ001", "SEJ002", "SEJ002", "SEJ003"),
  type_um = c("01C", "17C", "01C", "15C"),
  duree_rum = c(3L, 8L, 4L, 5L)
)

# ─────────────────────────────────────────────────────────────
# Run classification
# ─────────────────────────────────────────────────────────────

data_list <- list(
  fixe = fixe,
  fixe_2 = fixe_2,
  diag = diag,
  acte = acte,
  um = um
)

result <- autogrouper(
  .data = data_list,
  type = "mco",
  version = "v2025"
)

# ─────────────────────────────────────────────────────────────
# Extract and view results
# ─────────────────────────────────────────────────────────────

result$data$output$grp_ghs_tot_all |>
  select(ident, cmd, racine, ghm, ghs)
```

## Example 2: Large Data with Parquet Files

Process millions of stays from parquet files using DuckDB.

```r
library(groupeR)
library(duckdb)
library(dplyr)
library(DBI)

# ─────────────────────────────────────────────────────────────
# Configuration
# ─────────────────────────────────────────────────────────────

data_path <- "/partage/pmsi/mco/2024"
output_db <- "/results/groupage_mco_2024.duckdb"

# ─────────────────────────────────────────────────────────────
# Set up DuckDB with appropriate resources
# ─────────────────────────────────────────────────────────────

con <- dbConnect(
  duckdb(
    dbdir = output_db,
    config = list(
      threads = "16",
      memory_limit = "40GB"
    )
  )
)

# Ensure cleanup on exit
withr::defer(dbDisconnect(con, shutdown = TRUE))

# ─────────────────────────────────────────────────────────────
# Create lazy table pointers
# ─────────────────────────────────────────────────────────────

data_list <- list(
  fixe = tbl(con, glue::glue("read_parquet('{data_path}/fixe/*.parquet')")),
  fixe_2 = tbl(con, glue::glue("read_parquet('{data_path}/fixe_2/*.parquet')")),
  diag = tbl(con, glue::glue("read_parquet('{data_path}/diag/*.parquet')")),
  acte = tbl(con, glue::glue("read_parquet('{data_path}/acte/*.parquet')")),
  um = tbl(con, glue::glue("read_parquet('{data_path}/um/*.parquet')"))
)

# ─────────────────────────────────────────────────────────────
# Run classification
# ─────────────────────────────────────────────────────────────

cli::cli_alert_info("Starting classification...")
tictoc::tic("Total time")

result <- autogrouper(
  .data = data_list,
  type = "mco",
  version = "v2025"
)

tictoc::toc()

# ─────────────────────────────────────────────────────────────
# Export results to parquet
# ─────────────────────────────────────────────────────────────

output_query <- dbplyr::sql_render(result$data$output$grp_ghs_tot_all)

dbExecute(con, glue::glue("
  COPY ({output_query}) 
  TO '/results/ghm_ghs_2024.parquet' 
  (FORMAT PARQUET, COMPRESSION ZSTD)
"))

cli::cli_alert_success("Results exported to parquet")
```

## Example 3: Multiple Years Processing

Process multiple years with the same classification version.

```r
library(groupeR)
library(duckdb)
library(dplyr)
library(purrr)

# ─────────────────────────────────────────────────────────────
# Configuration
# ─────────────────────────────────────────────────────────────

years <- 2020:2024
base_path <- "/partage/pmsi/mco"
version <- "v2025"  # Use latest version for historical comparison

# ─────────────────────────────────────────────────────────────
# Process each year
# ─────────────────────────────────────────────────────────────

process_year <- function(year) {
  cli::cli_h2("Processing {year}")
  
  data_path <- file.path(base_path, year)
  output_db <- glue::glue("/results/groupage_{year}.duckdb")
  
  con <- dbConnect(duckdb(dbdir = output_db))
  on.exit(dbDisconnect(con, shutdown = TRUE))
  
  data_list <- list(
    fixe = tbl(con, glue::glue("read_parquet('{data_path}/fixe/*.parquet')")),
    fixe_2 = tbl(con, glue::glue("read_parquet('{data_path}/fixe_2/*.parquet')")),
    diag = tbl(con, glue::glue("read_parquet('{data_path}/diag/*.parquet')")),
    acte = tbl(con, glue::glue("read_parquet('{data_path}/acte/*.parquet')")),
    um = tbl(con, glue::glue("read_parquet('{data_path}/um/*.parquet')"))
  )
  
  result <- autogrouper(
    .data = data_list,
    type = "mco",
    version = version
  )
  
  # Return summary stats
  result$data$output$grp_ghs_tot_all |>
    count(cmd) |>
    collect() |>
    mutate(year = year)
}

# Run all years
summaries <- map(years, process_year) |>
  list_rbind()

# View cross-year comparison
summaries |>
  pivot_wider(names_from = year, values_from = n) |>
  arrange(cmd)
```

## Example 4: Comparing Two Classification Versions

Compare how the same stays are classified in different versions.

```r
library(groupeR)
library(duckdb)
library(dplyr)

# ─────────────────────────────────────────────────────────────
# Setup
# ─────────────────────────────────────────────────────────────

con <- dbConnect(duckdb())
withr::defer(dbDisconnect(con))

data_path <- "/partage/pmsi/mco/2024"

data_list <- list(
  fixe = tbl(con, glue::glue("read_parquet('{data_path}/fixe/*.parquet')")),
  fixe_2 = tbl(con, glue::glue("read_parquet('{data_path}/fixe_2/*.parquet')")),
  diag = tbl(con, glue::glue("read_parquet('{data_path}/diag/*.parquet')")),
  acte = tbl(con, glue::glue("read_parquet('{data_path}/acte/*.parquet')")),
  um = tbl(con, glue::glue("read_parquet('{data_path}/um/*.parquet')"))
)

# ─────────────────────────────────────────────────────────────
# Run both versions
# ─────────────────────────────────────────────────────────────

result_v2025 <- autogrouper(data_list, "mco", "v2025")
result_v2026 <- autogrouper(data_list, "mco", "v2026")

# ─────────────────────────────────────────────────────────────
# Compare results
# ─────────────────────────────────────────────────────────────

comparison <- result_v2025$data$output$grp_ghs_tot_all |>
  select(ident, ghm_v2025 = ghm, ghs_v2025 = ghs) |>
  inner_join(
    result_v2026$data$output$grp_ghs_tot_all |>
      select(ident, ghm_v2026 = ghm, ghs_v2026 = ghs),
    by = "ident"
  ) |>
  mutate(
    ghm_changed = ghm_v2025 != ghm_v2026,
    ghs_changed = ghs_v2025 != ghs_v2026
  ) |>
  collect()

# Summary of changes
comparison |>
  summarise(
    total_stays = n(),
    ghm_changes = sum(ghm_changed),
    ghs_changes = sum(ghs_changed),
    pct_ghm_change = mean(ghm_changed) * 100,
    pct_ghs_change = mean(ghs_changed) * 100
  )

# Detailed changes
comparison |>
  filter(ghm_changed) |>
  count(ghm_v2025, ghm_v2026, sort = TRUE) |>
  head(20)
```

## Example 5: SMR (Rehabilitation) Classification

Classify rehabilitation stays.

```r
library(groupeR)
library(duckdb)
library(dplyr)

# ─────────────────────────────────────────────────────────────
# SMR data has different structure
# ─────────────────────────────────────────────────────────────

con <- dbConnect(duckdb())
withr::defer(dbDisconnect(con))

data_path <- "/partage/pmsi/smr/2024"

data_list <- list(
  fixe = tbl(con, glue::glue("read_parquet('{data_path}/rhs/*.parquet')")),
  fixe_sej = tbl(con, glue::glue("read_parquet('{data_path}/ssrha/*.parquet')")),
  diag = tbl(con, glue::glue("read_parquet('{data_path}/diag/*.parquet')")),
  acte = tbl(con, glue::glue("read_parquet('{data_path}/acte_csarr/*.parquet')"))
)

# ─────────────────────────────────────────────────────────────
# Run SMR classification
# ─────────────────────────────────────────────────────────────

result <- autogrouper(
  .data = data_list,
  type = "smr",
  version = "v2025"
)

# SMR output structure differs from MCO
# Primary outputs are GME (activity) and GMT (financing)
result$data$output$grp_gmt_all |>
  head() |>
  collect()
```

## Example 6: Testing with Simulated Data

Create synthetic data to test classification logic.

```r
library(groupeR)
library(dplyr)

# ─────────────────────────────────────────────────────────────
# Generate synthetic stays with known expected results
# ─────────────────────────────────────────────────────────────

create_test_stay <- function(
  ident,
  dp_code,
  das_codes = character(0),
  acte_codes = character(0),
  age = 50L,
  duree = 3L
) {
  list(
    fixe = tibble(
      ident = ident,
      finess = "TEST00001",
      annee = 2025L, mois = 1L,
      age = age, agejour = 0L,
      sexe = 1L, poids = 0L,
      modeentree = 8L, modesortie = 8L,
      provenance = 1L, destination = 1L,
      duree = duree, nbseance = 0L
    ),
    fixe_2 = tibble(
      ident = ident,
      lit_palliatif = 0L,
      rescrit_tarif = NA_character_,
      raac = 0L,
      contexte_pat = NA_character_,
      admin_rh = NA_character_,
      cat_nb_inter = NA_integer_
    ),
    diag = tibble(
      ident = ident,
      code = c(dp_code, das_codes),
      pos_code = c("DP", rep("DAS", length(das_codes)))
    ),
    acte = if (length(acte_codes) > 0) {
      tibble(
        ident = ident,
        code = acte_codes,
        activite = 1L
      )
    } else {
      tibble(ident = character(), code = character(), activite = integer())
    },
    um = tibble(
      ident = ident,
      type_um = "01C",
      duree_rum = duree
    )
  )
}

# ─────────────────────────────────────────────────────────────
# Create test cases
# ─────────────────────────────────────────────────────────────

test_cases <- list(
  # Cardiac stay with intervention
  create_test_stay("TEST001", dp_code = "I2510", acte_codes = "DBLA001"),
  # Simple pneumonia
  create_test_stay("TEST002", dp_code = "J189", duree = 5L),
  # Neonate
  create_test_stay("TEST003", dp_code = "P073", age = 0L, duree = 10L)
)

# Combine into single dataset
combined <- list(
  fixe = bind_rows(map(test_cases, "fixe")),
  fixe_2 = bind_rows(map(test_cases, "fixe_2")),
  diag = bind_rows(map(test_cases, "diag")),
  acte = bind_rows(map(test_cases, "acte")),
  um = bind_rows(map(test_cases, "um"))
)

# ─────────────────────────────────────────────────────────────
# Run and verify
# ─────────────────────────────────────────────────────────────

result <- autogrouper(combined, "mco", "v2025")

result$data$output$grp_ghs_tot_all |>
  select(ident, cmd, racine, ghm)
```

## Example 7: Debugging Classification Issues

When a stay classifies unexpectedly, inspect intermediate steps.

```r
library(groupeR)
library(dplyr)

# Run classification
result <- autogrouper(data_list, "mco", "v2025")

# ─────────────────────────────────────────────────────────────
# Inspect a specific stay through the pipeline
# ─────────────────────────────────────────────────────────────

problem_ident <- "PROBLEM_STAY_123"

# Check CMD assignment
result$data$output$grp_cmd |>
  filter(ident == problem_ident)

# Check racine assignment
result$data$output$grp_racine |>
  filter(ident == problem_ident)

# Check severity calculation
result$data$staging$fixe_staging |>
  filter(ident == problem_ident) |>
  select(ident, age, duree, contains("niv"), contains("sev"))

# Check what diagnoses were considered
result$data$staging$diag_staging |>
  filter(ident == problem_ident) |>
  select(ident, code, pos_code, liste)

# Check what procedures were considered
result$data$staging$acte_staging |>
  filter(ident == problem_ident) |>
  select(ident, code, liste, acte_classant)

# Check exclusions applied
result$data$staging$diag_exclu_staging |>
  filter(ident == problem_ident)
```
