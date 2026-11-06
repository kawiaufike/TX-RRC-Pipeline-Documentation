# TX-RRC-Pipeline-Documentation
This project documents the steps taken to ingest a massive 12.6 GB .dsv fact table, and discovering the accompanying dimension tables that would assist in creating a practical report.

## 📐 Data Architecture & Modeling

### Schema Evolution & Pivots

During the initial extraction, transformation, and loading (ETL) phase, several critical architectural pivots were executed to optimize performance over a massive **79M+ row dataset** totaling **33.8 GB** of raw Texas Railroad Commission (RRC) data:

1. **Platform Pivot (Excel to Power BI):** Initial attempts to process multi-gigabyte files inside Excel proved highly inefficient beyond basic data inspection. The architecture was shifted completely to Power BI to leverage its robust Power Query engine and VertiPaq columnar compression.
2. **Schema Optimization:** Out of the 16 original RRC files, the dataset was isolated down to three essential source files, completely removing redundant pre-aggregated tables (e.g., `OG_COUNTY_LEASE_CYCLE_DATA_TABLE`) to save memory and eliminate circular dependency risks.
3. **Many-to-Many Resolution:** Direct connections from the raw operator and county data files to the primary fact table would have triggered highly problematic Many-to-Many relationships. To achieve a clean relational model, two dedicated dimension bridge tables were engineered to establish highly performant, deterministic connections.
