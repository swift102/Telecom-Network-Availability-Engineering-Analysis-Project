# 📊 Data Engineering Implementation
## Telecom Network Availability Platform – Microsoft Fabric

[![Fabric](https://img.shields.io/badge/Microsoft%20Fabric-2.0-blue?style=flat-square&logo=microsoft)](https://fabric.microsoft.com)
[![Lakehouse](https://img.shields.io/badge/Architecture-Medallion-green?style=flat-square)](https://www.databricks.com/glossary/medallion-architecture)
[![Status](https://img.shields.io/badge/Phase-Bronze%20Complete-success?style=flat-square)]()

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

**Notebook:** `NB_Bronze_Validation`

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

### Lakehouse Naming
```
Pattern: lh_{Layer}_{Domain}
Examples:
  - lh_Bronze_Telecom
  - lh_Silver_Telecom
  - lh_Gold_Telecom
```

### Pipeline Naming
```
Pattern: pl_{Layer}_{Entity}_{Action}_{Source}
Examples:
  - pl_Bronze_NetworkEvents_Ingest_GitHub
  - pl_Telecom_EndToEnd (master orchestration)
  - pl_Silver_SLA_Calculate (future)
```

### Notebook Naming
```
Pattern: nb_{Layer}_{Purpose}
Examples:
  - nb_Bronze_Validation
  - nb_Silver_NetworkEvents
  - nb_Gold_DimSite_SCD2
  - nb_Gold_FactNetworkAvailability
```

### Table Naming

**Bronze Tables:**
```
Pattern: bronze_{entity}
Example: bronze_network_events
```

**Silver Tables:**
```
Pattern: silver_{entity}
Example: silver_network_events, silver_data_quality_metrics
```

**Gold Tables:**
```
Pattern: {fact|dim}_{entity}
Examples:
  - fact_network_availability
  - dim_date
  - dim_site
```

### Parameter Naming
```
Pattern: p_{parameter_name}
Examples:
  - p_year
  - p_month
  - p_file_name
```

**Rationale:** Clear prefixes indicate object type at a glance, reducing cognitive load and preventing naming collisions.

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
- Integration with external SQL tools
- Governance requirements for SQL-only access

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
   # Notebook: NB_Generate_Months_To_Process
   # Compares: Available months vs. Processed months
   # Output: List of months needing ingestion
   ```

3. **ForEach Orchestration:**
   ```
   ForEach Activity in Pipeline
   Items: @activity('Get_Unprocessed_Months').output.value
   └─ Execute Pipeline: PL_Bronze_NetworkEvents_Ingest_GitHub
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

## 🚧 Intentionally Deferred (Not Done Yet)

The following capabilities are **deliberately not implemented** in the current Bronze phase:

### Deferred to Phase 3 (Silver Layer):
- ❌ SLA enforcement logic (99.9% uptime calculations)
- ❌ Data quality metrics table
- ❌ Timestamp standardization (ISO 8601 formatting)
- ❌ Duration recalculation/validation
- ❌ Late-arriving data reconciliation
- ❌ Deduplication logic

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
- [ ] Month detection notebook (`NB_Generate_Months_To_Process`)
- [ ] Control table update notebook (`NB_Update_Control_Table`)
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

**Deliverables:**
- [ ] `nb_Silver_NetworkEvents` - Core transformation logic
- [ ] `nb_Silver_DataQualityMetrics` - DQ checks
- [ ] `nb_Silver_Sites` - Site dimension
- [ ] `nb_Silver_Vendors` - Vendor dimension
- [ ] `nb_Silver_Technologies` - Technology dimension
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
- [ ] Fact table aggregation logic
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

**Regression Tests:**
- Reingesting same month produces identical results
- Historical data not impacted by new loads

**Test Data:**
- 4 months of real network availability data (Jan-Apr 2024)
- ~XXX records per month
- Multiple vendors, technologies, provinces

---

## 📚 References & Further Reading

### Microsoft Fabric Documentation:
- [Lakehouse Architecture](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview)
- [Delta Lake on Fabric](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-and-delta-tables)
- [Data Pipelines](https://learn.microsoft.com/en-us/fabric/data-factory/data-factory-overview)

### Architecture Patterns:
- [Medallion Architecture (Databricks)](https://www.databricks.com/glossary/medallion-architecture)
- [Lambda Architecture](https://en.wikipedia.org/wiki/Lambda_architecture)

### Best Practices:
- [Spark Performance Tuning](https://spark.apache.org/docs/latest/tuning.html)
- [Delta Lake Best Practices](https://docs.delta.io/latest/best-practices.html)

---



## 📄 License

This implementation is part of a personal portfolio project showcasing data engineering capabilities on Microsoft Fabric.

---

## 🔗 Related Documentation

- **Project README:** High-level business context and objectives
- **Data Dictionary:** Column definitions and business rules
- **Pipeline Documentation:** Detailed parameter reference
- **Incremental Loading Plan:** Automation implementation guide

---

**Last Updated:** January 2026  
**Phase:** Bronze Complete | Silver Planned  

