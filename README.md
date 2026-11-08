# Texas RRC Production Analytics Pipeline & Dashboard

[📊 Click Here to View the Live Interactive Power BI Dashboard](https://app.powerbi.com/view?r=eyJrIjoiMzFmZDlkNmItYjdhZC00NDVlLThlZWItYzg3MjU4Y2JlMzRhIiwidCI6IjA4NmQ3ZjlmLWNmNTktNDFhZS04YmVmLTdjNDYxZjNkYmEzNSJ9)  

## 📌 Executive Summary
This enterprise-grade portfolio project demonstrates an end-to-end data engineering and business intelligence pipeline utilizing public datasets from the **Texas Railroad Commission (RRC)**. The objective was to ingest, clean, and model **33.8 GB of raw, pipe-delimited (.dsv) text data** containing over **79 million rows** of historical upstream production records to deliver a highly responsive reporting suite optimized for county and operator benchmarking.

Faced with massive file footprints that exceeded traditional hardware processing limits, the project architecture successfully pivoted from Excel-driven extraction to a fully relational **Star Schema in Power BI**. Key engineering milestones achieved include:
* **Advanced Data Cleansing:** Resolving character encoding mismatches, header promotion anomalies, and trailing whitespace variations across 79M+ rows in Power Query.
* **Many-to-Many Resolution:** Engineering custom, unique dimension bridge tables (`OG_OPERATOR_DW_DATA_TABLE` and `GP_DISTRICT_DATA_TABLE`) to guarantee deterministic 1-to-Many filter propagation and eliminate circular dependency risks.
* **Custom Time-Intelligence Logic:** Bypassing traditional daily DAX time modifiers to construct variable-driven cumulative Year-to-Date (YTD) and Month-over-Month (MoM) growth calculations mapped to a condensed monthly integer grain (`YYYYMM`).

#  Update

### 2. Ingestion Pipeline Iteration Log

* **Iteration 1: Native Web Interface Extraction**
  * *Methodology:* Attempted direct data extraction via the RRC Production Data Query (PDQ) portal across 436 pages of web results.
  * *Limitation:* Discovered the portal lacks a global 'Export All' feature. Manual screen scraping was rejected as inefficient for comprehensive big data analysis.

* **Iteration 2: Bulk Database Ingestion**
  * *Methodology:* Switched upstream strategy to leverage the raw monthly RRC database backend archives. Downloaded 16 compressed .dsv volumes, extracting them into a dedicated 'Raw Data' directory.
  * *Limitation:* Files failed native Excel CSV parsing due to an unconventional closing curly bracket delimiter ('}') structure utilized by state mainframes.

* **Iteration 3: Auto-Schema Detection & Data Model Failure**
  * *Methodology:* Formatted the custom '}' delimiter via Power Query and allowed the compiler to auto-assign column profiles via an initial sample block.
  * *Result:* Pipeline Failure. The stream logged over 10 million data-type errors out of 48+ million processing records before execution was terminated.
  * *Root Cause Analysis:* Historical rows deep inside the 12.6 GB master file contained unexpected text strings ('N/A', structural characters, spaces) in purely mathematical fields. The system threw catastrophic formatting errors when trying to parse string content into numerical formats.

* **Iteration 4: Explicit Text Schema Mapping & Production Success**
  * *Methodology:* Purged the corrupted stream. Removed the automated 'Changed Type' compiler step entirely within the Advanced Editor and cast every field explicitly as Text. This shifted the load logic from evaluation to pure ingestion.
  * *Result:* CRITICAL SUCCESS. The pipeline safely processed 79,374,069 rows with exactly zero errors, loading the 12.6 GB primary matrix securely into memory.

---

### 3. Core Technical Implementations
* **Power Query Custom M-Code Delimiter Signature:**
  ```powerquery
  Table.SplitColumn(Source, "Column1", Splitter.SplitTextByDelimiter("}", QuoteStyle.None))
  ```
* **Infrastructural Limit Overrides:** Deployed an Online Analytical Processing (OLAP) connection model directly into Power BI's Data Model. The underlying VertiPaq engine converts flat text rows into highly compressed vertical memory segments, allowing fast aggregations of millions of rows.

---

### 4. Advanced Relational Architecture & Conformed Dimensions
* **Multi-Fact Integration:** The architecture utilizes a secondary transactional matrix: `OG_COUNTY_LEASE_CYCLE_DATA_TABLE` (77,212,240 rows), breaking down historical production at the intersection of county boundaries. The full model holds 156.5M+ records.
* **Many-to-Many Resolution:** Bypassed the engine limitation of connecting two 70M+ row fact tables by normalizing the schema upstream. Ingested `OG_REGULATORY_LEASE_DW_DATA_TABLE`, applied a de-duplication pipeline step in Power Query to reduce records down to 365,707 unique leases, and deployed it as a conformed master dimension.
* **Geographic Mapping & Validation:** Integrated `GP_COUNTY_DATA_TABLE` using a Power Query de-duplication step to establish a clean One-to-Many (1:*) relationship down to the county fact table. Verified architectural integrity via a dual-fact aggregation test, yielding 100% identical totals across both data streams.

---

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
---

### 6. Phase Transition: Pivoting from Excel to Power BI

* Initial Fact Table Optimization: Attempted initial file size reduction on the primary production table (OG_LEASE_CYCLE_DATA_TABLE). Kept only 8 critical columns out of 32, converted text types to whole numbers for numeric values, and applied a TRIM function to text columns to remove whitespace.
* Architectural Pivot: During this initial phase, it became clear that processing multi-gigabyte Texas RRC files in Excel was highly inefficient and time-consuming beyond basic data extraction. To build a scalable solution, the project architecture was shifted entirely to Power BI to leverage its robust Power Query engine and VertiPaq compression.

### 7. Power BI Data Engineering & ETL Pipeline

* Project Scope Redefinition: The core deliverable was refined to build a high-performing production dashboard that seamlessly filters 79M+ rows of data by County and Operator.
* Optimized Schema Requirements: Out of the 16 original RRC files, the dataset was isolated down to three essential tables to establish a clean Star Schema, plus an engineered date table:
	* OG_OPERATOR_CYCLE_DATA_TABLE.dsv: Dimension table containing operator profiles and corporate names.
	* GP_COUNTY_DATA_TABLE.dsv: Dimension table containing unique county names and geographic mappings.
	* OG_LEASE_CYCLE_DATA_TABLE.dsv: Central Fact table containing historical monthly volumetric production data.
	* Calendar: A custom DAX-calculated table compressed to a monthly grain to allow for fast time-intelligence metrics.

* Development Iterations & Troubleshooting Log

  * Iteration 1 (Folder Ingestion Failure): Initially attempted to use Power Query's Folder Data Source to ingest the .dsv files as a single batch. While the initial setup and folder copying took 12 minutes to pull in 12.2 GB (presumably) of raw text, only the tiny 8 KB GP_COUNTY_DATA_TABLE.dsv file successfully loaded into the model.
  * Iteration 2 (Root Cause Analysis): A deep dive into the Power Query pipeline revealed that the "Combine and Transform" method failed because each RRC file possesses a completely distinct column schema. Stacking them vertically corrupted the payload, highlighting the need to ingest each file as an independent data source.
  * Iteration 3 (Successful File Ingestion & Transformation): Isolated the OG_LEASE_CYCLE_DATA_TABLE.dsv fact table and established a clean, reproducible transformation pipeline:
  	* Reduced the total column footprint from 32 down to 9 essential metrics.
  	* Removed the automatic, default "Changed Type" step generated by Power BI to prevent early column breakage.
  	* Promoted the first row to headers to correctly identify columns across the entire dataset.
  	* Applied a TRIM operation to the four primary key columns to strip away trailing whitespaces.
  	* Converted the cleaned text columns into a Whole Number data type to finalize the integration keys.
  	* Time taken for OG_LEASE_CYCLE_DATA_TABLE to become usable, one 1 and 2 minutes.


## 📊 Analytical DAX Measures

To drive business logic and prevent data inflation, explicit DAX measures were constructed rather than relying on default implicit column summaries. 

### 1. Volumetric Base Measures
Aggregates the raw production metrics streamed from the primary fact table.

```dax
Total Oil (Bbl) = 
SUM( 'OG_LEASE_CYCLE_DATA_TABLE'[LEASE_BBL_PROD_VOL] )
```

```dax
Total Gas Well Gas (Mcf) = 
SUM( 'OG_LEASE_CYCLE_DATA_TABLE'[LEASE_MCF_PROD_VOL] )
```

```dax
Total Casinghead Gas (Mcf) = 
SUM( 'OG_LEASE_CYCLE_DATA_TABLE'[LEASE_CASINGHEAD_PROD_VOL] )
```

```dax
Total Gas Combined (Mcf) = 
[Total Gas Well Gas (Mcf)] + [Total Casinghead Gas (Mcf)]
```

### 2. Standardized Energy Metrics (BOE)
Normalizes asset evaluation across oil-centric and gas-centric basins utilizing the industry-standard 6:1 thermal energy conversion factor.

```dax
Total BOE = 
[Total Oil (Bbl)] + DIVIDE( [Total Gas Combined (Mcf)], 6, 0 )
```

### 3. Custom Time-Intelligence Analytics
Because the data schema operates at a condensed monthly integer grain (`YYYYMM`) rather than a continuous daily date pattern, standard time-intelligence modifiers are unsupported. Custom structural filters were engineered using variables to evaluate periods dynamically.

#### Year-to-Date (YTD) Cumulative Volume
```dax
BOE YTD = 
VAR _SelectedYear = SELECTEDVALUE( 'Calendar'[Year] )
VAR _SelectedMonth = SELECTEDVALUE( 'Calendar'[Month No] )
RETURN
CALCULATE(
    [Total BOE],
    REMOVEFILTERS( 'Calendar' ),
    'Calendar'[Year] = _SelectedYear,
    'Calendar'[Month No] <= _SelectedMonth
)
```

#### Month-over-Month (MoM) Growth Percentage
```dax
BOE MoM % Change = 
VAR _CurrentYearMonth = SELECTEDVALUE( 'Calendar'[Year Month] )
VAR _PreviousYearMonth = 
    CALCULATE(
        MAX( 'Calendar'[Year Month] ),
        REMOVEFILTERS( 'Calendar' ),
        'Calendar'[Year Month] < _CurrentYearMonth
    )
VAR _CurrentBOE = [Total BOE]
VAR _PreviousBOE = 
    CALCULATE(
        [Total BOE],
        REMOVEFILTERS( 'Calendar' ),
        'Calendar'[Year Month] = _PreviousYearMonth
    )
RETURN
DIVIDE( _CurrentBOE - _PreviousBOE, _PreviousBOE, 0 )
```

### 4. Competitive Benchmarking (Dynamic Ranking)
Allows visuals to dynamically rank operators or counties based on volumetric size, actively trimming out-of-scope context nodes.

```dax
Operator Rank by BOE = 
IF(
    ISINSCOPE( 'OG_OPERATOR_DW_DATA_TABLE'[OPERATOR_NAME] ),
    RANKX(
        ALLSELECTED( 'OG_OPERATOR_DW_DATA_TABLE' ),
        [Total BOE],
        ,
        DESC
    ),
    BLANK()
)
```

### The Relational Star Schema

The final optimized model is configured as a **Star Schema** utilizing single and bi-directional filter context propagation to ensure flawless granularity slicing:

* **Fact Table:** `OG_LEASE_CYCLE_DATA_TABLE` (Houses historical monthly volumetric oil and gas data).
* **Operator Dimension:** `OG_OPERATOR_DW_DATA_TABLE` (Transformed to contain strictly unique operator names and keys, connecting via a 1-to-Many relationship with bi-directional cross-filtering to optimize user slicers).
* **Geographic Dimension:** `GP_COUNTY_DATA_TABLE` (Links through the `GP_DISTRICT_DATA_TABLE` bridge to flow geographic filters down to the lease level).
* **Temporal Dimension:** `Calendar` (A custom monthly grain DAX table supporting time-intelligence logic).

## 🐛 Known Issues & Future Optimization Pipeline

While the core reporting milestone and Star Schema architecture have been successfully delivered, the following edge-case bugs have been identified and are actively queued for optimization:

*   **Slicer Lag:** Noticeable 1.5-second caching latency occurs inside the Operator dropdown visual when a global year filter is dropped. *Planned Fix: Implement cross-report drill-through paths to minimize initial visual query parsing.*
*   **Geographic Nulls:** A tiny subset of historical RRC lease lines map to an outdated district key, returning blank polygons on the Texas Shape Map. *Planned Fix: Write a conditional fallback loop in Power Query to reassign orphaned leases by coordinate matching.*
*  **Filter Propagation Block (Regional & County Insights)**: Slicing by District_Name triggers correct geographic boundaries on the map visual, but fails to segment the underlying matrix rows, returning identical regional summaries for all counties.
*Planned Fix: Revise the DAX measures using KEEPFILTERS or adjust the physical relationship cardinality to ensure the granular county filter context is not overwritten by the high-level district selection.*

## 🔄 Production Optimization & Bug Fixes (Update: Nov 2026)

Following the initial delivery milestone, a full data-model and visual-layer audit was conducted to resolve filter-context bugs, eliminate model bloat, and optimize report performance:

1. **Resolved County-Level Filter Context Inflation & Shape Map Mapping:**
   * **Issue:** Because the core production fact table (`OG_LEASE_CYCLE_DATA_TABLE`) lacks a native `COUNTY_CODE`, county visuals inherited parent `DISTRICT_NO` totals—repeating identical regional volumes across every county row—while custom Shape Map polygons failed to match unformatted county keys.
   * **Fix:** Engineered a normalized star-schema bridge using `OG_WELL_COMPLETION` keyed on a standardized 3-part `Lease_Composite_Key` (`DISTRICT_NO` + `OIL_GAS_CODE` + `LEASE_NO`) to link `GP_COUNTY_DATA` directly to lease-level production via strict single-direction `1:*` relationships. Standardized 5-digit `Texas_FIPS` codes (`48` + `COUNTY_CODE`) to bind all 254 Texas county polygons accurately.

2. **Time-Series Schema & Date Dimension Normalization:**
   * **Issue:** Plotting raw `CYCLE_YEAR_MONTH` integers (`201912` → `202001`) on a continuous X-axis produced artificial annual "stair-step" jumps and unformatted numeric labels.
   * **Fix:** Parsed `CYCLE_YEAR_MONTH` into a true `Cycle_Date` field in Power Query and connected it to a dedicated Calendar dimension table, enabling continuous chronological scaling and accurate time-intelligence calculations.

3. **VertiPaq Memory & `.pbix` Footprint Optimization:**
   * **Issue:** Ingesting raw denormalized strings and zero-volume historical cycles inflated `.pbix` memory consumption.
   * **Fix:** Stripped redundant text attributes from the fact table into deduplicated warehouse dimensions (`OG_OPERATOR_DW`, `OG_FIELD_DW`, `GP_COUNTY_DATA`) and filtered out zero-production cycles (`Oil = Cond = Gas = Casinghead = 0`) at ingestion, substantially reducing model size with zero loss of analytical granularity.

4. **UI/UX Slicer Refinement:**
   * Collapsed the vertical Operator slicer into a search-enabled dropdown, reclaiming ~20% of canvas real estate for core volumetric bar charts and scatter visuals.




