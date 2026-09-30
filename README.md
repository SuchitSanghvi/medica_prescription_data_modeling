# Member Health and Prescription use data modeling

**Modeling Medical and Prescription Drug Use to Enable Exploration of Members' Health Status**

A dbt project built on DuckDB that transforms healthcare claims, CCS clinical reference data, and prescription fills into a dimensional model for population health analysis.

---

## Business Context

Oscar Health collects premiums and pays out medical costs. When members stay healthier, costs stay lower than premiums — that's the core business model. For the data team, this means identifying which members need attention before small problems become expensive ones.

This model surfaces two views into a member's health: what they were diagnosed with, and what medications they are actually taking. The gap between those two signals is where the most clinically useful questions live:

- A member diagnosed with a chronic condition who never fills a prescription may not be receiving necessary care.
- A member filling medications across six drug classes may have complex, poorly-managed needs.
- A sudden spike in new diagnoses within a CCS category across the population may signal an opportunity for a prevention program.

---

## Source Data

| Table | Rows | Grain | Description |
|---|---|---|---|
| `claim_lines` | ~1.9M | Member + date + ICD-10 code | Diagnosis events. One visit can produce multiple rows if multiple codes were recorded. |
| `prescription_drugs` | ~3M | Member + date + NDC code | Prescription fills, with a drug taxonomy hierarchy (category / group / class). |
| `ccs` | Reference | ICD-10 code | CCS lookup mapping each ICD-10 code to three levels of clinical categories (e.g., I21.0 → "Acute myocardial infarction" → "Coronary artery disease" → "Diseases of the circulatory system"). |

Data covers 200,000 members over 3 years.

**Formatting note:** `claim_lines` stores ICD-10 codes with dots (`N92.6`); the CCS table stores them without (`N926`). This is reconciled in the staging layer.

---

## Data Quality

Issues found during exploratory analysis, all handled in staging with a flag-and-preserve approach — no rows are silently dropped:

| Issue | Count | Handling |
|---|---|---|
| Null service dates | 24 rows | Flagged `invalid_date_of_service = 1` |
| Date of 1899 (Excel epoch artifact) | 1 row | Flagged `invalid_date_of_service = 1` |
| Sentinel / invalid diagnosis codes (`999.99`, `KHD.X`, `000`) | ~1,700 rows | Flagged `invalid_diagnosis_code = 1` |
| ICD-10 code with embedded stray comma (`L02.,821`) | 1 row | Cleaned via `REPLACE` in staging |
| Valid ICD-10 codes with no CCS match | 217 codes | Kept via `LEFT JOIN`; CCS fields default to `'Unknown'` |

Staging tables preserve every row for audit purposes. Marts filter on the flag columns.

---

## Dimensional Model

Two factless fact tables and four conformed dimensions:

```
fct_diagnosis ──────┐
                    ├── dim_member
fct_prescription ───┘
        │
        ├── dim_date
        │
fct_diagnosis ── dim_diagnosis (ICD-10 + CCS hierarchy)
        │
fct_prescription ── dim_drug (NDC + drug taxonomy)
```

**Factless fact tables** — both fact tables have no financial or quantity measures because the source data has none. The row itself is the event; analysts count rows to measure volume.

**Conformed dimensions** — `dim_member` and `dim_date` are shared across both fact tables, enabling cross-domain analysis: comparing diagnosis timing against fill timing for the same member, or finding members who appear in one fact table but not the other.

### Dimension tables

| Model | Grain | Notes |
|---|---|---|
| `dim_member` | One row per member | Stub containing only `member_id`. Demographics and enrollment data would populate this in production. |
| `dim_date` | One row per distinct service date | Includes month, quarter, and year extracts for time-series analysis. |
| `dim_diagnosis` | One row per ICD-10 code | Enriched with full CCS hierarchy via `LEFT JOIN`. Supports slicing diagnoses at any level of clinical specificity. |
| `dim_drug` | One row per NDC code | Deduplicated via `QUALIFY ROW_NUMBER()`. Carries drug category, group, and class taxonomy. |

### Fact tables

| Model | Grain | Foreign keys |
|---|---|---|
| `fct_diagnosis` | One row per member + date + ICD-10 code | `member_id`, `date_of_service`, `diagnosis_code` |
| `fct_prescription_filled` | One row per member + date + NDC code | `member_id`, `date_of_service`, `drug_code` |

---

## Model Layers

### Staging (`models/staging/`)

One model per source. Renames columns to consistent conventions, casts types, cleans ICD-10 formatting, and flags invalid rows. Contains no business logic — all exclusion decisions are explicit boolean flags.

| Model | Source | Key transformations |
|---|---|---|
| `stg_claim_lines` | `raw.claim_lines` | Strips dots from ICD-10 codes; flags invalid dates and sentinel diagnosis codes |
| `stg_prescription_drugs` | `raw.prescription_drugs` | Renames NDC to `drug_code`; flags invalid dates |
| `stg_ccs` | `raw.ccs` | Renames columns; coalesces NULL CCS levels to `'Unknown'` |

### Marts (`models/marts/`)

Dimensional model tables materialized as tables. Filter staging flags, join dimensions, and deduplicate where needed.

---

## Semantic Layer Metrics

### `diagnosis_without_treatment_rate`

Percentage of diagnosed members who have no prescription fill activity. A proxy for care gaps.

**Calculation:** `COUNT(DISTINCT member_id in fct_diagnosis WHERE member_id NOT IN fct_prescription_filled)` ÷ `COUNT(DISTINCT member_id in fct_diagnosis)` × 100

**Useful dimensions:** `ccs_level_1/2/3`, `year`

**Limitation:** Measures absence of *any* prescription, not condition-specific adherence. True PDC (Proportion of Days Covered) requires `days_supply` data and a drug-to-condition crosswalk not present in this dataset.

---

### `new_condition_incidence`

Count of members receiving a first-ever diagnosis within a clinical category during a time period. Each member is counted only once, in the period of their earliest diagnosis for that condition.

**Calculation:** For each `member_id + ccs_level_1`, identify `MIN(date_of_service)` from `fct_diagnosis` joined to `dim_diagnosis`. `COUNT(DISTINCT member_id)` grouped by the time period of that first date.

**Useful dimensions:** `ccs_level_1/2/3`, `year`, `quarter`

**Business use:** A spike in new diabetes diagnoses in a quarter could trigger proactive care management outreach before those members become high-cost.

---

## Design Decisions

- **Two separate fact tables** because diagnosis events and prescription fills happen independently and at different times. Combining them would require lossy joins or introduce large amounts of nulls.
- **Natural keys as primary keys** (`member_id`, `diagnosis_code`, `drug_code`, `date_of_service`) because they're stable and unique for this dataset. Surrogate keys would be warranted in production with SCD Type 2.
- **Flag-and-preserve in staging** rather than silent filtering, so every exclusion is explicit and auditable in the marts layer.
- **SCD Type 1 on all dimensions** because the source data has no historical attributes to track. `dim_member` would use Type 2 in production.
- **Stub `dim_member`** included because every fact table FK should point to a dimension, and the model is designed for attributes Oscar has elsewhere even if this dataset doesn't contain them.

---

## Future Enhancements

**Drug-to-condition crosswalk** — the biggest current gap. A reference table mapping drug classes to the conditions they treat (e.g., Biguanides → Type 2 Diabetes) would enable condition-specific care gap analysis: "show members with a diabetes diagnosis who have not filled any diabetes medication within 30 days." That is the foundation of a real PDC calculation.

**Member demographics and enrollment data** — would populate `dim_member` and enable cohort analysis by age, plan type, and geography.

**`days_supply` on prescription fills** — required for true medication adherence calculations.

**Provider and facility data** — would enable care fragmentation analysis.

---

## Prerequisites

- Python 3.9+
- dbt-fusion or `dbt-duckdb`: `pip install dbt-duckdb`

---

## Setup

Set the environment variable pointing to the folder containing the three source CSVs:

```bash
export RAW_DATA_PATH="/path/to/csv/folder"
```

Or add it to a `.env` file and run `source .env` before building.

---

## Run

```bash
# Build all models and run all tests (data tests + unit tests)
dbt build

# Run unit tests only
dbt test --select "stg_claim_lines,dim_drug,test_type:unit"

# Generate and serve docs
dbt compile --write-catalog
# Download index.html from dbt-core and open target/ with a local server
cd target && python3 -m http.server 8080
# Then open http://localhost:8080
```

---

## Repository

Full dbt project: https://github.com/SuchitSanghvi/ae_casestudy_oscarhealth

Interactive schema diagram: https://dbdiagram.io/d/ae-casestudy-ER-6a201d3ad2fbd72c4d41ac27
