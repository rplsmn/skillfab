---
name: grouper-consumer
description: Use when calling the groupeR R package API 
---

## 1. Conceptual Overview

### What groupeR Does

groupeR is an R package that classifies French hospital stays into reimbursement groups (the French DRG system). Given patient data (demographics, diagnoses, procedures), it determines:

- **GHM** (Groupe Homogène de Malades): The activity classification
- **GHS** (Groupe Homogène de Séjours): The reimbursement/financing classification

### The Problem It Solves

French hospitals must classify every stay for reimbursement. The classification algorithm is defined by ATIH (government agency) and changes yearly. groupeR:

1. Encodes these classification rules as executable R/SQL code
2. Handles multiple classification versions (v2023, v2025, v2026...)
3. Supports multiple hospital sectors (acute, rehab, home care)
4. Works with both small in-memory data and large database-backed datasets

### Core Data Flow

```
Input Tables          →    groupeR Pipeline    →    Output
─────────────────────────────────────────────────────────────
fixe (stay header)    
diag (diagnoses)      →    autogrouper()       →    GHM + GHS codes
acte (procedures)          type + version           per stay
um (care units)
fixe_2 (extra vars)
```

### Mental Model

Think of groupeR as a **decision tree executor**. Classification rules are defined as:
- "If patient has diagnosis X AND age > 70 → Group A"
- "Else if procedure Y was performed → Group B"
- ...continuing through hundreds of rules

groupeR converts these rules into optimized `CASE WHEN` logic and applies them to your data.

---

## 2. API Surface

### Primary Entry Point

```r
autogrouper(.data, type, version, sources = NULL, ...)
```

This is the **only function most consumers need**.

#### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `.data` | list | Yes | Named list of input tables (see Data Contracts) |
| `type` | character | Yes | Hospital sector: `"mco"`, `"smr"`, or `"had"` |
| `version` | character | Yes | Classification version: `"v2023"`, `"v2025"`, `"v2026"` |
| `sources` | list | No | Custom referentiels (rarely needed) |
| `...` | various | No | Backend-specific options |

#### Valid Combinations

| type | Available versions |
|------|-------------------|
| `"mco"` | `"v2023"`, `"v2025"`, `"v2026"` |
| `"smr"` | `"v2022"`, `"v2025"` |
| `"had"` | `"vExp"` (experimental only) |

#### Return Value

Returns a **grouper object** (S3 class list) containing:

```r
result <- autogrouper(.data, "mco", "v2025")

# Access final results:
result$data$output$grp_ghs_tot_all  # Main output: GHM + GHS per stay

# Access intermediate results (debugging):
result$data$output$grp_cmd          # CMD classification
result$data$output$grp_racine       # Root GHM before severity
result$data$staging$*               # Prepared intermediate tables

# Access configuration:
result$opts$type                    # "mco"
result$opts$version                 # "v2025"
result$opts$con                     # DBI connection (if SQL backend)
```

### Output Table Structure

The main output `result$data$output$grp_ghs_tot_all` contains:

| Column | Type | Description |
|--------|------|-------------|
| `ident` | character | Stay identifier (from input) |
| `ghm` | character | Final GHM code (e.g., "05C092") |
| `ghs` | integer | Final GHS code |
| `cmd` | character | Major diagnostic category |
| `racine` | character | Root GHM (before severity) |
| `niv_sev` | integer | Severity level (1-4) |

---

## 3. Data Contracts

### Input Table Requirements

groupeR requires specific input tables with specific columns. **Missing required columns will cause errors.**

For detailed column specifications, see: [data-contracts.md](./data-contracts.md)

#### Quick Reference: MCO (Acute Care)

```r
.data <- list(
  fixe = <table>,      # Required: stay header (1 row per stay)
  fixe_2 = <table>,    # Required: additional stay vars
  diag = <table>,      # Required: diagnoses (multiple per stay)
  acte = <table>,      # Required: procedures (multiple per stay)
  um = <table>         # Required: care units (multiple per stay)
)
```

#### Data Types Supported

| Type | Class | When to Use |
|------|-------|-------------|
| In-memory | `data.frame` / `tibble` | Small data (<10K stays) |
| DuckDB pointer | `tbl_sql` | Large data, parquet files |
| File paths | `character` | Parquet files on disk |

### Automatic Backend Detection

groupeR detects your data type and optimizes accordingly:

```r
# In-memory: Uses R's case_when()
autogrouper(list_of_dataframes, "mco", "v2025")

# DuckDB: Pushes SQL to database
autogrouper(list_of_tbl_sql, "mco", "v2025")

# File paths: Creates DuckDB connection automatically
autogrouper(
  list(fixe = "/path/to/fixe/*.parquet", ...),
  "mco", "v2025",
  path_db_results = "/path/to/results.duckdb"  # Required for file paths
)
```

---

## 4. Integration Patterns

### Pattern A: Small In-Memory Data

Best for: Testing, small samples, simulations (<10,000 stays)

```r
library(groupeR)
library(dplyr)

# Prepare your data as data.frames
data_list <- list(
  fixe = my_fixe_df,
  fixe_2 = my_fixe2_df,
  diag = my_diag_df,
  acte = my_acte_df,
  um = my_um_df
)

# Run classification
result <- autogrouper(
  .data = data_list,
  type = "mco",
  version = "v2025"
)

# Extract results
ghm_results <- result$data$output$grp_ghs_tot_all
```

### Pattern B: Large Data with DuckDB

Best for: Production, millions of stays, parquet files

```r
library(groupeR)
library(duckdb)
library(dplyr)

# Create DuckDB connection
con <- dbConnect(
  duckdb(
    dbdir = "results.duckdb",
    config = list(threads = "8", memory_limit = "32GB")
  )
)

# Create lazy table pointers
data_list <- list(
  fixe = tbl(con, "read_parquet('/data/fixe/*.parquet')"),
  fixe_2 = tbl(con, "read_parquet('/data/fixe_2/*.parquet')"),
  diag = tbl(con, "read_parquet('/data/diag/*.parquet')"),
  acte = tbl(con, "read_parquet('/data/acte/*.parquet')"),
  um = tbl(con, "read_parquet('/data/um/*.parquet')")
)

# Run classification (SQL pushed to DuckDB)
result <- autogrouper(
  .data = data_list,
  type = "mco",
  version = "v2025"
)

# Results are also lazy - collect when needed
final <- result$data$output$grp_ghs_tot_all |> collect()

# Clean up
dbDisconnect(con)
```

### Pattern C: Direct File Paths

Best for: Simplest large-data workflow

```r
library(groupeR)

# Just provide paths - groupeR handles DuckDB setup
result <- autogrouper(
  .data = list(
    fixe = "/data/mco/fixe/*.parquet",
    fixe_2 = "/data/mco/fixe_2/*.parquet",
    diag = "/data/mco/diag/*.parquet",
    acte = "/data/mco/acte/*.parquet",
    um = "/data/mco/um/*.parquet"
  ),
  type = "mco",
  version = "v2025",
  path_db_results = "/output/groupage_results.duckdb"  # REQUIRED
)
```

---

## 5. Gotchas and Invariants

### Critical Requirements

#### 1. Column Names Must Match Exactly
groupeR expects lowercase column names matching the PMSI format:
- `ident` not `IDENT` or `id`
- `age` not `AGE` or `patient_age`
- `modeentree` not `mode_entree`

**Fix**: Rename columns before calling autogrouper.

#### 2. All Input Tables Must Have Same Backend
You cannot mix data.frames with tbl_sql pointers:

```r
# ❌ WRONG - mixed types
list(fixe = my_dataframe, diag = tbl(con, "diag"))

# ✅ CORRECT - all same type
list(fixe = tbl(con, "fixe"), diag = tbl(con, "diag"))
```

#### 3. Version Must Be Compatible with Data Year
Classification version should be >= data year. v2025 rules can group 2020-2025 data, but v2023 rules cannot properly group 2025 data.

#### 4. DuckDB Memory for Large Data
For >5M stays, configure DuckDB memory:

```r
con <- dbConnect(duckdb(config = list(memory_limit = "40GB")))
```

#### 5. Diagnosis Position Codes
The `pos_code` column in diag table must use exact values:
- `"DP"` - Principal diagnosis
- `"DR"` - Related diagnosis  
- `"DAS"` - Associated diagnosis

Not `"principal"`, `"dp"`, etc.

### Performance Considerations

| Data Size | Recommended Backend | Expected Time |
|-----------|--------------------|--------------| 
| <10K stays | data.frame | Seconds |
| 10K-1M stays | DuckDB | 1-10 minutes |
| >1M stays | DuckDB + chunking | 10-60 minutes |

For very large data, groupeR automatically chunks at 5M stays.

### Required Columns by Step

Some columns are only needed for specific steps:
- `lit_palliatif`, `rescrit_tarif`, `raac`, `contexte_pat`, `admin_rh`: Only for GHS calculation
- If missing, groupeR adds them as NA (may affect GHS accuracy)

---

## 6. What NOT to Do

### Anti-Pattern 1: Calling Individual Pipeline Functions

```r
# ❌ WRONG - Don't call internal functions directly
result <- mco_grp_cmd(.grouper)  # Internal function

# ✅ CORRECT - Use autogrouper
result <- autogrouper(.data, "mco", "v2025")
```

### Anti-Pattern 2: Modifying the Grouper Object

```r
# ❌ WRONG - Don't mutate the returned object
result$data$output$grp_ghs_tot_all <- modified_data

# ✅ CORRECT - Extract and transform separately
final <- result$data$output$grp_ghs_tot_all |> 
  left_join(my_other_data)
```

### Anti-Pattern 3: Ignoring Connection Cleanup

```r
# ❌ WRONG - Connection leak
con <- dbConnect(duckdb())
result <- autogrouper(...)
# forgot dbDisconnect(con)

# ✅ CORRECT - Always clean up
con <- dbConnect(duckdb())
withr::defer(dbDisconnect(con))
result <- autogrouper(...)
```

### Anti-Pattern 4: Using Deprecated Versions

```r
# ❌ WRONG - Old versions may be removed
autogrouper(.data, "mco", "v2020")

# ✅ CORRECT - Use supported versions
autogrouper(.data, "mco", "v2025")
```

### Anti-Pattern 5: Large Data with data.frame

```r
# ❌ WRONG - Will be very slow or OOM
big_data <- list(fixe = huge_dataframe, ...)  # 10M rows
autogrouper(big_data, "mco", "v2025")

# ✅ CORRECT - Use DuckDB backend
big_data <- list(fixe = tbl(con, "read_parquet(...)"), ...)
autogrouper(big_data, "mco", "v2025")
```

---

## 7. Error Handling

### Common Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `Les données d'input ne sont pas du même type` | Mixed backends | Ensure all tables are same type |
| `Veuillez préciser un chemin pour le fichier duckdb` | File paths without output path | Add `path_db_results` argument |
| `Column 'X' not found` | Missing required column | Check data-contracts.md |
| `Version 'X' not available` | Invalid version string | Check valid versions in API Surface |

### Defensive Coding Pattern

```r
run_groupage <- function(data_path, output_path) {
  # Validate inputs
 stopifnot(
    dir.exists(data_path),
    !file.exists(output_path) || file.access(output_path, 2) == 0
  )
  
  # Setup with cleanup
  con <- dbConnect(duckdb(dbdir = output_path))
  on.exit(dbDisconnect(con), add = TRUE)
  
  # Build data list with validation
  required_tables <- c("fixe", "fixe_2", "diag", "acte", "um")
  data_list <- lapply(required_tables, function(tbl_name) {
    path <- file.path(data_path, tbl_name, "*.parquet")
    if (length(Sys.glob(path)) == 0) {
      stop(sprintf("No parquet files found for %s at %s", tbl_name, path))
    }
    tbl(con, sprintf("read_parquet('%s')", path))
  }) |> setNames(required_tables)
  
  # Run groupage
  result <- autogrouper(
    .data = data_list,
    type = "mco",
    version = "v2025"
  )
  
  # Return collected results
  result$data$output$grp_ghs_tot_all |> collect()
}
```

---

## 8. Supplementary Files

For detailed information, consult:

- **[data-contracts.md](./data-contracts.md)**: Complete column specifications for each input table
- **[examples.md](./examples.md)**: Full working code examples for common scenarios
- **[troubleshooting.md](./troubleshooting.md)**: Detailed error resolution guide

---

## 9. Version Compatibility

| groupeR Version | R Version | DuckDB Version | Notes |
|-----------------|-----------|----------------|-------|
| 1.0.x | >= 4.1.0 | >= 0.8.0 | Current stable |

### Checking Installation

```r
# Verify groupeR is installed
packageVersion("groupeR")

# Verify required dependencies
requireNamespace("duckdb", quietly = TRUE)
requireNamespace("dbplyr", quietly = TRUE)
```

---

## Quick Checklist Before Calling autogrouper()

- [ ] All input tables present in named list
- [ ] Column names are lowercase PMSI format
- [ ] All tables have same backend (all data.frame OR all tbl_sql)
- [ ] Version string is valid for chosen type
- [ ] If using file paths, `path_db_results` is specified
- [ ] DuckDB has sufficient memory configured for data size
- [ ] Connection cleanup is handled (withr::defer or on.exit)
