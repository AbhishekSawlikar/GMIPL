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

### 2. OGF Check — Derived Layer
* **Objective:** Validates whether the reporting metrics maintain parity after transformation into aggregated reporting index tables.
* **Tables Queried:**
  * Current: `glbl_cpw_prod.derived.retailindextotal`
  * Prior: `glbl_cpw_prod.history_conformed.retailindextotal_Jun26`
* **Logic:** Computes market segment aggregates (`Jul26_SegmentAggregates`), compares with per-database totals, and checks for discrepancy between conformed and derived outputs.

---

### 3. Manufacturer Check — Conformed & Derived Layers
* **Objective:** Tracks manufacturer market share shifts (e.g., `KELLOGGS`, `NESTLE`, `PRIVATE LABEL`, `SANITARIUM`, `WEETABIX`) within each market between releases.
* **Key Metrics:**
  * `Sum_of_Value_000_CHF_Jul26` vs `Sum_of_Value_000_CHF_Jun26`
  * `Sum_of_Volume_000_Jul26` vs `Sum_of_Volume_000_Jun26`
  * Percentage delta on Value and Volume.

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

########################################################################################################################
# Market Share & BPS Reconciliation: RTE & RTE_OTHER (July 2026 vs June 2026)

## 📌 Executive Summary

The Databricks notebook **`Share_Check_RTE_010926`** performs regression testing, variance analysis, and validation across **Conformed** and **Derived** analytical data layers. 

Its primary purpose is to compare manufacturer type market share dynamics between the **July 2026 release (`July26`)** and the **June 2026 release (`June26`)** for Ready-To-Eat cereal categories (`RTE` and `RTE_OTHER`). It calculates Year-to-Date (YTD) performance for This Year (TY) versus Last Year (LY), calculates Basis Point (BPS) movements, and verifies dimension stability across historical releases.

---

## 🧮 Core Metrics & Formulas

1. **Value Share (%):**
   $$\text{Share}_{\text{Manufacturer}} = \left( \frac{\sum \text{Value CHF}_{\text{Manufacturer}}}{\sum \text{Value CHF}_{\text{Market Total}}} \right) \times 100$$
   *Calculated separately for `YTD TY` (Year-to-Date This Year) and `YTD LY` (Year-to-Date Last Year).*

2. **Basis Points (BPS):**
   $$\text{BPS} = (\text{YTD TY Share \%} - \text{YTD LY Share \%}) \times 100$$
   *A difference of $1.00\%$ share equals $100\text{ BPS}$. Positive values indicate market share gain against last year.*

3. **Release-over-Release Share Delta:**
   $$\Delta \text{ Share}_{\text{TY}} = \text{Share}_{\text{July26 YTD TY}} - \text{Share}_{\text{June26 YTD TY}}$$
   $$\Delta \text{ Share}_{\text{LY}} = \text{Share}_{\text{July26 YTD LY}} - \text{Share}_{\text{June26 YTD LY}}$$

4. **Release-over-Release BPS Delta:**
   $$\Delta \text{BPS} = \text{BPS}_{\text{July26}} - \text{BPS}_{\text{June26}}$$

---

## 🔬 Notebook Sections & Code Explanation

### 1. Shared Check Conformed
* **Source Tables:**
  * Current (`July26`): `factretailsales`, `dimproduct`, `dimmarket`, `dimperiod`, `period_mat_ytd`, `metadata.forex`
  * Historical (`June26`): `factretailsales_Jun26`, `dimproduct_Jun26`, `dimmarket_Jun26`, `dimperiod_Jun26`, `period_mat_ytd_Jun26`, `metadata.forex`
* **Key Logic & Deduplication:**
  * **`July26_Period_Dedup` & `June26_Period_Dedup`:** Deduplicates `(LocalPeriodKey, DataProvider, Database)` entries for `ytd_flag IN ('YTD TY', 'YTD LY')` to prevent bi-monthly double counting.
  * **Database Active Filter:** June historical CTE filters out retired databases not present in the current active `period_mat_ytd`.
  * **Forex Conversion:** Local currencies are normalized to CHF using `f.Value_000 * FX.ExchangeRate`.
  * **Pivoting & Joining:** Shares are calculated using window totals per market, pivoted into side-by-side columns, and compared against June release numbers.
  * **Manufacturer Types Evaluated:** `KELLOGGS_MFR TYPE`, `NESTLE_MFR TYPE`, `OTHER BRANDED`, `PRIVATE LABEL_MFR TYPE`.

### 2. Shared Check Derived
* **Source Tables:**
  * Current (`July26`): `glbl_cpw_prod.derived.retailindextotal`
  * Historical (`June26`): `glbl_cpw_prod.history_conformed.retailindextotal_Jun26`
* **Objective:**
  * Verifies that the aggregated business reporting layer (`retailindextotal`) accurately reflects the underlying conformed facts.
  * Excludes non-comparable/distributor markets (`Colombia`, `Costa Rica`, `El Salvador`, `Estonia`, `Latvia`, `Lithuania`, `Guatemala`, `Greece_CC`, `Honduras`, `Nicaragua`, `Panamá`, `Peru`).
  * Validates parity between the calculated shares in the conformed layer and derived reporting tables.

### 3. Manufacturer Type Reclassification Audit
* **Objective:** Identifies if unexpected shifts in market shares (specifically in volatile markets such as Spain or Switzerland) are driven by product master data reassignments rather than true consumer sales changes.
* **Logic:**
  * Compares `p_cur.Global_Manufacturer_Type` against `p_old.Global_Manufacturer_Type` where `product_id` and `Local_Market` match.
  * Joins with sales volume/value to quantify sales impacted by reclassification.
* **Findings:**
  * The query returned **0 rows**, proving that manufacturer type classification remained 100% consistent for Spain and Switzerland across the June and July snapshots.

---

## 📋 Data Dictionary (Output Columns)

| Column Name | Type | Description |
| :--- | :--- | :--- |
| `Global_Market` | `STRING` | Geographic market / country name. |
| `Global_Manufacturer_Type` | `STRING` | Standardized manufacturer bucket (`KELLOGGS_MFR TYPE`, `NESTLE_MFR TYPE`, `PRIVATE LABEL_MFR TYPE`, `OTHER BRANDED`). |
| `July26_YTD TY` | `STRING (%)` | Current release Value Share for Year-to-Date This Year. |
| `July26_YTD LY` | `STRING (%)` | Current release Value Share for Year-to-Date Last Year. |
| `July26_Grand Total` | `STRING (%)` | Total value share across all evaluated periods. |
| `July26_BPS` | `NUMERIC` | Current release basis points: $(\text{TY Share} - \text{LY Share}) \times 100$. |
| `Diff. Value Share July26 vs June26 YTD TY` | `STRING (%)` | Shift in TY value share between the July and June releases. |
| `Diff. Value Share July26 vs June26 YTD LY` | `STRING (%)` | Shift in LY baseline share between releases (detects restatements). |
| `Diff. BPS July26 vs June26` | `NUMERIC` | Shift in BPS between the two release cycles. |
| `June26_YTD TY` | `STRING (%)` | Baseline June release Value Share for YTD TY. |
| `June26_YTD LY` | `STRING (%)` | Baseline June release Value Share for YTD LY. |
| `June26_Grand Total` | `STRING (%)` | Baseline June release Total Value Share. |
| `June26_BPS` | `NUMERIC` | Baseline June release basis points. |

---

## ⚙️ Execution Instructions

1. **Environment:** Execute within a Databricks SQL Warehouse or cluster configured with Spark SQL.
2. **Prerequisites:** Ensure permissions to:
   - `glbl_cpw_prod.conformed`
   - `glbl_cpw_prod.derived`
   - `glbl_cpw_prod.history_conformed`
   - `glbl_cpw_prod.adhoc`
   - `metadata.forex`
3. **Threshold Guidelines:**
   - $\lvert\Delta\text{ Share TY}\rvert \le 0.5\%$: Standard monthly adjustment window.
   - $\lvert\Delta\text{ BPS}\rvert > 50\text{ BPS}$: Flags market for deeper manufacturer SKU restatement audit.
 
 #################################################################################################################
 
 
