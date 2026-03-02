# Data Contracts - groupeR Input Requirements

This document specifies the exact column requirements for each input table when using `autogrouper()`.

## MCO (Acute Care) Tables

### Table: `fixe` (Stay Header)

One row per hospital stay. Contains administrative and demographic data.

| Column | Type | Required | Description | Example |
|--------|------|----------|-------------|---------|
| `ident` | character | **Yes** | Unique stay identifier | "2025M01XXXXX" |
| `finess` | character | **Yes** | Hospital identifier | "750000001" |
| `annee` | integer | Yes* | Year of discharge | 2025 |
| `mois` | integer | Yes* | Month of discharge | 6 |
| `date_sortie` | date | Yes* | Discharge date | 2025-06-15 |
| `age` | integer | **Yes** | Patient age in years | 45 |
| `agejour` | integer | **Yes** | Patient age in days (if <1 year) | 0 |
| `sexe` | integer | **Yes** | Sex (1=M, 2=F) | 1 |
| `poids` | integer | **Yes** | Birth weight in grams (neonates) | 0 |
| `modeentree` | integer | **Yes** | Entry mode (6,7,8,0) | 8 |
| `modesortie` | integer | **Yes** | Exit mode (6,7,8,9,0) | 8 |
| `provenance` | integer | No | Origin code | 1 |
| `destination` | integer | No | Destination code | 1 |
| `duree` | integer | **Yes** | Length of stay in days | 5 |
| `nbseance` | integer | **Yes** | Number of sessions (outpatient) | 0 |

*Note: Either (`annee` + `mois`) OR `date_sortie` must be present. groupeR will derive the missing one.

### Table: `fixe_2` (Extended Stay Variables)

One row per stay. Contains variables needed for GHS calculation.

| Column | Type | Required | Description | Example |
|--------|------|----------|-------------|---------|
| `ident` | character | **Yes** | Stay identifier (matches fixe) | "2025M01XXXXX" |
| `lit_palliatif` | integer | No | Palliative care bed flag | 0 |
| `rescrit_tarif` | character | No | Tariff ruling code | NA |
| `raac` | integer | No | Enhanced recovery flag | 0 |
| `contexte_pat` | character | No | Patient context code | NA |
| `admin_rh` | character | No | Admin category code | NA |
| `cat_nb_inter` | integer | No | Intervention count category | NA |

**Note**: If columns are missing, groupeR adds them as NA. This may affect GHS accuracy for specific cases.

### Table: `diag` (Diagnoses)

Multiple rows per stay. Each diagnosis code associated with a stay.

| Column | Type | Required | Description | Example |
|--------|------|----------|-------------|---------|
| `ident` | character | **Yes** | Stay identifier | "2025M01XXXXX" |
| `code` | character | **Yes** | ICD-10 diagnosis code | "I2510" |
| `pos_code` | character | **Yes** | Position: "DP", "DR", "DAS" | "DP" |

**Position codes explained**:
- `"DP"`: Principal diagnosis (one per stay)
- `"DR"`: Related diagnosis (zero or one per stay)
- `"DAS"`: Associated diagnoses (zero to many per stay)

### Table: `acte` (Procedures)

Multiple rows per stay. Each procedure performed.

| Column | Type | Required | Description | Example |
|--------|------|----------|-------------|---------|
| `ident` | character | **Yes** | Stay identifier | "2025M01XXXXX" |
| `code` | character | **Yes** | CCAM procedure code | "YYYY001" |
| `activite` | integer | **Yes** | Activity code (1,2,3,4,5) | 1 |
| `phase` | integer | No | Phase code | 0 |
| `date_acte` | date | No | Procedure date | 2025-06-10 |

**Note**: Procedure code should be 7 characters. 8th character (activity) is separate.

### Table: `um` (Care Units)

Multiple rows per stay. Each care unit passage during the stay.

| Column | Type | Required | Description | Example |
|--------|------|----------|-------------|---------|
| `ident` | character | **Yes** | Stay identifier | "2025M01XXXXX" |
| `type_um` | character | **Yes** | Unit type code | "01C" |
| `duree_rum` | integer | **Yes** | Days in this unit | 3 |

---

## SMR (Subacute/Rehab) Tables

### Table: `fixe` (Week Record)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `ident_sejour` | character | **Yes** | Stay identifier |
| `age` | integer | **Yes** | Patient age in years |
| `sexe` | integer | **Yes** | Sex (1=M, 2=F) |
| `modeentree` | integer | **Yes** | Entry mode |
| `modesortie` | integer | **Yes** | Exit mode |
| `nbj_presence` | integer | **Yes** | Days present this week |

### Table: `fixe_sej` (Stay Summary)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_sejour` | character | **Yes** | Stay identifier |
| `duree_sejour` | integer | **Yes** | Total stay duration |
| `type_hospitalisation` | character | **Yes** | HC/HP/HTP |

### Table: `diag` (Diagnoses)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `code` | character | **Yes** | ICD-10 code |
| `pos_code` | character | **Yes** | FPP/MMP/AE/DAS |

**SMR Position codes**:
- `"FPP"`: Principal finality diagnosis
- `"MMP"`: Morbidity diagnosis
- `"AE"`: Associated etiology
- `"DAS"`: Associated diagnoses

### Table: `acte` (Rehabilitation Acts)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `code` | character | **Yes** | CSARR code |
| `nb_realisation` | integer | **Yes** | Number of times performed |

---

## HAD (Home Care) Tables

*Note: HAD support is experimental (vExp only)*

### Table: `fixe` (Sequence Record)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_seq` | character | **Yes** | Sequence identifier |
| `ident_sejour` | character | **Yes** | Stay identifier |
| `age` | integer | **Yes** | Patient age |
| `sexe` | integer | **Yes** | Sex |
| `duree_seq` | integer | **Yes** | Sequence duration |

### Table: `diag` (Diagnoses)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_seq` | character | **Yes** | Sequence identifier |
| `code` | character | **Yes** | ICD-10 code |
| `pos_code` | character | **Yes** | MPP/MPA/DAS |

---

## Column Type Coercion

groupeR expects specific types. If your data has different types, convert before calling:

```r
library(dplyr)

# Fix common type issues
fixe <- fixe |>
  mutate(
    ident = as.character(ident),
    age = as.integer(age),
    sexe = as.integer(sexe),
    duree = as.integer(duree),
    modeentree = as.integer(modeentree),
    modesortie = as.integer(modesortie)
  )

diag <- diag |>
  mutate(
    ident = as.character(ident),
    code = as.character(code),
    pos_code = toupper(trimws(pos_code))  # Ensure uppercase, no whitespace
  )
```

## Missing Data Handling

| Column | If Missing | Impact |
|--------|-----------|--------|
| Required column | Error | Cannot proceed |
| `poids` | Assumes 0 | Neonatal logic may fail |
| `agejour` | Assumes 0 | Neonatal logic may fail |
| `nbseance` | Assumes 0 | Session groups may fail |
| `fixe_2` columns | Added as NA | Some GHS rules may not apply |

## Validation Function

Use this to validate your data before calling autogrouper:

```r
validate_mco_input <- function(data_list) {
  required_tables <- c("fixe", "fixe_2", "diag", "acte", "um")
  missing_tables <- setdiff(required_tables, names(data_list))
  
  if (length(missing_tables) > 0) {
    stop("Missing tables: ", paste(missing_tables, collapse = ", "))
  }
  
  # Check fixe columns
  fixe_required <- c("ident", "age", "sexe", "modeentree", "modesortie", "duree")
  fixe_cols <- colnames(data_list$fixe)
  missing_fixe <- setdiff(fixe_required, fixe_cols)
  
  if (length(missing_fixe) > 0) {
    stop("fixe missing columns: ", paste(missing_fixe, collapse = ", "))
  }
  
  # Check diag columns
  diag_required <- c("ident", "code", "pos_code")
  diag_cols <- colnames(data_list$diag)
  missing_diag <- setdiff(diag_required, diag_cols)
  
  if (length(missing_diag) > 0) {
    stop("diag missing columns: ", paste(missing_diag, collapse = ", "))
  }
  
  # Check pos_code values
  if (inherits(data_list$diag, "data.frame")) {
    valid_pos <- c("DP", "DR", "DAS")
    actual_pos <- unique(data_list$diag$pos_code)
    invalid_pos <- setdiff(actual_pos, valid_pos)
    
    if (length(invalid_pos) > 0) {
      warning("Invalid pos_code values: ", paste(invalid_pos, collapse = ", "))
    }
  }
  
  message("✓ Input validation passed")
  invisible(TRUE)
}
```
