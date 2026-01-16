# Response to Data 2.0 Management Document
## A Case for an Open, Composable Lakehouse Architecture

**Author:** [DBA/Data Engineer]  
**Date:** January 2026  
**Subject:** Technical Review & Alternative Architecture Proposal  
**Version:** 3.0

---

## 1. Executive Summary

The Data 2.0 document correctly identifies the need for a dedicated **OLAP/analytics platform** to complement the existing operational systems (OLTP). The current situation—delivering analytics as a byproduct of KWS via Select*Plus and stored queries—is unsustainable.

**Key clarification:** We are NOT replacing operational databases. We are adding a unified analytics platform that:
- Ingests data FROM operational systems (Sybase today, AlloyDB/PostgreSQL tomorrow)
- Provides a dedicated environment for analytics, BI, and data science
- Enables secure data sharing with hospitals and third parties

However, **the proposed BigQuery + Looker solution creates significant vendor lock-in**. This response proposes a **true Open Lakehouse Architecture** built on:

- **Open table format** (Apache Iceberg) for portable storage
- **Open catalog** as the single point of governance
- **Composable components** where each layer can be swapped independently
- **Lakehouse manager** to present a unified view to all consumers
- **Credential vending** for secure, scalable data sharing

**Important principle:** This proposal does NOT recommend replacing one vendor lock-in with another. The architecture must be **composable**—each component should be replaceable without rebuilding the entire stack.

---

## 2. Issues Identified in the Data 2.0 Document

### 2.1 Vendor Lock-in Concerns

| Claim in Document | Reality |
|-------------------|---------|
| "BigQuery supports open standards like Iceberg" | BigQuery's BigLake can read/write Iceberg, but with performance overhead. Native BigQuery format remains proprietary. |
| "LookML provides a semantic layer" | LookML is 100% proprietary to Looker. Zero portability. |
| "BigQuery Omni enables multi-cloud" | Omni requires BigQuery licensing everywhere—still locked to Google. |
| "Pay-as-you-go is transparent" | Per-TB-scanned pricing is unpredictable. Hospitals cannot forecast costs. |

### 2.2 BigQuery + Iceberg: The Efficiency Question

The document mentions Iceberg support, but let's be clear about what BigQuery actually offers:

| BigQuery Iceberg Mode | What It Means | Efficiency Concern |
|-----------------------|---------------|-------------------|
| **BigLake external tables** | Read-only access to Iceberg on GCS | Query performance depends on file layout; no native optimization |
| **BigLake Managed Tables (Iceberg)** | Read/write Iceberg tables | Write overhead vs. native; BigQuery becomes "just another engine" |
| **Native BigQuery tables** | Proprietary format | Fast, but completely locked in |

**The fundamental question:** If we're using Iceberg anyway, why pay BigQuery premium pricing when other engines can query the same data?

### 2.3 Critical Missing: Governance Architecture

The document scatters governance across multiple systems:
- Access control in BigQuery IAM
- Semantic definitions in LookML
- Audit in Cloud Audit Logs
- Data quality... unclear

**A proper lakehouse architecture centralizes governance in the CATALOG.**

### 2.4 Critical Missing: Data Sharing Architecture

The document mentions "data sharing" but lacks a concrete architecture for **credential vending**—the mechanism that allows:
- Hospitals to query ONLY their own data
- Fine-grained access control without copying data
- Audit trails of who accessed what
- Scalable onboarding of new consumers

### 2.5 Other Gaps

| Gap | Impact |
|-----|--------|
| **No explicit data quality strategy** | Where does validation happen? How are issues tracked? |
| **No schema evolution strategy** | Source changes break pipelines |
| **No disaster recovery** | What if GCP region fails? |
| **No cost allocation model** | How exactly are hospitals charged? |

---

## 3. Proposed Alternative: Open Lakehouse Architecture

### 3.1 Core Principles

| Principle | Meaning |
|-----------|---------|
| **OLTP remains untouched** | Operational DBs (Sybase → AlloyDB) continue serving KWS |
| **Open table format** | All analytics data stored in Apache Iceberg |
| **Catalog-centric governance** | ALL governance flows through the catalog |
| **Composable architecture** | Each component replaceable independently |
| **Unified view for consumers** | Lakehouse manager abstracts complexity |
| **No new vendor lock-in** | Avoid replacing Google lock-in with another vendor |

### 3.2 The Composable Lakehouse: Component Roles

Before showing the architecture, let's be clear about what each layer DOES and WHY it must be swappable:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     COMPOSABLE LAKEHOUSE PRINCIPLES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Each layer has a ROLE. The role is fixed. The IMPLEMENTATION is not.      │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  LAYER              │  ROLE                    │  SWAPPABLE?         │   │
│   ├─────────────────────┼──────────────────────────┼─────────────────────┤   │
│   │  Storage            │  Persist data durably    │  Yes (GCS/S3/ADLS)  │   │
│   │  Table Format       │  Organize data openly    │  Yes (Iceberg/Delta)│   │
│   │  Catalog            │  Govern everything       │  Yes (see options)  │   │
│   │  Query Engine       │  Execute queries         │  Yes (many options) │   │
│   │  Transformation     │  Build data products     │  Yes (dbt/Spark)    │   │
│   │  Lakehouse Manager  │  Unify consumer access   │  Yes (see options)  │   │
│   │  BI / Consumption   │  Visualize & analyze     │  Yes (many options) │   │
│   └─────────────────────┴──────────────────────────┴─────────────────────┘   │
│                                                                              │
│   If any component creates lock-in, it violates the architecture.           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONSUMPTION LAYER                               │
│                     (User's choice - no lock-in here)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   HOSPITALS / EXTERNAL                    INTERNAL NEXUZHEALTH               │
│   ────────────────────                    ────────────────────               │
│                                                                              │
│   Commercial BI:          Open Source BI:       Data Science:                │
│   • Power BI              • Apache Superset     • Jupyter/Python             │
│   • Tableau               • Metabase            • R Studio                   │
│   • Qlik Sense            • Lightdash           • Spark notebooks            │
│   • Looker                • Redash                                           │
│                                                                              │
│   Custom Applications:    APIs:                                              │
│   • Hospital portals      • REST / GraphQL                                   │
│   • Embedded analytics    • JDBC / ODBC                                      │
│                                                                              │
│   ════════════════════════════════════════════════════════════════════════   │
│                    ALL access via CREDENTIAL VENDING                         │
│                    (scoped, audited, time-limited)                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                           LAKEHOUSE MANAGER                                  │
│              (Presents UNIFIED VIEW to all consumers)                        │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   PURPOSE: Abstract the complexity of the underlying data platform.          │
│   Users see ONE interface, not multiple disconnected systems.                │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   WHAT THE USER SEES:              WHAT'S UNDERNEATH:                │   │
│   │   ───────────────────              ──────────────────                │   │
│   │                                                                      │   │
│   │   "Show me patient visits         • Query routed to best engine     │   │
│   │    for Hospital X in 2024"        • Credentials checked             │   │
│   │                                   • Row-level security applied      │   │
│   │           │                       • Query optimized                 │   │
│   │           ▼                       • Results cached if appropriate   │   │
│   │                                   • Usage logged for billing        │   │
│   │   ┌─────────────┐                 • Audit trail recorded            │   │
│   │   │   RESULTS   │                                                   │   │
│   │   └─────────────┘                                                   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   CAPABILITIES PROVIDED:                                                     │
│   ──────────────────────                                                     │
│   • Unified SQL interface         • Query federation across engines         │
│   • Semantic layer (metrics)      • Caching / acceleration                  │
│   • Data sharing / vending        • Cost tracking per consumer              │
│                                                                              │
│   IMPLEMENTATION OPTIONS (choose based on needs, avoid lock-in):            │
│   ──────────────────────────────────────────────────────────────            │
│   • Dremio           - Full platform, good for unified experience           │
│   • Starburst        - Trino-based, strong federation                       │
│   • Trino (OSS)      - Open source, requires more assembly                  │
│   • Custom layer     - Build on catalog + query engine                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                    CATALOG: THE GOVERNANCE HUB                               │
│            (Single source of truth for ALL metadata & policies)              │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   The CATALOG is where governance LIVES. Not scattered across tools.         │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     CATALOG RESPONSIBILITIES                         │   │
│   ├──────────────────┬──────────────────────────────────────────────────┤   │
│   │                  │                                                   │   │
│   │  SCHEMA          │  • Table definitions (columns, types)            │   │
│   │  MANAGEMENT      │  • Schema versioning & evolution                 │   │
│   │                  │  • Partition information                         │   │
│   │                  │                                                   │   │
│   ├──────────────────┼──────────────────────────────────────────────────┤   │
│   │                  │                                                   │   │
│   │  ACCESS          │  • Who can access what tables                    │   │
│   │  CONTROL         │  • Row-level security policies                   │   │
│   │                  │  • Column masking rules                          │   │
│   │                  │  • Credential vending configuration              │   │
│   │                  │                                                   │   │
│   ├──────────────────┼──────────────────────────────────────────────────┤   │
│   │                  │                                                   │   │
│   │  DATA            │  • Quality metrics (freshness, completeness)     │   │
│   │  QUALITY         │  • Quality check results (from dbt/GE)           │   │
│   │  TRACKING        │  • Data contracts & SLAs                         │   │
│   │                  │  • Anomaly alerts                                │   │
│   │                  │                                                   │   │
│   ├──────────────────┼──────────────────────────────────────────────────┤   │
│   │                  │                                                   │   │
│   │  LINEAGE &       │  • Where did this data come from?                │   │
│   │  DISCOVERY       │  • What transformations were applied?            │   │
│   │                  │  • Who uses this table?                          │   │
│   │                  │  • Search & discovery for users                  │   │
│   │                  │                                                   │   │
│   ├──────────────────┼──────────────────────────────────────────────────┤   │
│   │                  │                                                   │   │
│   │  AUDIT &         │  • Who queried what, when                        │   │
│   │  COMPLIANCE      │  • Access denied logs                            │   │
│   │                  │  • Compliance reporting (GDPR, etc.)             │   │
│   │                  │                                                   │   │
│   └──────────────────┴──────────────────────────────────────────────────┘   │
│                                                                              │
│   CATALOG IMPLEMENTATION OPTIONS:                                            │
│   ───────────────────────────────                                            │
│                                                                              │
│   ┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐   │
│   │   Nessie    │ Apache      │  DataHub    │   OpenMeta  │   Project   │   │
│   │   (Dremio)  │ Polaris     │             │   data      │   Nessie    │   │
│   │             │ (incubating)│             │             │   (OSS)     │   │
│   ├─────────────┼─────────────┼─────────────┼─────────────┼─────────────┤   │
│   │ Iceberg-    │ Iceberg     │ Metadata    │ Metadata    │ Git-like    │   │
│   │ native,     │ REST        │ platform,   │ platform,   │ versioning  │   │
│   │ Git-like    │ standard    │ strong      │ open source │ for data    │   │
│   │ versioning  │ (emerging)  │ lineage     │ governance  │             │   │
│   └─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘   │
│                                                                              │
│   NOTE: "Unity Catalog" is Databricks' catalog. While they open-sourced     │
│   parts of it, it remains primarily designed for the Databricks ecosystem.  │
│   It's a CATALOG, not a full lakehouse manager.                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                           QUERY / COMPUTE ENGINES                            │
│                    (Multiple engines, same data, pick per use case)          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Because data is in OPEN FORMAT (Iceberg), ANY engine can query it:        │
│                                                                              │
│   ┌─────────────────┬─────────────────┬─────────────────┬───────────────┐   │
│   │   INTERACTIVE   │   BATCH / ETL   │   STREAMING     │  LIGHTWEIGHT  │   │
│   │   ANALYTICS     │   HEAVY LIFT    │   REAL-TIME     │  LOCAL/EDGE   │   │
│   ├─────────────────┼─────────────────┼─────────────────┼───────────────┤   │
│   │ • Trino         │ • Apache Spark  │ • Apache Flink  │ • DuckDB      │   │
│   │ • Dremio Sonar  │ • Dataproc      │ • Spark Stream  │ • Polars      │   │
│   │ • Starburst     │ • EMR           │ • Kafka + Flink │               │   │
│   │ • BigQuery*     │ • Databricks    │                 │               │   │
│   │ • Snowflake*    │                 │                 │               │   │
│   └─────────────────┴─────────────────┴─────────────────┴───────────────┘   │
│                                                                              │
│   * BigQuery and Snowflake CAN query Iceberg tables.                        │
│     They become options, not requirements.                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                          TRANSFORMATION LAYER                                │
│                        (Where DATA QUALITY happens)                          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   DATA QUALITY is enforced HERE, at transformation time.                     │
│   Results are REPORTED to the CATALOG for visibility.                        │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │   BRONZE (Raw)          SILVER (Cleansed)        GOLD (Analytics)   │   │
│   │   ────────────          ─────────────────        ─────────────────   │   │
│   │                                                                      │   │
│   │   ┌───────────┐         ┌───────────┐           ┌───────────┐       │   │
│   │   │ Raw CDC   │   DQ    │ Validated │    DQ     │  Star or  │       │   │
│   │   │ data      │ ──────► │ cleansed  │  ───────► │ Snowflake │       │   │
│   │   │ as-is     │ checks  │ conformed │  checks   │  Schema   │       │   │
│   │   └───────────┘         └───────────┘           └───────────┘       │   │
│   │                                                                      │   │
│   │        │                      │                       │              │   │
│   │        ▼                      ▼                       ▼              │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │              DATA QUALITY CHECKS AT EACH STAGE              │   │   │
│   │   ├─────────────────────────────────────────────────────────────┤   │   │
│   │   │                                                              │   │   │
│   │   │  BRONZE → SILVER:              SILVER → GOLD:                │   │   │
│   │   │  • Schema validation           • Business rule validation    │   │   │
│   │   │  • Null checks                 • Referential integrity       │   │   │
│   │   │  • Duplicate detection         • Aggregation accuracy        │   │   │
│   │   │  • Freshness check             • Dimensional conformance     │   │   │
│   │   │                                                              │   │   │
│   │   │  Tools: dbt tests, Great Expectations, Soda                  │   │   │
│   │   │                                                              │   │   │
│   │   │  Results → Published to CATALOG for visibility               │   │   │
│   │   │                                                              │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   GOLD LAYER: DIMENSIONAL MODELING                                           │
│   ────────────────────────────────                                           │
│   The Gold layer is where we build STAR or SNOWFLAKE schemas:               │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │                      ┌─────────────────┐                            │   │
│   │                      │   FACT TABLE    │                            │   │
│   │                      │  patient_visits │                            │   │
│   │                      │─────────────────│                            │   │
│   │                      │ visit_id (PK)   │                            │   │
│   │   ┌──────────────┐   │ patient_key(FK) │   ┌──────────────┐         │   │
│   │   │ DIM_PATIENT  │◄──│ date_key (FK)   │──►│  DIM_DATE    │         │   │
│   │   │──────────────│   │ hospital_key(FK)│   │──────────────│         │   │
│   │   │ patient_key  │   │ diagnosis_key   │   │ date_key     │         │   │
│   │   │ name         │   │─────────────────│   │ full_date    │         │   │
│   │   │ birth_date   │   │ duration_mins   │   │ year, month  │         │   │
│   │   │ gender       │   │ cost            │   │ quarter      │         │   │
│   │   └──────────────┘   │ ...             │   └──────────────┘         │   │
│   │                      └────────┬────────┘                            │   │
│   │   ┌──────────────┐            │            ┌──────────────┐         │   │
│   │   │ DIM_HOSPITAL │◄───────────┴───────────►│DIM_DIAGNOSIS │         │   │
│   │   │──────────────│                         │──────────────│         │   │
│   │   │ hospital_key │                         │diagnosis_key │         │   │
│   │   │ name         │                         │ icd_code     │         │   │
│   │   │ location     │                         │ description  │         │   │
│   │   └──────────────┘                         └──────────────┘         │   │
│   │                                                                      │   │
│   │   Benefits:                                                          │   │
│   │   • Optimized for analytical queries (aggregations, joins)          │   │
│   │   • Business users understand the model                             │   │
│   │   • BI tools work naturally with star schemas                       │   │
│   │   • Consistent metrics across all reports                           │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   TRANSFORMATION TOOL OPTIONS:                                               │
│   • dbt-core (recommended) - SQL-based, tests built-in, open source         │
│   • Apache Spark - for heavy transformations                                │
│   • Dataform - if staying in GCP ecosystem                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OPEN TABLE FORMAT: APACHE ICEBERG                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   WHY ICEBERG (or Delta Lake):                                               │
│   • ACID transactions          • Time travel                                │
│   • Schema evolution           • Partition evolution                        │
│   • Hidden partitioning        • Engine-agnostic                            │
│   • Row-level operations       • Open specification                         │
│                                                                              │
│   The table format is the FOUNDATION of avoiding lock-in.                   │
│   Any engine that speaks Iceberg can read/write this data.                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              STORAGE LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Object Storage (cloud-agnostic):                                           │
│   • Google Cloud Storage (GCS)  │  Amazon S3  │  Azure ADLS  │  MinIO       │
│                                                                              │
│   Data stored as: Parquet files + Iceberg metadata                           │
│   (Open formats, readable by any tool, portable across clouds)              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA INGESTION LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CDC (from OLTP)              STREAMING              BATCH / FILES          │
│   ───────────────              ─────────              ─────────────          │
│   • Debezium (OSS)             • Apache Kafka         • Airbyte              │
│   • Striim (commercial)        • Pub/Sub              • Fivetran             │
│   • Datastream (GCP)           • Redpanda             • Custom scripts       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOURCE SYSTEMS (UNCHANGED - OLTP)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Operational databases REMAIN as-is. The lakehouse READS from them.        │
│                                                                              │
│   CURRENT: Sybase ASE    ───────►    FUTURE: AlloyDB / PostgreSQL           │
│                                                                              │
│   KWS continues to serve clinical workflows.                                │
│   We do NOT replace operational systems.                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Key Concept Deep-Dives

### 4.1 Governance: Why the Catalog is the Center

The original document scatters governance across BigQuery IAM, LookML, Cloud Audit Logs. This creates:
- Multiple places to configure access
- Inconsistent policy enforcement
- Difficult auditing

**In a proper lakehouse, the CATALOG is the single governance layer:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CATALOG-CENTRIC GOVERNANCE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   WRONG (Document Approach):          RIGHT (Catalog-Centric):              │
│   ──────────────────────────          ────────────────────────              │
│                                                                              │
│   ┌─────────┐  ┌─────────┐            ┌─────────────────────────────┐       │
│   │BigQuery │  │ Looker  │            │         CATALOG             │       │
│   │  IAM    │  │  ACLs   │            │  (Single source of truth)   │       │
│   └────┬────┘  └────┬────┘            └──────────────┬──────────────┘       │
│        │            │                                │                       │
│        ▼            ▼                                ▼                       │
│   Different    Different              All engines respect                    │
│   policies     policies               the same policies                      │
│   (conflict!)  (conflict!)                                                   │
│                                       Query Engine A ─┐                      │
│   ┌─────────┐                         Query Engine B ─┼─► Same access rules │
│   │ Audit   │  ← Where?               BI Tool C ──────┘                      │
│   │ Logs    │                                                                │
│   └─────────┘                         Audit: ONE place, complete history    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**What the catalog governs:**

| Function | Description |
|----------|-------------|
| **Schema registry** | What tables exist, their columns, types, partitions |
| **Access policies** | Who can see what tables, rows, columns |
| **Data quality scores** | Freshness, completeness, validity (fed by dbt/GE) |
| **Lineage** | Where data came from, what transforms applied |
| **Audit trail** | Every query, every access, every denial |
| **Data contracts** | SLAs, ownership, refresh schedules |

### 4.2 Lakehouse Manager: Unified View for Consumers

The lakehouse manager's PRIMARY purpose is to present a **unified view** to all consumers—hiding the complexity of multiple engines, formats, and systems underneath.

**Without a lakehouse manager:**
```
User A (Power BI)     → connects to BigQuery     → sees BigQuery tables
User B (Python)       → connects to Spark        → sees Spark tables
User C (Superset)     → connects to Trino        → sees Trino tables

Problem: Different views, different permissions, different experiences
```

**With a lakehouse manager:**
```
User A (Power BI)  ─┐
User B (Python)    ─┼─► Lakehouse Manager ─► Unified view
User C (Superset)  ─┘   (routes to best engine)

Benefit: Same data, same permissions, same experience
```

**Key capabilities:**

| Capability | Why It Matters |
|------------|----------------|
| **Unified SQL interface** | Users don't care which engine runs the query |
| **Semantic layer** | Business terms defined once, used everywhere |
| **Query federation** | Join data across sources transparently |
| **Caching/acceleration** | Frequently-used queries are fast |
| **Credential vending** | Secure access for external consumers |
| **Cost tracking** | Know who's using what resources |

### 4.3 Data Quality: Where It Happens

Data quality is **enforced** during transformation, **tracked** in the catalog, and **visible** to consumers:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DATA QUALITY FLOW                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. DEFINE               2. ENFORCE              3. TRACK & EXPOSE          │
│   ────────────            ─────────────           ──────────────────         │
│                                                                              │
│   dbt schema.yml          dbt run + test          Catalog dashboard          │
│   Great Expectations      Pipeline execution      Quality scores visible     │
│   config files                                    to all consumers           │
│                                                                              │
│   ┌───────────┐           ┌───────────┐          ┌───────────────────┐      │
│   │ Define:   │           │ Execute:  │          │ View in Catalog:  │      │
│   │           │           │           │          │                   │      │
│   │ • not_null│  ───────► │ Bronze    │ ───────► │ Table: patients   │      │
│   │ • unique  │           │   ▼       │          │ Quality: 98.5%    │      │
│   │ • accepted│           │ Silver    │          │ Last check: 1h ago│      │
│   │   values  │           │   ▼       │          │ Issues: 2 nulls   │      │
│   │ • custom  │           │ Gold      │          │                   │      │
│   └───────────┘           └───────────┘          └───────────────────┘      │
│                                                                              │
│   If checks fail:                                                            │
│   • Pipeline stops (don't promote bad data)                                 │
│   • Alert sent to data owner                                                │
│   • Failure logged in catalog                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 Avoiding Lock-in: The Composability Test

The original document would create Google lock-in. But we must also avoid creating NEW lock-in (e.g., to Dremio).

**The Composability Test:** Can we replace any component without rebuilding everything?

| Component | Can we swap it? | How? |
|-----------|-----------------|------|
| Storage (GCS) | ✅ Yes | Iceberg works on S3, ADLS, MinIO |
| Table format (Iceberg) | ✅ Yes | Could switch to Delta Lake |
| Catalog (Nessie) | ✅ Yes | Switch to Polaris, DataHub, etc. |
| Query engine (Trino) | ✅ Yes | Add/replace with Spark, Dremio, etc. |
| Transformation (dbt) | ✅ Yes | Could use Spark, Dataform |
| Lakehouse manager | ✅ Yes | Dremio, Starburst, or custom |
| BI tool | ✅ Yes | Any tool that speaks SQL |

**If any answer is "No," we've created lock-in.**

---

## 5. Component Options (Avoiding Lock-in)

### 5.1 Lakehouse Manager Options

These are OPTIONS, not recommendations. Evaluate based on needs:

| Product | Type | Strengths | Watch Out For |
|---------|------|-----------|---------------|
| **Dremio** | Commercial + OSS | Unified platform, Nessie catalog, reflections, semantic layer | Commercial features require license |
| **Starburst** | Commercial | Trino-based, strong federation, Galaxy SaaS | Less integrated catalog |
| **Trino (OSS)** | Open Source | No licensing cost, widely adopted | Requires assembly of other components |
| **Custom** | Build yourself | Maximum control | Significant engineering investment |

### 5.2 Catalog Options

| Product | Type | Strengths | Watch Out For |
|---------|------|-----------|---------------|
| **Project Nessie** | OSS | Git-like versioning, Iceberg-native | Smaller community |
| **Apache Polaris** | OSS (incubating) | Iceberg REST standard, Snowflake-backed | Very new |
| **DataHub** | OSS | Strong lineage, metadata platform | Not Iceberg-native catalog |
| **OpenMetadata** | OSS | Full governance platform | Different focus (metadata vs. table catalog) |
| **Unity Catalog OSS** | OSS | Databricks donation | Primarily Databricks ecosystem |

**Note on Unity Catalog:** While Databricks open-sourced parts of Unity Catalog, it remains primarily a **metadata catalog** designed for the Databricks ecosystem. It provides table/schema management and access control, but it's not a full "lakehouse manager" in the sense of providing query federation, semantic layer, or data sharing to external consumers. It's one piece of the puzzle, not the complete solution.

### 5.3 BI Tool Options

Users should choose what works for them. No lock-in at this layer:

| Category | Options |
|----------|---------|
| **Commercial** | Power BI, Tableau, Qlik Sense, Looker, ThoughtSpot, Sigma |
| **Open Source** | Apache Superset, Metabase, Lightdash, Redash |
| **Embedded** | Preset (hosted Superset), Hex, Observable |

All of these can connect to the lakehouse via standard SQL/JDBC/ODBC.

### 5.4 Query Engine Options

| Use Case | Options |
|----------|---------|
| **Interactive SQL** | Trino, Dremio, Starburst, BigQuery*, Snowflake* |
| **Batch / Heavy ETL** | Apache Spark, Databricks, Dataproc, EMR |
| **Streaming** | Apache Flink, Spark Structured Streaming |
| **Lightweight / Local** | DuckDB, Polars |

*BigQuery and Snowflake can query Iceberg tables, making them options rather than requirements.

---

## 6. Revised Roadmap

### Phase 0: Foundation & Evaluation (Weeks 1-6)

| Task | Deliverable |
|------|-------------|
| Define architecture principles | Document: what composability means for us |
| Evaluate lakehouse managers | POC with 2-3 options (e.g., Dremio, Starburst, Trino+Nessie) |
| Evaluate catalogs | Decide on catalog strategy |
| Define Iceberg standards | Naming, partitioning, schema conventions |
| Design credential vending model | Hospital access patterns, quotas |

### Phase 1: Infrastructure Setup (Weeks 7-12)

| Task | Deliverable |
|------|-------------|
| Deploy object storage | GCS buckets with proper structure |
| Deploy chosen catalog | Nessie, Polaris, or alternative |
| Deploy lakehouse manager | Based on evaluation results |
| Implement CDC pipeline | Debezium → Kafka → Iceberg for pilot tables |
| Configure governance | Access policies in catalog |

### Phase 2: Data Pipeline & Quality (Weeks 13-20)

| Task | Deliverable |
|------|-------------|
| Build Bronze layer | Raw CDC data in Iceberg |
| Build Silver layer | dbt models with quality checks |
| Build Gold layer | **Star/snowflake schema** for analytics |
| Implement data quality | Great Expectations, results to catalog |
| Test credential vending | Hospital isolation verified |

### Phase 3: Consumption & Validation (Weeks 21-28)

| Task | Deliverable |
|------|-------------|
| Deploy BI tools | Superset + one commercial option for comparison |
| Build pilot dashboards | Anomaly detection use case |
| Enable hospital access | Credential vending in production |
| User acceptance testing | Stakeholders validate |
| Cost model validation | Per-hospital tracking works |

### Phase 4: Production & Scale (Weeks 29+)

| Task | Deliverable |
|------|-------------|
| Production hardening | Security, monitoring, DR |
| Documentation & training | Complete |
| Expand data scope | Additional source tables |
| Advanced capabilities | Streaming, ML feature store |

---

## 7. Summary: Document Issues & Proposed Solutions

| Issue in Original Document | Proposed Solution |
|---------------------------|-------------------|
| Vendor lock-in (BigQuery + Looker) | Open table format (Iceberg) + composable components |
| Governance scattered across tools | Catalog-centric governance |
| No data sharing architecture | Credential vending as first-class feature |
| LookML is proprietary | Open semantic layer (in lakehouse manager or dbt) |
| Data quality unclear | DQ at transformation layer, tracked in catalog |
| No dimensional modeling guidance | Gold layer = star/snowflake schema |
| BigQuery-only compute | Multiple query engines on same data |
| Looker-only BI | Any BI tool (commercial or open source) |

---

## 8. Recommendations

### 8.1 Immediate Actions

1. **Do not commit** to long-term Looker licensing
2. **Mandate Iceberg** as storage format (even if using BigQuery via BigLake)
3. **Evaluate lakehouse managers** - POC with multiple options
4. **Design catalog-centric governance** before implementation
5. **Define credential vending model** for hospital access

### 8.2 Evaluation Criteria for Lakehouse Manager

| Criterion | Questions to Answer |
|-----------|---------------------|
| **Composability** | Can we swap it out later? |
| **Catalog integration** | Does it work with our chosen catalog? |
| **Query performance** | Fast enough for interactive analytics? |
| **Credential vending** | Does it support secure external sharing? |
| **Semantic layer** | Is it open/portable or proprietary? |
| **Cost model** | Predictable pricing we can pass to hospitals? |
| **Operational complexity** | Can our team manage it? |

### 8.3 Questions for the Team

1. Which lakehouse managers should we evaluate?
2. What's our catalog strategy (Nessie, Polaris, other)?
3. What commercial BI tools do hospitals already use?
4. What's our timeline for the evaluation phase?
5. Do we have skills for Spark/Trino, or do we need training?

---

## 9. Conclusion

The Data 2.0 initiative is necessary. The problems are real. But the proposed solution trades Sybase lock-in for Google lock-in.

An **Open, Composable Lakehouse Architecture** delivers:

| Benefit | How |
|---------|-----|
| Same functionality | Catalog + lakehouse manager + query engines |
| Same security | Credential vending + row-level security + audit |
| Same semantic layer | Open, SQL-based, portable |
| **PLUS: No lock-in** | Every component swappable |
| **PLUS: Future-proof** | Ready for whatever comes next |
| **PLUS: Hospital choice** | They pick their BI tools |

**The key insight:** The value is in the DATA (in open format) and the GOVERNANCE (in the catalog), not in any particular vendor's tools.

Build on open standards. Keep options open. Avoid new lock-in while escaping the old.

---

## Appendix: Glossary of Lakehouse Terms

| Term | Definition |
|------|------------|
| **Lakehouse** | Architecture combining data lake storage with warehouse features (ACID, schema enforcement) |
| **Open Table Format** | Specification (Iceberg, Delta) that enables multiple engines to read/write same tables |
| **Catalog** | System that tracks table metadata, schemas, and governance policies |
| **Lakehouse Manager** | Platform that provides unified query interface, semantic layer, and data sharing |
| **Credential Vending** | Mechanism to issue scoped, temporary credentials to external consumers |
| **Star Schema** | Dimensional model with central fact table surrounded by dimension tables |
| **Medallion Architecture** | Data organization pattern: Bronze (raw) → Silver (cleansed) → Gold (analytics) |
| **Composability** | Ability to swap components independently without rebuilding the stack |

---

*This document advocates for architectural principles over specific vendor choices. The goal is a data platform that serves Nexuzhealth for the next decade, regardless of how the vendor landscape evolves.*
