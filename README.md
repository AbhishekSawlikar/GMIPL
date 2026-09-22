# Trend Check & Reconciliation: RTE & RTE_OTHER (Jul26 vs Jun26)

## 📌 Overview

This Databricks notebook (`Trend_Check_RTE_310826`) performs automated data reconciliation and validation between monthly snapshot layers:
- **Current / Target Release**: July 2026 (`Jul26`)
- **Baseline / Prior Historical Release**: June 2026 (`Jun26`)

The checks validate retail sales performance (Value in CHF '000 and Volume in '000) across two major category domains: **`RTE`** (Ready-To-Eat) and **`RTE_OTHER`**.

## 🔍 Key Notebook Sections & Code Logic

### 1. OGF Check — Conformed Layer
* **Objective:** Compares segment shares (`Global_OGF_Segment`) at each `Global_Market` level between the current conformed fact tables and baseline tables.
* **Tables Queried:**
  * Current: `glbl_cpw_prod.conformed.factretailsales`, `dimproduct`, `dimmarket`, `dimperiod`, `metadata.forex`, `glbl_cpw_prod.adhoc.period_mat_ytd`
  * Prior: `glbl_cpw_prod.history_conformed.factretailsales_Jun26`, `dimproduct_Jun26`, `dimmarket_Jun26`, `dimperiod_Jun26`, `period_mat_ytd_Jun26`
* **Filtering Conditions:**
  * `Global_Total_Mkt_Flag = 'Y'`
  * Categories: `'RTE'`, `'RTE_OTHER'`
  * Calendar years: `2022` through `2026`
  * MAT filter: `pmt.MAT_Flag IN ('MAT TY', 'MAT LY', 'MAT 2LY')`
  * Excluded local markets: Non-standard/scan markets (e.g., `'Baltics'`, `'France_Scan'`, `'Brazil_Scan'`, `'Greece_C&C'`, etc.).
* **Calculations:**
  * Uses SQL window functions `OVER (PARTITION BY gm.Global_Market)` to calculate market totals in a single pass.
  * Measures segment percentage share differences: $\text{Jul26 \%} - \text{Jun26 \%}$.
  * Appends an overall grand total row via `UNION ALL`.

---

### 2. OGF Check — Derived Layer (`Cell 4`)
* **Objective:** Validates whether the reporting metrics maintain parity after transformation into aggregated reporting index tables.
* **Tables Queried:**
  * Current: `glbl_cpw_prod.derived.retailindextotal`
  * Prior: `glbl_cpw_prod.history_conformed.retailindextotal_Jun26`
* **Logic:** Computes market segment aggregates (`Jul26_SegmentAggregates`), compares with per-database totals, and checks for discrepancy between conformed and derived outputs.

---

### 3. Manufacturer Check — Conformed & Derived Layers (`Cells 6 & 8`)
* **Objective:** Tracks manufacturer market share shifts (e.g., `KELLOGGS`, `NESTLE`, `PRIVATE LABEL`, `SANITARIUM`, `WEETABIX`) within each market between releases.
* **Key Metrics:**
  * `Sum_of_Value_000_CHF_Jul26` vs `Sum_of_Value_000_CHF_Jun26`
  * `Sum_of_Volume_000_Jul26` vs `Sum_of_Volume_000_Jun26`
  * Percentage delta on Value and Volume.

---

### 4. Switzerland Segment Shift Root-Cause Analysis
* **Trigger:** An observed variance in Switzerland's segment split:
  * `NATURALLY DELICIOUS`: $+5.10\%$ Value / $+6.59\%$ Volume
  * `SIMPLE GOODNESS`: $-5.06\%$ Value / $-6.56\%$ Volume
* **Diagnostic Queries:**
  * **Cell 10:** Identifies specific products reclassified between `dimproduct` snapshots:
    * **Product ID `25592`**: `M CLASSIC . CEREAL FLAKES CHOCO BTL 01 ER 600 G`
    * Prior Segment (`Jun26`): `SIMPLE GOODNESS`
    * Current Segment (`Jul26`): `EVERYDAY WELLNESS`
  * **Cell 12 & 13:** Isolates sales for Product `25592` to check if it caused the multi-million CHF delta. (Found zero values for this SKU, ruling it out as the root cause).
  * **Cell 14:** Aggregates absolute value/volume for Switzerland across segments:
    * `NATURALLY DELICIOUS`: Jul26 Value `151,927.50` vs Jun26 Value `130,267.99` ($\Delta +21,659.51\text{ CHF '000}$)
    * `SIMPLE GOODNESS`: Jul26 Value `146,814.06` vs Jun26 Value `168,233.16` ($\Delta -21,419.10\text{ CHF '000}$)
    * Confirms an active master data re-mapping of significant SKUs between `SIMPLE GOODNESS` and `NATURALLY DELICIOUS` within the Swiss market.

---

## 📊 Summary of Segment & Metric Definitions

| Column Name | Description |
| :--- | :--- |
| `Global_Market` / `Market` | Country market domain (e.g., Australia, Germany, Switzerland, UK). |
| `Global_OGF_Segment` | Global OGF category segment (e.g., Childhood Fun, Everyday Wellness, Naturally Delicious, Simple Goodness, Tasty Favourites). |
| `Global_Manufacturer` | Product manufacturer (e.g., Nestle, Kelloggs, Private Label). |
| `Sum_of_Value_000_CHF_Jul26` | Total sales value in thousand CHF for July 2026 MAT window. |
| `Sum_of_Volume_000_Jul26` | Total sales volume in thousand units for July 2026 MAT window. |
| `Value/Volume difference` | Release-over-release share delta: $\text{Share}_{\text{Jul26}} - \text{Share}_{\text{Jun26}}$. |

---

## 🛠️ Usage & Execution

1. Open the notebook in a Databricks workspace attached to a cluster with SQL warehouse or PySpark runtime access.
2. Ensure read permissions on catalogs/schemas:
   * `glbl_cpw_prod.conformed.*`
   * `glbl_cpw_prod.derived.*`
   * `glbl_cpw_prod.history_conformed.*`
   * `metadata.forex`
3. Execute the cells sequentially to validate that month-over-month shifts remain within expected operational tolerance limits ($\pm 0.5\%$ typical baseline, with flagged anomalies reviewed individually).

4. 
