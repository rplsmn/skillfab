# Troubleshooting Guide - groupeR

Common errors, their causes, and solutions when using groupeR.

## Error Categories

1. [Input Validation Errors](#input-validation-errors)
2. [Backend/Connection Errors](#backendconnection-errors)
3. [Classification Errors](#classification-errors)
4. [Performance Issues](#performance-issues)
5. [Memory Errors](#memory-errors)

---

## Input Validation Errors

### Error: "Les données d'input ne sont pas du même type"

**Cause**: Mixed data backends in the input list.

```r
# ❌ This causes the error
list(
  fixe = my_dataframe,           # data.frame
  diag = tbl(con, "diag_table")  # tbl_sql
)
```

**Solution**: Ensure all tables have the same class.

```r
# Option 1: Convert everything to data.frame
list(
  fixe = my_dataframe,
  diag = collect(tbl(con, "diag_table"))
)

# Option 2: Convert everything to tbl_sql
copy_to(con, my_dataframe, "fixe_temp", temporary = TRUE)
list(
  fixe = tbl(con, "fixe_temp"),
  diag = tbl(con, "diag_table")
)
```

---

### Error: "Veuillez préciser un chemin pour le fichier duckdb de résultats"

**Cause**: Using file paths as input without specifying where to store results.

```r
# ❌ Missing path_db_results
autogrouper(
  list(fixe = "/path/to/*.parquet", ...),
  "mco", "v2025"
)
```

**Solution**: Add the `path_db_results` argument.

```r
# ✅ Correct
autogrouper(
  list(fixe = "/path/to/*.parquet", ...),
  "mco", "v2025",
  path_db_results = "/output/results.duckdb"
)
```

---

### Error: Column 'X' not found / Column 'X' does not exist

**Cause**: Missing required column in input data.

**Solution**: Check the [data-contracts.md](./data-contracts.md) for required columns. Add missing columns:

```r
# Add missing columns with defaults
fixe <- fixe |>
  mutate(
    nbseance = ifelse(is.null(nbseance), 0L, nbseance),
    poids = ifelse(is.null(poids), 0L, poids)
  )
```

---

### Error: "Version 'X' not available for type 'Y'"

**Cause**: Invalid combination of type and version.

**Valid combinations**:
| type | versions |
|------|----------|
| `"mco"` | `"v2023"`, `"v2025"`, `"v2026"` |
| `"smr"` | `"v2022"`, `"v2025"` |
| `"had"` | `"vExp"` |

**Solution**: Use a valid version for your chosen type.

```r
# ❌ Wrong
autogrouper(data, "mco", "v2022")  # v2022 not available for MCO

# ✅ Correct
autogrouper(data, "mco", "v2023")
```

---

### Error: Invalid pos_code values

**Cause**: Diagnosis position codes don't match expected values.

**Expected values by type**:
- **MCO**: `"DP"`, `"DR"`, `"DAS"`
- **SMR**: `"FPP"`, `"MMP"`, `"AE"`, `"DAS"`
- **HAD**: `"MPP"`, `"MPA"`, `"DAS"`

**Solution**: Standardize pos_code values before calling.

```r
diag <- diag |>
  mutate(
    pos_code = case_match(
      pos_code,
      c("dp", "principal", "1") ~ "DP",
      c("dr", "related", "2") ~ "DR",
      .default = "DAS"
    )
  )
```

---

## Backend/Connection Errors

### Error: "Connection is closed" / Database connection error

**Cause**: DuckDB connection was closed before accessing results.

```r
# ❌ Connection closed too early
con <- dbConnect(duckdb())
result <- autogrouper(data_list, "mco", "v2025")
dbDisconnect(con)  # ← Closed here
result$data$output$grp_ghs_tot_all  # ← Error: connection closed
```

**Solution**: Keep connection open until you've collected results.

```r
# ✅ Correct
con <- dbConnect(duckdb())

result <- autogrouper(data_list, "mco", "v2025")
final <- collect(result$data$output$grp_ghs_tot_all)  # Collect first

dbDisconnect(con)  # Now safe to close
```

---

### Error: "No such file or directory" for parquet files

**Cause**: File path pattern doesn't match any files.

**Solution**: Verify paths exist and match files.

```r
# Check if files exist
list.files("/path/to/fixe/", pattern = "\\.parquet$")

# Use absolute paths
normalizePath("/path/to/fixe/*.parquet")

# Check for typos in folder names
# Common: "fixe_2" vs "fixe2"
```

---

### Error: "Unable to connect to database"

**Cause**: DuckDB can't create/open the database file.

**Solutions**:

```r
# Check write permissions
file.access(dirname("/path/to/db.duckdb"), mode = 2)

# Try in-memory instead
con <- dbConnect(duckdb())  # No dbdir = in-memory

# Check disk space
system("df -h /path/to/")
```

---

## Classification Errors

### Problem: All stays classified as error GHM (90Z*)

**Causes**:
1. Missing or invalid DP (principal diagnosis)
2. All diagnosis codes invalid/unknown

**Diagnosis**:

```r
# Check if DP exists for all stays
data_list$diag |>
  filter(pos_code == "DP") |>
  count(ident) |>
  filter(n != 1)  # Should be empty (exactly 1 DP per stay)

# Check diagnosis code format
data_list$diag |>
  filter(!grepl("^[A-Z][0-9]{2}", code)) |>
  distinct(code)  # Should be empty or minimal
```

---

### Problem: Wrong severity level assigned

**Causes**:
1. DAS (associated diagnoses) not properly coded
2. Missing exclusion data

**Diagnosis**:

```r
# Check DAS counts
data_list$diag |>
  filter(pos_code == "DAS") |>
  count(ident, name = "n_das") |>
  summary()

# Check after classification
result$data$staging$diag_exclu_staging |>
  filter(ident == "PROBLEM_STAY") |>
  select(code, pos_code, liste, exclu)
```

---

### Problem: GHS is NA or unexpected

**Causes**:
1. Missing `fixe_2` columns needed for GHS rules
2. Missing care unit information

**Solution**: Ensure `fixe_2` has all columns (even if NA):

```r
required_cols <- c("lit_palliatif", "rescrit_tarif", "raac", 
                   "contexte_pat", "admin_rh", "cat_nb_inter")

fixe_2 <- fixe_2 |>
  bind_cols(
    setNames(
      as.list(rep(NA, length(setdiff(required_cols, names(fixe_2))))),
      setdiff(required_cols, names(fixe_2))
    )
  )
```

---

## Performance Issues

### Problem: Classification takes hours

**Causes and solutions**:

| Cause | Solution |
|-------|----------|
| Using data.frame with large data | Switch to DuckDB backend |
| DuckDB memory too low | Increase `memory_limit` |
| Single-threaded | Increase `threads` config |
| Excessive intermediate materialization | Use lazy evaluation |

```r
# Optimized configuration for large data
con <- dbConnect(
  duckdb(
    dbdir = "results.duckdb",
    config = list(
      threads = as.character(parallel::detectCores() - 2),
      memory_limit = "40GB",
      temp_directory = "/fast/ssd/tmp"  # Use fast storage for temp
    )
  )
)
```

---

### Problem: Query gets slower over time

**Cause**: DuckDB accumulating temporary objects.

**Solution**: Periodically checkpoint or restart connection.

```r
# Force checkpoint
dbExecute(con, "CHECKPOINT")

# Or use fresh connection per batch
process_batch <- function(batch_data) {
  con <- dbConnect(duckdb())
  on.exit(dbDisconnect(con, shutdown = TRUE))
  # ... process ...
}
```

---

## Memory Errors

### Error: "Out of memory" / R crashes

**Causes**:
1. Data too large for in-memory processing
2. DuckDB memory limit too high (competing with R)
3. Collecting too much data at once

**Solutions**:

```r
# 1. Reduce DuckDB memory to leave room for R
con <- dbConnect(duckdb(config = list(memory_limit = "20GB")))  # Not all RAM

# 2. Never collect full results
# ❌ Bad
all_results <- collect(result$data$output$grp_ghs_tot_all)

# ✅ Good: filter/aggregate first
summary <- result$data$output$grp_ghs_tot_all |>
  count(cmd, ghm) |>
  collect()

# 3. Export to parquet instead of collecting
dbExecute(con, "
  COPY (SELECT * FROM grp_ghs_tot_all) 
  TO 'results.parquet' (FORMAT PARQUET)
")
```

---

### Error: "Cannot allocate vector of size X"

**Cause**: R trying to allocate huge vector (usually from `collect()`).

**Solution**: Stream results or export directly.

```r
# Stream by chunks
process_results <- function(result) {
  # Get unique CMDs
  cmds <- result$data$output$grp_ghs_tot_all |>
    distinct(cmd) |>
    collect() |>
    pull(cmd)
  
  # Process each CMD separately
  for (cmd in cmds) {
    chunk <- result$data$output$grp_ghs_tot_all |>
      filter(cmd == !!cmd) |>
      collect()
    
    # Process chunk...
    write_parquet(chunk, glue::glue("results_{cmd}.parquet"))
    
    rm(chunk)
    gc()
  }
}
```

---

## Debugging Checklist

When classification results are unexpected:

1. **Check input data quality**
   ```r
   # All stays have DP?
   data_list$diag |> filter(pos_code == "DP") |> count(ident) |> filter(n != 1)
   
   # Valid diagnosis codes?
   data_list$diag |> filter(nchar(code) < 3) |> distinct(code)
   
   # Valid procedure codes?
   data_list$acte |> filter(nchar(code) != 7) |> distinct(code)
   ```

2. **Compare with expected results**
   ```r
   # If you have reference results
   comparison <- result$data$output$grp_ghs_tot_all |>
     inner_join(reference_results, by = "ident", suffix = c("_new", "_ref")) |>
     filter(ghm_new != ghm_ref)
   ```

3. **Trace specific stays**
   ```r
   # See all intermediate values
   trace_stay <- function(result, ident) {
     list(
       cmd = filter(result$data$output$grp_cmd, ident == !!ident),
       racine = filter(result$data$output$grp_racine, ident == !!ident),
       fixe = filter(result$data$staging$fixe_staging, ident == !!ident),
       diag = filter(result$data$staging$diag_staging, ident == !!ident),
       acte = filter(result$data$staging$acte_staging, ident == !!ident)
     ) |>
       map(collect)
   }
   ```

4. **Check version-specific rules**
   ```r
   # Verify referentiels loaded
   result$sources$arbres$cmd  # Should not be NULL
   result$sources$listes$codes |> head()  # Should have data
   ```

---

## Getting Help

If issues persist:

1. **Create minimal reproducible example**
   - Smallest possible data that reproduces the issue
   - Include all code to set up and run

2. **Include version info**
   ```r
   sessionInfo()
   packageVersion("groupeR")
   packageVersion("duckdb")
   ```

3. **Check GitHub issues**
   - Search existing issues first
   - Include error messages and stack traces
