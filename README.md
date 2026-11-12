
**Analyst:** Dennis P Johnson\
**Dataset Size, Version 1:** 156,586,309 Rows (Fact Aggregated)\
**Dataset Size, Version 2:** 77,212,240 Rows (Fact Single Table)\
**Target Architecture:** Power BI Columnar VertiPaq Data Model 

# Texas RRC Production Analytics Pipeline & Dashboard

[📊 Click Here to View the Live Interactive Power BI Dashboard](https://app.powerbi.com/view?r=eyJrIjoiMzFmZDlkNmItYjdhZC00NDVlLThlZWItYzg3MjU4Y2JlMzRhIiwidCI6IjA4NmQ3ZjlmLWNmNTktNDFhZS04YmVmLTdjNDYxZjNkYmEzNSJ9)  

## 📌 Executive Summary
This enterprise-grade portfolio project demonstrates an end-to-end data engineering and business intelligence pipeline utilizing public datasets from the **Texas Railroad Commission (RRC)**. The objective was to ingest, clean, and model **33.8 GB of raw, pipe-delimited (.dsv) text data** containing over **79 million rows** of historical upstream production records to deliver a highly responsive reporting suite optimized for county and operator benchmarking.

Faced with massive file footprints that exceeded traditional hardware processing limits, the project architecture successfully pivoted from Excel-driven extraction to a fully relational **Star Schema in Power BI**. Key engineering milestones achieved include:
* **Advanced Data Cleansing:** Resolving character encoding mismatches, header promotion anomalies, and trailing whitespace variations across 79M+ rows in Power Query.
* **Many-to-Many Resolution:** Engineering custom, unique dimension bridge tables (`OG_OPERATOR_DW_DATA_TABLE` and `GP_DISTRICT_DATA_TABLE`) to guarantee deterministic 1-to-Many filter propagation and eliminate circular dependency risks.
* **Custom Time-Intelligence Logic:** Bypassing traditional daily DAX time modifiers to construct variable-driven cumulative Year-to-Date (YTD) and Month-over-Month (MoM) growth calculations mapped to a condensed monthly integer grain (`YYYYMM`).


