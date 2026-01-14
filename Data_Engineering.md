# 📊 Data Engineering Implementation
## Telecom Network Availability Platform – Microsoft Fabric

[![Fabric](https://img.shields.io/badge/Microsoft%20Fabric-2.0-blue?style=flat-square&logo=microsoft)](https://fabric.microsoft.com)
[![Lakehouse](https://img.shields.io/badge/Architecture-Medallion-green?style=flat-square)](https://www.databricks.com/glossary/medallion-architecture)
[![Status](https://img.shields.io/badge/Phase-Bronze%20Complete-success?style=flat-square)]()

---

## 📋 Table of Contents

1. [Purpose of This Document](#-purpose-of-this-document)
2. [Architecture Overview](#️-architecture-overview)
   - [High-Level Data Flow](#high-level-data-flow)
   - [Core Architectural Principles](#core-architectural-principles)
3. [Current Implementation Status](#-current-implementation-status)
   - [Bronze Layer (Complete)](#-bronze-layer-complete)
   - [Silver Layer (Not Started)](#-silver-layer-not-started)
   - [Gold Layer (Not Started)](#-gold-layer-not-started)
4. [Naming Conventions & Standards](#-naming-conventions--standards)
5. [Design Decisions & Engineering Rationale](#-design-decisions--engineering-rationale)
6. [Incremental Ingestion Strategy](#-incremental-ingestion-strategy)
7. [Intentionally Deferred](#-intentionally-deferred-not-done-yet)
8. [Roadmap & Next Engineering Phase](#-roadmap--next-engineering-phase)
9. [Development Environment & Tools](#️-development-environment--tools)
10. [Testing Strategy](#-testing-strategy)
11. [References & Further Reading](#-references--further-reading)
12. [Related Documentation](#-related-documentation)

---

## 📌 Purpose of This Document

This document details the **data engineering implementation** of the Telecom Network Availability Analytics Platform. It complements the main project README by focusing on:

- 🏗️ **Architecture decisions** and design patterns
- 🔄 **Data ingestion** strategies and orchestration
- 🗄️ **Lakehouse design** and modeling approach
- 📏 **Naming conventions** and standards
- ⚖️ **Engineering trade-offs** and rationale
- 🚀 **Planned enhancements** and roadmap

**Target Audience:** Data engineers, platform architects, and technical reviewers evaluating implementation quality and production readiness.

---

## 🏗️ Architecture Overview

### High-Level Data Flow

```
┌─────────────────┐
│  GitHub (CSV)   │  Source System
│  Raw Data Files │  
└────────┬────────┘
         │ HTTP
         ↓
┌─────────────────────────────┐
│  Fabric Pipeline            │  Orchestration Layer
│  • Copy Activity (HTTP)     │
│  • Parameter-driven         │
│  • Partitioned ingestion    │
└────────┬────────────────────┘
         │
         ↓
┌─────────────────────────────┐
│  🥉 Bronze Lakehouse        │  Raw Layer
│  • Immutable storage        │  Schema-on-read
│  • Files + Delta tables     │  No transformations
│  • Partitioned by year/month│
└────────┬────────────────────┘
         │ PySpark
         ↓
┌─────────────────────────────┐
│  🥈 Silver Lakehouse        │  Cleansed Layer
│  • Business rules applied   │  Schema-on-write
│  • SLA calculations         │  Data quality checks
│  • Conformed dimensions     │
└────────┬────────────────────┘
         │ PySpark
         ↓
┌─────────────────────────────┐
│  🥇 Gold Lakehouse          │  Analytics Layer
│  • Star schema              │  Optimized for BI
│  • Dimensional model        │  Direct Lake ready
│  • Aggregated metrics       │
└────────┬────────────────────┘
         │ Direct Lake
         ↓
┌─────────────────────────────┐
│  Power BI                   │  Consumption Layer
│  • Network availability     │
│  • SLA compliance           │
│  • Vendor performance       │
└─────────────────────────────┘
```

### Core Architectural Principles

| Principle | Implementation | Rationale |
|-----------|---------------|-----------|
| **Lakehouse-First** | No Fabric Warehouse (currently) | Spark-native transformations, Direct Lake support |
| **Medallion Architecture** | Bronze → Silver → Gold layers | Clear separation of concerns, incremental value |
| **Schema Evolution** | Schema-on-read (Bronze) → Schema-on-write (Gold) | Flexibility early, consistency late |
| **Spark-Based Processing** | PySpark notebooks for all transformations | Scalability, performance, future-proofing |
| **BI Isolation** | Power BI consumes only Gold layer | Stable contracts, performance optimization |
| **Parameter-Driven** | Dynamic year/month/file parameters | Reusability, incremental loading support |

---

## ✅ Current Implementation Status

### 🥉 Bronze Layer (COMPLETE)

#### Data Ingestion

**Source Integration:**
- **Source System:** GitHub repository (public HTTP endpoint)
- **Ingestion Tool:** Fabric Data Pipeline with Copy Activity
- **Connection Type:** HTTP (anonymous)
- **File Format:** CSV with headers
- **Ingestion Pattern:** Parameter-driven batch loads

**Pipeline Configuration:**
```
Pipeline: pl_Bronze_NetworkEvents_Ingest_GitHub
Parameters:
  - p_year: String (e.g., "2024")
  - p_month: String (e.g., "01", "02", ...)
  - p_file_name: String (e.g., "network_availability_2024_01")

Source: 
  Base URL: https://raw.githubusercontent.com
  Relative URL: swift102/Telecom-Network-Availability-.../Data/network_availability_{year}_{month}.csv

Destination:
  Lakehouse: lh_Bronze_Telecom
  Path: Files/network_events/year={year}/month={month}/
  Format: CSV (preserved as-is)
```

**Data Partitioning Strategy:**
```
lh_Bronze_Telecom/
├── Files/
│   └── network_events/
│       └── year=2024/
│           ├── month=01/
│           │   └── network_availability_2024_01.csv
│           ├── month=02/
│           │   └── network_availability_2024_02.csv
│           ├── month=03/
│           │   └── network_availability_2024_03.csv
│           └── month=04/
│               └── network_availability_2024_04.csv
└── Tables/
    └── bronze_network_events (Delta table)
```

**Key Features:**
- ✅ **Immutability:** Raw files never modified
- ✅ **Lineage:** Partition structure preserves temporal context
- ✅ **Replayability:** Any month can be reingested independently
- ✅ **Audit trail:** Source file name preserved in metadata

---

#### Bronze Delta Table

**Table:** `bronze_network_events`

**Schema:**
```python
event_id              STRING    (NOT NULL)
site_id               STRING    (NOT NULL)
vendor                STRING
technology            STRING
province              STRING
event_timestamp       STRING    # Raw format from source
event_type            STRING
duration_minutes      INTEGER
severity              STRING
resolved              STRING
ingestion_timestamp   TIMESTAMP # Audit column
source_file           STRING    # Audit column
```

**Purpose:**
- Consolidate partitioned CSV files into queryable Delta format
- Enable downstream Silver transformations
- Maintain audit metadata for troubleshooting

**Storage Format:** Delta Lake (ACID compliance, time travel)

---

#### Bronze Validation

**Notebook:** `nb_Bronze_Validation`

**Validation Checks:**
1. **Row Count Verification**
   - Total records ingested
   - Count per partition (year/month)

2. **Schema Inspection**
   - Column presence validation
   - Data type verification
   - Null counts for critical columns

3. **Partition Validation**
   - Expected partitions exist
   - File structure matches convention

4. **Data Quality Metrics** (observational only)
   - Distinct value counts (vendors, technologies, provinces)
   - Event type distribution
   - Duration statistics (min, max, avg)

**Important:** No data cleansing or transformations occur in Bronze validation. Issues are logged but not corrected.

---

### 🥈 Silver Layer (NOT STARTED)

**Status:** 🔴 Planned for Phase 3

**Planned Components:**
- Silver Lakehouse: `lh_Silver_Telecom`
- Notebooks:
  - `nb_Silver_NetworkEvents` - Core transformation logic
  - `nb_Silver_DataQualityMetrics` - DQ framework
  - `nb_Silver_Sites` - Site dimension processing
  - `nb_Silver_Vendors` - Vendor dimension processing
  - `nb_Silver_Technologies` - Technology dimension processing

---

### 🥇 Gold Layer (NOT STARTED)

**Status:** 🔴 Planned for Phase 4

**Planned Star Schema:**

**Fact Table:**
- `fact_network_availability`

**Dimension Tables:**
- `dim_date`
- `dim_site` (SCD Type 2)
- `dim_vendor`
- `dim_technology`
- `dim_province`

---

## 📏 Naming Conventions & Standards

### Workspace Naming
```
Pattern: WS_{BusinessDomain}_{Purpose}
Example: WS_Telecom_Network_Availability
```

**Rationale:** Clear business context, avoids generic names like "test" or "dev"

---

### Lakehouse Naming
```
Pattern: lh_{Layer}_{Domain}
Examples:
  - lh_Bronze_Telecom
  - lh_Silver_Telecom
  - lh_Gold_Telecom
```

**Rationale:** Lowercase prefix `lh_` indicates lakehouse, layer prefix enables filtering/grouping

---

### Pipeline Naming
```
Pattern: pl_{Layer}_{Entity}_{Action}_{Source}
Examples:
  - pl_Bronze_NetworkEvents_Ingest_GitHub
  - pl_Telecom_EndToEnd (master orchestration)
  - pl_Silver_SLA_Calculate (future)
```

**Rationale:** Hierarchical naming shows layer, purpose, and source at a glance

---

### Notebook Naming
```
Pattern: nb_{Layer}_{Purpose}
Examples:
  - nb_Bronze_Validation
  - nb_Silver_NetworkEvents
  - nb_Gold_DimSite_SCD2
  - nb_Gold_FactNetworkAvailability
```

**Rationale:** Prefix `nb_` distinguishes from pipelines, layer indicates processing tier

---

### Dataflow Gen2 Naming
```
Pattern: df_{Layer}_{Entity}_{Purpose}
Examples:
  - df_Silver_Reference_Lookups
  - df_Silver_Vendors_Cleanse
  - df_Gold_Dimensions_Merge
```

**Rationale:** Prefix `df_` indicates Dataflow Gen2, reserved for low-code transformations

**When to use:**
- Simple reference data ingestion
- Business user-friendly transformations
- Lookup table population
- Light ETL without complex logic

**When NOT to use:**
- Complex business logic (use notebooks)
- Large-scale transformations (use Spark)
- SLA calculations or aggregations

---

### Warehouse Naming (If Implemented)
```
Pattern: wh_{Purpose}_{Domain}
Examples:
  - wh_Analytics_Telecom
  - wh_Reporting_Network
```

**Rationale:** Prefix `wh_` distinguishes from lakehouses

**Current Status:** Not implemented (lakehouse-first architecture)

**When Warehouse might be added:**
- SQL-heavy analytics requirements emerge
- Integration with external SQL tools needed
- Governance requires SQL-only access layer
- T-SQL stored procedure requirements

**Design Principle:**
- If added, warehouse would sit **alongside** Gold lakehouse, not replace it
- Warehouse would be **synchronized from** Gold Delta tables
- Power BI would continue using Direct Lake on Gold lakehouse
- Warehouse serves SQL-native tools (SSMS, Excel, Tableau, etc.)

---

### Table Naming

**Bronze Tables:**
```
Pattern: bronze_{entity}
Example: bronze_network_events
```

**Silver Tables:**
```
Pattern: silver_{entity}
Examples: 
  - silver_network_events
  - silver_data_quality_metrics
```

**Gold Tables:**
```
Pattern: {fact|dim}_{entity}
Examples:
  - fact_network_availability
  - dim_date
  - dim_site
```

**Rationale:** Layer prefix in Bronze/Silver, semantic naming (fact/dim) in Gold

---

### Parameter Naming
```
Pattern: p_{parameter_name}
Examples:
  - p_year
  - p_month
  - p_file_name
  - p_watermark_date (for incremental loading)
```

**Rationale:** Clear prefix indicates pipeline parameter, prevents collision with variables

---

### Variable Naming
```
Pattern: v_{variable_name}
Examples:
  - v_source_url
  - v_row_count
  - v_max_timestamp
```

**Rationale:** Distinguishes pipeline variables from parameters

---

### Control/Metadata Tables
```
Pattern: {layer}_control_{purpose}
Examples:
  - bronze_file_control (tracking ingested files)
  - silver_watermark (incremental processing checkpoint)
  - gold_refresh_log (fact table refresh history)
```

**Rationale:** Layer prefix + "control" suffix indicates metadata purpose

---

**Summary of Naming Standards:**

| Object Type | Prefix | Example | Use Case |
|-------------|--------|---------|----------|
| Workspace | `WS_` | `WS_Telecom_Network_Availability` | Business domain grouping |
| Lakehouse | `lh_` | `lh_Bronze_Telecom` | Data storage layer |
| Warehouse | `wh_` | `wh_Analytics_Telecom` | SQL analytics (future) |
| Pipeline | `pl_` | `pl_Bronze_NetworkEvents_Ingest_GitHub` | Orchestration |
| Notebook | `nb_` | `nb_Silver_NetworkEvents` | PySpark transformations |
| Dataflow Gen2 | `df_` | `df_Silver_Reference_Lookups` | Low-code ETL |
| Parameter | `p_` | `p_year`, `p_month` | Dynamic pipeline inputs |
| Variable | `v_` | `v_source_url` | Pipeline variables |
| Control Table | `{layer}_control_` | `bronze_file_control` | Metadata tracking |

**Consistency Principle:** All naming follows lowercase with underscores, no camelCase or spaces

---

## 🎯 Design Decisions & Engineering Rationale

### Why No Fabric Warehouse (Yet)?

**Decision:** Implement lakehouse-only architecture initially

**Rationale:**
- ✅ **Transformation engine:** All logic is Spark/PySpark-based
- ✅ **Direct Lake support:** Power BI can query Gold tables natively
- ✅ **Simplicity:** Reduces infrastructure complexity in early phases
- ✅ **Cost efficiency:** Avoids warehouse compute costs for MVP
- ⚠️ **Future consideration:** Can add Warehouse later if SQL-heavy analytics emerge

**When Warehouse might be added:**
- Complex SQL-based reporting requirements
- Integration with external SQL tools (SSMS, Tableau)
- Governance requirements for SQL-only access
- T-SQL stored procedure needs

**Architecture if Warehouse added:**
```
Gold Lakehouse (Delta Tables)
    ├──→ Direct Lake → Power BI (primary path)
    └──→ Sync → Fabric Warehouse → SQL Tools (secondary path)
```

---

### Why Parameterized Pipelines?

**Decision:** Build parameters into pipeline from day one

**Rationale:**
- ✅ **Incremental loading readiness:** Supports month-by-month ingestion
- ✅ **Replayability:** Any historical month can be reprocessed on demand
- ✅ **Backfill support:** Easy to load multiple months via orchestration
- ✅ **Avoids hard-coding:** No brittle path dependencies
- ✅ **Production pattern:** Industry standard for data pipelines

**Alternative considered:** Hard-coded monthly pipelines ❌  
**Why rejected:** Not scalable, high maintenance burden

---

### Why Manual Batch Execution (Currently)?

**Decision:** Run pipelines manually during Bronze phase implementation

**Rationale:**
- ✅ **Validation focus:** Ensures correctness before automation
- ✅ **Debugging ease:** Clear failure points, easier troubleshooting
- ✅ **Incremental complexity:** Build automation after core logic is proven
- ✅ **Learning curve:** Allows understanding of each component

**Next step:** Implement automated incremental orchestration (see Planned Enhancements)

---

### Why Partition by Year/Month?

**Decision:** Partition Bronze files using `year=YYYY/month=MM` structure

**Rationale:**
- ✅ **Query performance:** Spark partition pruning reduces scans by 90%+
- ✅ **Data management:** Easy to delete/reprocess specific time periods
- ✅ **Standard pattern:** Hive-style partitioning widely understood
- ✅ **Future-proof:** Aligns with common time-series analytics needs

**Performance Impact:**
```
Without partitioning:
  Query: SELECT * FROM events WHERE year=2024 AND month=01
  Scan: ~40 MB (all 4 months)

With partitioning:
  Query: SELECT * FROM events WHERE year=2024 AND month=01
  Scan: ~10 MB (only January partition)
  Speedup: ~4x for this dataset, scales with data growth
```

---

### Why Separate Lakehouses per Layer?

**Decision:** Three dedicated lakehouses (Bronze, Silver, Gold)

**Rationale:**
- ✅ **Separation of concerns:** Clear boundaries between raw/cleansed/curated
- ✅ **Access control:** Can grant different permissions per layer
- ✅ **Performance isolation:** Gold queries don't impact Bronze ingestion
- ✅ **Architectural clarity:** Visually enforces medallion pattern

**Alternative considered:** Single lakehouse with folder separation ❌  
**Why rejected:** Weaker governance, risk of cross-layer pollution

---

## 🔄 Incremental Ingestion Strategy

### Current State (Manual Incremental)

**How it works today:**
```
1. Data engineer manually runs pipeline
2. Passes parameters: p_year=2024, p_month=01
3. Pipeline ingests January 2024 data
4. Repeat for subsequent months
```

**Characteristics:**
- ✅ **Incremental by design:** Each run processes one logical batch (one month)
- ✅ **Idempotent:** Rerunning same parameters replaces partition (no duplicates)
- ⚠️ **Manual trigger:** Requires human intervention

---

### Planned Enhancement: Automated Incremental Orchestration

**Target state:**

```
Master Pipeline: pl_Telecom_EndToEnd
├─ Step 1: Detect unprocessed months
│  └─ Query control table: bronze_file_control
│  └─ Identify gaps: [01, 03] (if 02 and 04 exist)
│
├─ Step 2: ForEach unprocessed month
│  └─ Execute: pl_Bronze_NetworkEvents_Ingest_GitHub
│     └─ Parameters: year, month (dynamic)
│
└─ Step 3: Update control table
   └─ Record: year, month, timestamp, row_count, status
```

**Implementation Components:**

1. **Control Table:** `bronze_file_control`
   ```sql
   CREATE TABLE bronze_file_control (
     year STRING,
     month STRING,
     file_name STRING,
     processed_timestamp TIMESTAMP,
     row_count INT,
     status STRING  -- SUCCESS, FAILED, PROCESSING
   );
   ```

2. **Month Detection Logic:**
   ```python
   # Notebook: nb_Generate_Months_To_Process
   # Compares: Available months vs. Processed months
   # Output: List of months needing ingestion
   ```

3. **ForEach Orchestration:**
   ```
   ForEach Activity in Pipeline
   Items: @activity('Get_Unprocessed_Months').output.value
   └─ Execute Pipeline: pl_Bronze_NetworkEvents_Ingest_GitHub
      Parameters: 
        - p_year: @item().year
        - p_month: @item().month
   ```

**Benefits:**
- ✅ **Fully automated:** No manual parameter entry
- ✅ **Handles late-arriving data:** Backfills gaps automatically
- ✅ **Idempotent at scale:** Safe to run daily/hourly
- ✅ **Observable:** Control table provides audit trail

**Timeline:** Planned for Phase 2 completion (before moving to Silver)

---

### Watermark-Based Incremental Loading (Silver/Gold)

**Pattern:** Track last processed timestamp to identify new records

**Implementation (Silver Layer):**

1. **Watermark Table:** `silver_watermark`
   ```sql
   CREATE TABLE silver_watermark (
     table_name STRING,
     watermark_column STRING,
     watermark_value TIMESTAMP,
     last_updated TIMESTAMP
   );
   ```

2. **Incremental Query Pattern:**
   ```sql
   -- Get last watermark
   last_watermark = spark.sql("""
     SELECT watermark_value 
     FROM silver_watermark 
     WHERE table_name = 'silver_network_events'
   """).collect()[0][0]
   
   -- Process only new records
   df_incremental = spark.sql(f"""
     SELECT * FROM bronze_network_events
     WHERE ingestion_timestamp > '{last_watermark}'
   """)
   
   -- Update watermark
   new_watermark = df_incremental.agg(max("ingestion_timestamp")).collect()[0][0]
   ```

3. **Merge Pattern (Upsert):**
   ```python
   from delta.tables import DeltaTable
   
   delta_table = DeltaTable.forName(spark, "silver_network_events")
   
   delta_table.alias("target").merge(
       df_incremental.alias("source"),
       "target.event_id = source.event_id"
   ).whenMatchedUpdateAll() \
    .whenNotMatchedInsertAll() \
    .execute()
   ```

**Benefits:**
- ✅ **Efficient processing:** Only new data processed
- ✅ **Handles late arrivals:** Updates existing records
- ✅ **ACID compliance:** Delta Lake ensures consistency
- ✅ **Production pattern:** Standard incremental ETL approach

**Use Cases:**
- Silver layer transformations (process new Bronze records)
- Gold layer aggregations (update facts incrementally)
- Change Data Capture (CDC) scenarios

---

## 🚧 Intentionally Deferred (Not Done Yet)

The following capabilities are **deliberately not implemented** in the current Bronze phase:

### Deferred to Phase 3 (Silver Layer):
- ❌ SLA enforcement logic (99.9% uptime calculations)
- ❌ Data quality metrics table
- ❌ Timestamp standardization (ISO 8601 formatting)
- ❌ Duration recalculation/validation
- ❌ Late-arriving data reconciliation
- ❌ Deduplication logic
- ❌ Watermark-based incremental processing

### Deferred to Phase 4 (Gold Layer):
- ❌ Star schema dimensional model
- ❌ SCD Type 2 dimension handling
- ❌ Aggregated fact tables
- ❌ Date dimension generation

### Deferred to Phase 5 (BI Layer):
- ❌ Power BI semantic model
- ❌ Network availability dashboards
- ❌ SLA compliance reports

**Rationale for phased approach:**
- Ensures each layer is implemented correctly and independently
- Validates architecture before adding complexity
- Allows clear testing and validation at each stage
- Follows agile incremental delivery principles

---

## 🚀 Roadmap & Next Engineering Phase

### ✅ Phase 1: Bronze Ingestion (COMPLETE)
- [x] GitHub HTTP connection established
- [x] Parameterized ingestion pipeline created
- [x] Partitioned file storage implemented
- [x] Bronze Delta table created
- [x] Validation notebook developed
- [x] End-to-end ingestion tested (4 months of data)

---

### 🔄 Phase 2: Incremental Automation (IN PROGRESS)
**Target:** Fully automated incremental loading

**Deliverables:**
- [ ] Control table (`bronze_file_control`) creation
- [ ] Month detection notebook (`nb_Generate_Months_To_Process`)
- [ ] Control table update notebook (`nb_Update_Control_Table`)
- [ ] Master pipeline refactoring with ForEach orchestration
- [ ] End-to-end testing (new month scenarios, reprocessing)

**Success Criteria:**
- Pipeline automatically processes only unprocessed months
- Control table accurately tracks ingestion history
- Adding new month to GitHub triggers automatic processing
- Running pipeline multiple times produces no duplicates

**Timeline:** 1-2 weeks

---

### 🔜 Phase 3: Silver Transformations (PLANNED)
**Target:** Cleansed, business-ready data layer

**Key Transformations:**

1. **Timestamp Standardization**
   ```python
   # Convert: "2024-01-15 08:30:00" → ISO 8601 TIMESTAMP
   # Validate: Reject future dates, invalid formats
   # Add: event_date (DATE), event_hour (INT) for partitioning
   ```

2. **SLA Compliance Logic**
   ```python
   # Business Rule: 99.9% uptime target
   # Calculate:
   #   - Total minutes in month
   #   - Outage minutes per site
   #   - SLA compliance % per site
   #   - SLA breach flag (TRUE/FALSE)
   ```

3. **Data Quality Framework**
   ```sql
   Table: silver_data_quality_metrics
   Columns:
     - check_name: "null_event_id", "invalid_duration", etc.
     - check_timestamp: When check ran
     - failed_records: Count of records failing check
     - severity: CRITICAL, HIGH, MEDIUM, LOW
   ```

4. **Dimension Conformance**
   ```python
   # Sites: Standardize site_id format, geocode provinces
   # Vendors: Clean vendor names (Huawei vs. HUAWEI)
   # Technologies: Map to standard categories (4G, 5G, LTE)
   ```

5. **Watermark Implementation**
   ```python
   # Implement watermark table for incremental processing
   # Track last processed ingestion_timestamp
   # Enable efficient delta processing
   ```

**Deliverables:**
- [ ] `nb_Silver_NetworkEvents` - Core transformation logic
- [ ] `nb_Silver_DataQualityMetrics` - DQ checks
- [ ] `nb_Silver_Sites` - Site dimension
- [ ] `nb_Silver_Vendors` - Vendor dimension
- [ ] `nb_Silver_Technologies` - Technology dimension
- [ ] `silver_watermark` - Incremental processing control
- [ ] Silver Delta tables created
- [ ] Data quality dashboard (Fabric)

**Timeline:** 2-3 weeks

---

### 🔜 Phase 4: Gold Star Schema (PLANNED)
**Target:** Analytics-optimized dimensional model

**Star Schema Design:**

**Fact Table:** `fact_network_availability`
```sql
Grain: One row per site per day
Columns:
  - date_key (FK → dim_date)
  - site_key (FK → dim_site)
  - vendor_key (FK → dim_vendor)
  - technology_key (FK → dim_technology)
  - total_outage_minutes (MEASURE)
  - sla_compliance_pct (MEASURE)
  - sla_breach_flag (MEASURE)
  - event_count (MEASURE)
```

**Dimension Tables:**
- `dim_date` - Calendar dimension (generated, not sourced)
- `dim_site` - Site master with SCD Type 2
- `dim_vendor` - Vendor reference (Type 1)
- `dim_technology` - Technology types (Type 1)
- `dim_province` - Geographic hierarchy (Type 1)

**Deliverables:**
- [ ] Date dimension generator (`nb_Gold_DimDate`)
- [ ] SCD Type 2 implementation for sites
- [ ] Fact table aggregation logic (watermark-based incremental)
- [ ] Gold Delta tables created
- [ ] Data validation (referential integrity checks)

**Timeline:** 2-3 weeks

---

### 🔜 Phase 5: BI Layer (PLANNED)
**Target:** Production-ready Power BI dashboards

**Dashboards:**
1. **Network Availability Overview**
   - Nationwide SLA compliance trend
   - Top 10 sites by outages
   - Outage severity distribution

2. **Vendor Performance**
   - Availability by vendor
   - Technology comparison (4G vs 5G)
   - Incident response times

3. **Geographic Analysis**
   - Province-level heatmap
   - Regional SLA compliance
   - Urban vs rural performance

**Technical Implementation:**
- Direct Lake mode (no import)
- Row-level security (by province)
- Incremental refresh configuration

**Timeline:** 1-2 weeks

---

## 🛠️ Development Environment & Tools

### Microsoft Fabric Components Used:
- **Lakehouse:** Data storage (OneLake Delta tables)
- **Data Pipeline:** Orchestration and ingestion
- **Notebook:** PySpark transformations
- **Dataflow Gen2:** Low-code transformations (planned for reference data)
- **Direct Lake:** Future BI connectivity

### Languages & Frameworks:
- **PySpark 3.x:** Data transformations
- **SQL (Spark SQL):** Querying and validation
- **Python 3.10+:** Control logic

### External Dependencies:
- **GitHub:** Source data repository
- **Delta Lake:** Storage format (ACID transactions)

### Version Control:
- Pipeline definitions: Exported as JSON (future)
- Notebooks: Stored in Fabric workspace
- Source data: GitHub repository

---

## 🧪 Testing Strategy

### Bronze Layer Testing:

**Unit Tests (Notebook-level):**
- Schema validation passes
- Row counts match expected
- Partitions created correctly
- Audit columns populated

**Integration Tests (Pipeline-level):**
- End-to-end ingestion from GitHub succeeds
- Parameters correctly passed to copy activity
- Delta table updated after pipeline run
- Control table updated correctly (Phase 2)

**Regression Tests:**
- Reingesting same month produces identical results
- Historical data not impacted by new loads

**Test Data:**
- 4 months of real network availability data (Jan-Apr 2024)
- Multiple vendors, technologies, provinces
- Coverage of different event types and severities

---

