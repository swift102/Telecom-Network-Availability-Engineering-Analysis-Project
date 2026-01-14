📘 Data Engineering Implementation Notes

Telecom Network Availability Platform – Microsoft Fabric

1️⃣ Purpose of This Document

This document focuses on the data engineering implementation of the platform.
It complements the main project README by detailing:

Architecture decisions

Ingestion patterns

Lakehouse design

Naming conventions

Engineering trade-offs

Planned enhancements

This README is written from a data engineer’s perspective, not a business or BI perspective.

2️⃣ Architecture Summary
High-Level Flow
Source CSV Files (GitHub)
→ Fabric Pipelines (HTTP Copy Activity)
→ Bronze Lakehouse (Raw Files)
→ Bronze Delta Table
→ Silver Lakehouse (Cleansed + SLA Logic)
→ Gold Lakehouse (Star Schema)
→ Power BI (Direct Lake)

Key Architectural Principles

Lakehouse-first (no Fabric Warehouse)

Medallion architecture (Bronze / Silver / Gold)

Schema-on-read in Bronze

Spark-based transformations in Silver and Gold

BI consumes only Gold

3️⃣ What Has Been Implemented (Current State)
✅ Ingestion (Bronze)

Source system: GitHub (HTTP)

Tooling: Fabric Pipelines – Copy Activity

Parameter-driven ingestion:

year

month

file_name

Data is ingested one logical batch (one month) per run

Files are stored in OneLake with clean partitioning:

Files/network_events/year=YYYY/month=MM/

✅ Bronze Lakehouse

Raw CSV files preserved (no transformations)

Partition-based lineage using year and month

Data consolidated into a single Delta table:

bronze_network_events

✅ Bronze Validation

Row count checks

Partition verification

Schema inspection

No data cleansing or business logic applied

4️⃣ Naming Conventions
Workspaces
WS_Telecom_Network_Availability

Lakehouses
lh_Bronze_Telecom
lh_Silver_Telecom
lh_Gold_Telecom

Pipelines
PL_Bronze_NetworkEvents_Ingest_GitHub
PL_Telecom_EndToEnd

Notebooks
NB_Bronze_Validation
NB_Silver_NetworkEvents
NB_Gold_FactNetworkAvailability

Tables

Bronze: bronze_*

Silver: silver_*

Gold: fact_*, dim_*

Consistent naming is used to clearly indicate layer, responsibility, and data domain.

5️⃣ Design Decisions & Rationale
Why No Fabric Warehouse?

Transformations are Spark-based

Direct Lake allows Power BI to query Gold tables directly

Warehouse can be added later if SQL-heavy analytics are required

Why Parameterized Pipelines?

Enables incremental ingestion

Supports replayability

Simplifies late-arriving data handling

Avoids hard-coded paths

Why Manual Batch Runs (for now)?

Keeps early phases simple and debuggable

Allows clear validation of each monthly load

Ensures correctness before automation

6️⃣ Incremental Ingestion (Planned Enhancement)
Current State

Ingestion is incremental by design

Each pipeline run processes a single month

Execution is currently manual

Planned Improvement

Future enhancements will introduce:

ForEach orchestration in Fabric Pipelines

Dynamic detection of new files/months from GitHub

Conditional ingestion of missing partitions

Integration with Silver watermark logic

This will allow the platform to operate in a fully automated incremental mode without changing the core architecture.

7️⃣ What Is Intentionally NOT Done Yet

The following are deliberately deferred:

Automated incremental orchestration

SLA enforcement logic (Silver phase)

Data quality metrics table

Deduplication and late-arriving data reconciliation

Gold star schema modeling

BI dashboards

This ensures each layer is implemented correctly and independently.

8️⃣ Next Engineering Phase
Phase 3 – Silver Transformations

Planned work includes:

Timestamp standardisation

Duration recalculation

SLA (99.9%) compliance logic

Data quality metrics

Late-arriving data handling
