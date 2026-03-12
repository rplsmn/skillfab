# Data Contracts — groupeR Input/Output Requirements

## MCO (Acute Care) Input Tables

### `fixe` — Stay header (1 row per stay)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Unique stay identifier |
| `finess` | character | **Yes** | Hospital identifier |
| `annee` | integer | Yes* | Year of discharge |
| `mois` | integer | Yes* | Month of discharge |
| `date_sortie` | date | Yes* | Discharge date |
| `age` | integer | **Yes** | Patient age in years |
| `agejour` | integer | **Yes** | Age in days (if <1 year), else 0 |
| `sexe` | integer | **Yes** | 1=Male, 2=Female |
| `poids` | integer | **Yes** | Birth weight in grams (neonates), else 0 |
| `modeentree` | integer | **Yes** | Entry mode (6, 7, 8, 0) |
| `modesortie` | integer | **Yes** | Exit mode (6, 7, 8, 9=death, 0) |
| `provenance` | integer | No | Origin code |
| `destination` | integer | No | Destination code |
| `duree` | integer | **Yes** | Length of stay in days |
| `nbseance` | integer | **Yes** | Number of sessions (outpatient) |

*Either (`annee` + `mois`) OR `date_sortie` required. groupeR derives the missing one.

### `fixe_2` — Extended stay variables (1 row per stay)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Stay identifier (matches fixe) |
| `lit_palliatif` | integer | No | Palliative care bed flag |
| `rescrit_tarif` | character | No | Tariff ruling code |
| `raac` | integer | No | Enhanced recovery flag |
| `contexte_pat` | character | No | Patient context code |
| `admin_rh` | character | No | Admin category code |
| `cat_nb_inter` | integer | No | Intervention count category |

Missing columns are added as NA. Affects GHS accuracy for specific cases.

### `diag` — Diagnoses (multiple rows per stay)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Stay identifier |
| `code` | character | **Yes** | ICD-10 diagnosis code |
| `pos_code` | character | **Yes** | `"DP"` (principal), `"DR"` (related), `"DAS"` (associated) |

Exactly one DP per stay. Zero or one DR. Zero or more DAS.

### `acte` — Procedures (multiple rows per stay)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Stay identifier |
| `code` | character | **Yes** | CCAM procedure code (7 chars) |
| `activite` | integer | **Yes** | Activity code (1, 2, 3, 4, 5) |
| `phase` | integer | No | Phase code |
| `date_acte` | date | No | Procedure date |

### `um` — Care units (multiple rows per stay)

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Stay identifier |
| `type_um` | character | **Yes** | Unit type code (e.g., "01C") |
| `duree_rum` | integer | **Yes** | Days in this unit |

---

## SMR (Rehabilitation) Input Tables

### `fixe` — Week record

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `ident_sejour` | character | **Yes** | Stay identifier |
| `age` | integer | **Yes** | Patient age |
| `sexe` | integer | **Yes** | 1=M, 2=F |
| `modeentree` | integer | **Yes** | Entry mode |
| `modesortie` | integer | **Yes** | Exit mode |
| `nbj_presence` | integer | **Yes** | Days present this week |

### `fixe_sej` — Stay summary

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_sejour` | character | **Yes** | Stay identifier |
| `duree_sejour` | integer | **Yes** | Total stay duration |
| `type_hospitalisation` | character | **Yes** | HC/HP/HTP |

### `diag` — Diagnoses

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `code` | character | **Yes** | ICD-10 code |
| `pos_code` | character | **Yes** | `"FPP"` / `"MMP"` / `"AE"` / `"DAS"` |

### `acte` — Rehabilitation acts

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident` | character | **Yes** | Week identifier |
| `code` | character | **Yes** | CSARR code |
| `nb_realisation` | integer | **Yes** | Times performed |

---

## HAD (Home Care) Input Tables

*Note: HAD support is experimental (vExp only)*

### `fixe` — Sequence record

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_seq` | character | **Yes** | Sequence identifier |
| `ident_sejour` | character | **Yes** | Stay identifier |
| `age` | integer | **Yes** | Patient age |
| `sexe` | integer | **Yes** | 1=M, 2=F |
| `duree_seq` | integer | **Yes** | Sequence duration |

### `diag` — Diagnoses

| Column | Type | Required | Description |
|--------|------|----------|-------------|
| `ident_seq` | character | **Yes** | Sequence identifier |
| `code` | character | **Yes** | ICD-10 code |
| `pos_code` | character | **Yes** | `"MPP"` / `"MPA"` / `"DAS"` |

---

## MCO Output Tables

### `grp_ghs_tot_all` — Main output

| Column | Type | Description |
|--------|------|-------------|
| `ident` | character | Stay identifier (from input) |
| `ghm` | character | Final GHM code (e.g., "05C092") |
| `ghs` | integer | Final GHS code |
| `cmd` | character | Major diagnostic category |
| `racine` | character | Root GHM (before severity) |
| `niv_sev` | integer | Severity level (1-4) |

### Intermediate outputs (for debugging)

| Table | Path | Content |
|-------|------|---------|
| `grp_cmd` | `$data$output$grp_cmd` | CMD per stay |
| `grp_racine_all` | `$data$output$grp_racine_all` | Root GHM per stay |
| `grp_ghm_all` | `$data$output$grp_ghm_all` | GHM with severity decoration |
| `sev_niv_das_init` | `$data$output$sev_niv_das_init` | Initial DAS severity levels |
| `exclusion$*` | `$data$output$exclusion$*` | Excluded diagnosis lists |

## SMR Output Tables

| Table | Path | Content |
|-------|------|---------|
| `grp_gmt_all` | `$data$output$grp_gmt_all` | GMT (financing group) |
| `grp_gme_all` | `$data$output$grp_gme_all` | GME (activity group) |

## HAD Output Tables

| Table | Path | Content |
|-------|------|---------|
| `grp_gpsl` | `$data$output$grp_gpsl` | GPSL (group/procedure/severity/burden) |

---

## Type Coercion Checklist

```r
fixe <- fixe |> mutate(
    ident = as.character(ident),
    age = as.integer(age),
    sexe = as.integer(sexe),
    duree = as.integer(duree),
    modeentree = as.integer(modeentree),
    modesortie = as.integer(modesortie)
)

diag <- diag |> mutate(
    ident = as.character(ident),
    code = as.character(code),
    pos_code = toupper(trimws(pos_code))
)
```

## Missing Data Impact

| Column | If Missing | Impact |
|--------|-----------|--------|
| Required column | **Error** | Pipeline halts |
| `poids` | Assumes 0 | Neonatal logic may fail |
| `agejour` | Assumes 0 | Neonatal logic may fail |
| `nbseance` | Assumes 0 | Session groups may fail |
| `fixe_2` columns | Added as NA | Some GHS rules may not apply |
