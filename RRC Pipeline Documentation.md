# Data Engineering & Ingestion Pipeline Documentation
## Texas Railroad Commission (RRC) Historical Production Data Pipeline
### Portfolio Project

**Analyst:** Dennis P Johnson\
**Dataset Size, Version 1:** 156,586,309 Rows (Fact Aggregated)\
**Dataset Size, Version 2:** 77,212,240 Rows (Fact Single Table)\
**Target Architecture:** Power BI Columnar VertiPaq Data Model   

### 1. Executive Summary
This document details the engineering steps taken to ingest, clean, and model the definitive historical oil and gas production backend database provided by the Railroad Commission of Texas (RRC). Due to the staggering volume of this dataset—culminating in over 156 million rows across multiple fact tables spanning over 33 years—traditional spreadsheet applications completely lock up or truncate rows natively. This data engineering pipeline establishes an optimal, reproducible framework utilizing Power Query and the VertiPaq database engine to securely store, schema-optimize, and query massive data lakes without memory exhaustion or code corruption.

Portfolio link: [Click Here](https://app.powerbi.com/view?r=eyJrIjoiMzFmZDlkNmItYjdhZC00NDVlLThlZWItYzg3MjU4Y2JlMzRhIiwidCI6IjA4NmQ3ZjlmLWNmNTktNDFhZS04YmVmLTdjNDYxZjNkYmEzNSJ9)

### 2. Ingestion Pipeline Iteration Log

* **Iteration 1: Native Web Interface Extraction**
  * *Methodology:* Attempted direct data extraction via the RRC Production Data Query (PDQ) portal across 436 pages of web results.
  * *Limitation:* Discovered the portal lacks a global 'Export All' feature. Manual screen scraping was rejected as inefficient for comprehensive big data analysis.

* **Iteration 2: Bulk Database Ingestion**
  * *Methodology:* Switched upstream strategy to leverage the raw monthly RRC database backend archives. Downloaded 16 compressed `.dsv` volumes, extracting them into a dedicated `Raw Data` directory.
  * *Limitation:* Files failed native Excel CSV parsing due to an unconventional closing curly bracket delimiter (`}`) structure utilized by state mainframes.

* **Iteration 3: Auto-Schema Detection & Data Model Failure**
  * *Methodology:* Formatted the custom `}` delimiter via Power Query and allowed the compiler to auto-assign column profiles via an initial sample block.
  * *Result:* Pipeline Failure. The stream logged over 10 million data-type errors out of 48+ million processing records before execution was terminated.
  * *Root Cause Analysis:* Historical rows deep inside the 12.6 GB master file contained unexpected text strings (`N/A`, structural characters, spaces) in purely mathematical fields. The system threw catastrophic formatting errors when trying to parse string content into numerical formats.

* **Iteration 4: Explicit Text Schema Mapping & Production Success**
  * *Methodology:* Purged the corrupted stream. Removed the automated 'Changed Type' compiler step entirely within the Advanced Editor and cast every field explicitly as Text. This shifted the load logic from evaluation to pure ingestion.
  * *Result:* CRITICAL SUCCESS. The pipeline safely processed 79,374,069 rows with exactly zero errors, loading the 12.6 GB primary matrix securely into memory.

### 3. Core Technical Implementations
* **Power Query Custom M-Code Delimiter Signature:**
  ```powerquery
  Table.SplitColumn(Source, "Column1", Splitter.SplitTextByDelimiter("}", QuoteStyle.None))
  ```
* **Infrastructural Limit Overrides:** Deployed an Online Analytical Processing (OLAP) connection model directly into Power BI's Data Model. The underlying VertiPaq engine converts flat text rows into highly compressed vertical memory segments, allowing fast aggregations of millions of rows.

### 4. Advanced Relational Architecture & Conformed Dimensions
* **Multi-Fact Integration:** The architecture utilizes a secondary transactional matrix: `OG_COUNTY_LEASE_CYCLE_DATA_TABLE` (77,212,240 rows), breaking down historical production at the intersection of county boundaries. The full model holds 156.5M+ records.
* **Many-to-Many Resolution:** Bypassed the engine limitation of connecting two 70M+ row fact tables by normalizing the schema upstream. Ingested `OG_REGULATORY_LEASE_DW_DATA_TABLE`, applied a de-duplication pipeline step in Power Query to reduce records down to 365,707 unique leases, and deployed it as a conformed master dimension.
* **Geographic Mapping & Validation:** Integrated `GP_COUNTY_DATA_TABLE` using a Power Query de-duplication step to establish a clean One-to-Many (`1:*`) relationship down to the county fact table. Verified architectural integrity via a dual-fact aggregation test, yielding 100% identical totals across both data streams.

### 5. Visual Layer Troubleshooting & DAX Optimization
* **The 1996 Filter Trap:** During initial reporting, standard iterations like `MINX` and alphabetical text sorting `MIN` natively hit an optimization wall, returning a frozen value of `1996` despite the physical presence of historical data going back to `1993` in the raw data view.
* **Root Cause Analysis:** Active, multi-fact database table relationships and background cross-filtering masks were completely hijacking the visual layer's context evaluation, causing the engine to discard records prior to 1996.
* **The Solution:** Bypassed relationship engine constraints entirely by applying a targeted calculation override that ignores visual table contexts, forcing the calculation engine to parse raw columnar strings directly:
  ```dax
  Absolute Raw Min Year = 
  CALCULATE(
      MIN(OG_LEASE_CYCLE_DATA_TABLE[CYCLE_YEAR]),
      REMOVEFILTERS()
  )
  ```

### 6. Phase Transition: Pivoting from Excel to Power BI

* **Initial Fact Table Optimization:** Attempted initial file size reduction on the primary production table (`OG_LEASE_CYCLE_DATA_TABLE`). Kept only 8 critical columns out of 32, converted text types to whole numbers for numeric values, and applied a `TRIM` function to text columns to remove whitespace.
* **Architectural Pivot:** During this initial phase, it became clear that processing multi-gigabyte Texas RRC files in Excel was highly inefficient and time-consuming beyond basic data extraction. To build a scalable solution, the project architecture was shifted entirely to Power BI to leverage its robust Power Query engine and VertiPaq compression.

### 7. Power BI Data Engineering & ETL Pipeline

* **Project Scope Redefinition:** The core deliverable was refined to build a high-performing production dashboard that seamlessly filters 79M+ rows of data by County and Operator.
* **Optimized Schema Requirements:** Out of the 16 original RRC files, the dataset was isolated down to three essential tables to establish a clean Star Schema, plus an engineered date table:
  * `OG_OPERATOR_CYCLE_DATA_TABLE.dsv`: Dimension table containing operator profiles and corporate names.
  * `GP_COUNTY_DATA_TABLE.dsv`: Dimension table containing unique county names and geographic mappings.
  * `OG_LEASE_CYCLE_DATA_TABLE.dsv`: Central Fact table containing historical monthly volumetric production data.
  * `Calendar`: A custom DAX-calculated table compressed to a monthly grain to allow for fast time-intelligence metrics.

* **Development Iterations & Troubleshooting Log:**
  * **Iteration 1 (Folder Ingestion Failure):** Initially attempted to use Power Query's Folder Data Source to ingest the `.dsv` files as a single batch. While the initial setup and folder copying took 12 minutes to pull in 12.2 GB of raw text, only the tiny 8 KB `GP_COUNTY_DATA_TABLE.dsv` file successfully loaded into the model.
  * **Iteration 2 (Root Cause Analysis):** A deep dive into the Power Query pipeline revealed that the "Combine and Transform" method failed because each RRC file possesses a completely distinct column schema. Stacking them vertically corrupted the payload, highlighting the need to ingest each file as an independent data source.
  * **Iteration 3 (Successful File Ingestion & Transformation):** Isolated the `OG_LEASE_CYCLE_DATA_TABLE.dsv` fact table and established a clean, reproducible transformation pipeline:
    * Reduced the total column footprint from 32 down to 9 essential metrics.
    * Converted high-cardinality numeric identifiers and production volumes to `Int64` whole numbers and consolidated `CYCLE_YEAR` and `CYCLE_MONTH` into a unified `Cycle_Date` column for optimal VertiPaq compression.

### 8. Model Refactoring, Spatial Key Engineering & Data Quality Audit

* **Star Schema Consolidation (`OG_COUNTY_LEASE_CYCLE_DATA_TABLE`):**
  * Replaced the initial multi-table well-completion bridge architecture with `OG_COUNTY_LEASE_CYCLE_DATA_TABLE` as the single unified fact table, eliminating referential integrity gaps between partial well completion filings and active production cycles.
  * Retained all four core hydrocarbon production streams—**Crude Oil (BBL)**, **Casinghead Gas (MCF)**, **Gas Well Gas (MCF)**, and **Condensate (BBL)**—and filtered out zero-activity rows where all four metrics equaled zero to maximize VertiPaq compression across 1993–present lease-level history.
  * Converted `OPERATOR_NO`, `COUNTY_NO`, and `DISTRICT_NO` foreign keys to `Int64` whole numbers to optimize relationship join performance.

* **Resolving the 277 vs. 254 County Spatial Mismatch:**
  * Identified that the RRC tracks 277 geographic entities (254 standard terrestrial counties plus 23 Gulf of Mexico offshore lease/state blocks), whereas the `tx counties.topojson` spatial layer contains 254 land boundaries.
  * Preserved 100% of offshore production data for accurate statewide totals while isolating terrestrial counties via `ONSHORE_ASSC_CNTY_FLAG` (`"N"` or `"Y"`) and displaying offshore zones in companion breakdown tables.

* **5-Digit FIPS Spatial Key Standardization (Resolving Camel-Case Bugs):**
  * Discovered that title-casing county names (`Text.Proper`) broke Shape Map rendering for camel-case Texas counties such as **DeWitt County** (`Dewitt` vs. `DeWitt`).
  * Engineered a standardized 5-digit federal **FIPS Code** (`"48" & Text.PadStart(Text.From([COUNTY_NO]), 3, "0")`) in `GP_COUNTY_DATA_TABLE` and bound the Shape Map visual directly to `FIPS_Code`, eliminating 100% of spelling and casing mismatches across all 254 counties.

* **Mainframe Surrogate Keys & Multi-District Anomaly Audit:**
  * Reconciled the offset between legacy mainframe surrogate IDs (`DISTRICT_NO` `1`–`14`, `20`) and true regulatory district codes (`DISTRICT_NAME` `01`–`10`, `6E`, `7B`, `7C`, `8A`, `20`) in `GP_DISTRICT_DATA_TABLE`.
  * Built a dedicated **Data Quality & Audit** hidden report page featuring a cross-referenced `FIPS_Code` exception matrix to differentiate legitimate multi-district geological overlaps (such as **DeWitt County** straddling Districts 01 and 02 along the Eagle Ford trend) from isolated historical filing miscodes (such as **Roberts County Oil Lease `26455`** reporting 13,343 BBL under District 03 in 2013).

### 9. Core DAX Measure Reference Appendix

* **Total Liquids (BBL):**
  ```dax
  Total Liquids (BBL) = 
  SUM(OG_COUNTY_LEASE_CYCLE_DATA_TABLE[CNTY_LSE_OIL_PROD_VOL]) + 
  SUM(OG_COUNTY_LEASE_CYCLE_DATA_TABLE[CNTY_LSE_COND_PROD_VOL])
  ```

* **Total Gas (MCF):**
  ```dax
  Total Gas (MCF) = 
  SUM(OG_COUNTY_LEASE_CYCLE_DATA_TABLE[CNTY_LSE_GAS_PROD_VOL]) + 
  SUM(OG_COUNTY_LEASE_CYCLE_DATA_TABLE[CNTY_LSE_CSGD_PROD_VOL])
  ```

* **Total Production (BOE — 6:1 Industry Conversion):**
  ```dax
  Total Production (BOE) = 
  [Total Liquids (BBL)] + DIVIDE([Total Gas (MCF)], 6)
  ```

* **Multi-District County Audit Flag (For Hidden Troubleshooting Page):**
  ```dax
  Districts per County = 
  CALCULATE(
      DISTINCTCOUNT(OG_COUNTY_LEASE_CYCLE_DATA_TABLE[DISTRICT_NO]),
      REMOVEFILTERS(GP_DISTRICT_DATA_TABLE)
  )
  ```

