# Response to Data 2.0 Management Document (Revised)
## A Case for an Open, Composable Lakehouse Architecture

**Author:** [DBA/Data Engineer]  
**Date:** January 2026  
**Subject:** Technical Review & Alternative Architecture Proposal  
**Version:** 2.0

---

## 1. Executive Summary

The Data 2.0 document correctly identifies the need for a dedicated **OLAP/analytics platform** to complement the existing operational systems (OLTP). The current situation—delivering analytics as a byproduct of KWS via Select*Plus and stored queries—is unsustainable.

**Key clarification:** We are NOT replacing operational databases. We are adding a unified analytics platform that:
- Ingests data FROM operational systems (Sybase today, AlloyDB/PostgreSQL tomorrow)
- Provides a dedicated environment for analytics, BI, and data science
- Enables secure data sharing with hospitals and third parties

However, **the proposed BigQuery + Looker solution creates significant vendor lock-in** while the document claims to support "open standards." This response proposes a **true Open Lakehouse Architecture** with:

- **Open table format** (Apache Iceberg) for storage
- **Lakehouse manager** for catalog, governance, and query federation
- **Credential vending** for secure, scalable data sharing
- **Composable components** that can be swapped without migration

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

**The fundamental question:** If we're using Iceberg anyway, why pay BigQuery premium pricing when other engines (Dremio, Trino, Spark) can query the same data more cost-effectively?

### 2.3 Critical Missing Component: Data Sharing Architecture

The document mentions "data sharing" but lacks a concrete architecture for **credential vending**—the mechanism that allows:

- Hospitals to query ONLY their own data
- Fine-grained access control without copying data
- Audit trails of who accessed what
- Scalable onboarding of new consumers

This is **fundamental** for a multi-tenant healthcare data platform.

### 2.4 Missing: Lakehouse Manager

A modern lakehouse requires a **unified management layer** that handles:

| Function | What It Does | Document Gap |
|----------|--------------|--------------|
| **Catalog management** | Track all tables, schemas, partitions | Not addressed |
| **Data versioning** | Time travel, branching, rollback | Vaguely mentioned |
| **Access control** | Row/column security, credential vending | Scattered across BigQuery + Looker |
| **Query federation** | Query across sources with single interface | Not addressed |
| **Semantic layer** | Business definitions, metrics | Locked in LookML |
| **Data sharing** | Secure external access | Not architected |

### 2.5 Other Gaps

| Gap | Impact |
|-----|--------|
| **No schema evolution strategy** | Source changes break pipelines |
| **No data quality framework** | dbt tests mentioned, no comprehensive approach |
| **No disaster recovery** | What if GCP region fails? |
| **No cost allocation model** | How exactly are hospitals charged? |

---

## 3. Proposed Alternative: Open Lakehouse Architecture

### 3.1 Core Principles

1. **OLTP remains untouched** - Operational DBs (Sybase → AlloyDB) continue serving KWS
2. **Open table format** - All analytics data stored in Apache Iceberg
3. **Lakehouse manager** - Unified catalog, governance, query, and sharing
4. **Credential vending** - Secure, scalable data sharing as first-class feature
5. **Composable stack** - Each component replaceable independently
6. **Cloud-flexible** - Works on GCP, AWS, Azure, or hybrid

### 3.2 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CONSUMPTION LAYER                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   HOSPITALS / EXTERNAL              INTERNAL NEXUZHEALTH                     │
│   ────────────────────              ────────────────────                     │
│   • BI Tools (their choice)         • BI Dashboards                          │
│   • Data Science notebooks          • Operational reports                    │
│   • Custom applications             • Data Science / ML                      │
│   • API access                      • Ad-hoc analysis                        │
│                                                                              │
│   ▼ CREDENTIAL VENDING ▼            ▼ DIRECT ACCESS ▼                        │
│   (secure, audited, scoped)         (internal IAM)                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│                        LAKEHOUSE MANAGER                                     │
│                        (Dremio / Starburst / Tabular / Unity Catalog)        │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   UNIFIED   │  │  SEMANTIC   │  │   ACCESS    │  │    DATA     │         │
│  │   CATALOG   │  │    LAYER    │  │   CONTROL   │  │   SHARING   │         │
│  ├─────────────┤  ├─────────────┤  ├─────────────┤  ├─────────────┤         │
│  │ • Schema    │  │ • Metrics   │  │ • RBAC/ABAC │  │ • Credential│         │
│  │   registry  │  │ • Business  │  │ • Row-level │  │   vending   │         │
│  │ • Versioning│  │   terms     │  │   security  │  │ • Shares    │         │
│  │ • Lineage   │  │ • Joins     │  │ • Column    │  │ • Audit     │         │
│  │ • Discovery │  │ • Transforms│  │   masking   │  │ • Quotas    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUERY / COMPUTE ENGINES                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   The lakehouse manager can federate queries to multiple engines:            │
│                                                                              │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐                  │
│   │    DREMIO    │    │    SPARK     │    │   OPTIONAL   │                  │
│   │   (built-in) │    │  (batch/ML)  │    │   ENGINES    │                  │
│   ├──────────────┤    ├──────────────┤    ├──────────────┤                  │
│   │ • SQL queries│    │ • Heavy ETL  │    │ • BigQuery   │                  │
│   │ • Interactive│    │ • ML training│    │ • Trino      │                  │
│   │ • Reflections│    │ • Complex    │    │ • DuckDB     │                  │
│   │   (caching)  │    │   transforms │    │ • Snowflake  │                  │
│   └──────────────┘    └──────────────┘    └──────────────┘                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OPEN TABLE FORMAT: APACHE ICEBERG                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         ICEBERG TABLES                               │   │
│   ├─────────────────────────────────────────────────────────────────────┤   │
│   │  • ACID transactions        • Time travel (query any version)       │   │
│   │  • Schema evolution         • Partition evolution                   │   │
│   │  • Hidden partitioning      • Row-level deletes/updates             │   │
│   │  • Engine-agnostic          • Branch/tag for dev/test               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Data Organization (Medallion Architecture):                                │
│                                                                              │
│   ┌───────────────┐    ┌───────────────┐    ┌───────────────┐              │
│   │    BRONZE     │ -> │    SILVER     │ -> │     GOLD      │              │
│   │   (raw CDC)   │    │  (cleansed)   │    │  (analytics)  │              │
│   └───────────────┘    └───────────────┘    └───────────────┘              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              STORAGE LAYER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Object Storage (cloud-agnostic):                                           │
│   • Google Cloud Storage (GCS)  - if staying on GCP                          │
│   • Amazon S3                   - if multi-cloud                             │
│   • Azure ADLS                  - if multi-cloud                             │
│   • MinIO                       - if on-premises required                    │
│                                                                              │
│   Data stored as: Parquet files + Iceberg metadata                           │
│   (Open formats, readable by any tool)                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DATA INGESTION LAYER                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CHANGE DATA CAPTURE (CDC)         STREAMING              BATCH/FILES       │
│   ─────────────────────────         ─────────              ───────────       │
│   From operational DBs:             Real-time events:      Bulk loads:       │
│                                                                              │
│   ┌─────────────┐                   ┌─────────────┐       ┌─────────────┐   │
│   │  Debezium   │                   │   Kafka /   │       │  Airbyte /  │   │
│   │  (preferred)│                   │   Pub/Sub   │       │  Fivetran / │   │
│   ├─────────────┤                   ├─────────────┤       │   Custom    │   │
│   │ • Sybase    │                   │ • IoT data  │       ├─────────────┤   │
│   │ • PostgreSQL│                   │ • App events│       │ • CSV/JSON  │   │
│   │ • AlloyDB   │                   │ • Logs      │       │ • API pulls │   │
│   └─────────────┘                   └─────────────┘       └─────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SOURCE SYSTEMS (UNCHANGED - OLTP)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   These operational databases REMAIN as-is:                                  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  CURRENT                        FUTURE                               │   │
│   │  ───────                        ──────                               │   │
│   │  Sybase ASE  ──────────────>   AlloyDB / PostgreSQL / MS SQL        │   │
│   │                                                                      │   │
│   │  KWS (clinical workflows)       KWS (clinical workflows)            │   │
│   │  Operational transactions       Operational transactions            │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   The lakehouse READS from these systems via CDC.                            │
│   It does NOT replace them.                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                         CROSS-CUTTING CONCERNS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ORCHESTRATION              DATA QUALITY            OBSERVABILITY           │
│   ─────────────              ────────────            ─────────────           │
│   • Dagster (recommended)    • dbt tests             • Pipeline metrics      │
│   • Apache Airflow           • Great Expectations    • Query performance     │
│   • Prefect                  • Soda                  • Cost tracking         │
│                              • Custom validations    • Audit logs            │
│                                                                              │
│   SECURITY                   ML / AI                                         │
│   ────────                   ─────────                                       │
│   • Identity provider        • Platform-agnostic                             │
│     (Azure AD, Okta, etc.)   • Spark MLlib                                   │
│   • CMEK (customer keys)     • Any ML platform                               │
│   • Audit logging            • Feature store on Iceberg                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 The Lakehouse Manager: Why It's Essential

The document proposes BigQuery + Looker as separate components. A **lakehouse manager** unifies these functions:

| Function | BigQuery + Looker | Lakehouse Manager (e.g., Dremio) |
|----------|-------------------|----------------------------------|
| Catalog | BigQuery datasets | Unified Iceberg catalog (Nessie/Arctic) |
| Semantic layer | LookML (proprietary) | Built-in, SQL-based, open |
| Query engine | BigQuery only | Built-in + federation to Spark, etc. |
| Data sharing | Analytics Hub (GCP-only) | Credential vending (cloud-agnostic) |
| Caching | BI Engine (extra cost) | Reflections (built-in) |
| Governance | Scattered | Unified in one place |

#### Lakehouse Manager Options

| Product | Type | Strengths | Considerations |
|---------|------|-----------|----------------|
| **Dremio** | Commercial + OSS | Full lakehouse platform, Nessie catalog, reflections, credential vending, semantic layer | Commercial license for enterprise features |
| **Starburst** | Commercial | Trino-based, strong federation, Galaxy SaaS option | Less integrated catalog |
| **Tabular** | Commercial (SaaS) | Built by Iceberg creators, excellent catalog | Newer, less mature |
| **Databricks Unity Catalog** | Commercial | Strong if using Databricks ecosystem | Tied to Databricks |
| **Apache Polaris (incubating)** | Open Source | Snowflake-donated, REST catalog standard | Very new, not production-ready |

**Recommendation:** Evaluate Dremio as primary option—it fills multiple roles (query engine, catalog, semantic layer, data sharing) in one platform.

### 3.4 Credential Vending: The Data Sharing Foundation

This is **critical** for Nexuzhealth's multi-tenant model and was underspecified in the original document.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CREDENTIAL VENDING FLOW                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   HOSPITAL USER                                                              │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  1. AUTHENTICATE                                                     │   │
│   │     User logs in via hospital's identity provider (Azure AD, etc.)  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  2. REQUEST ACCESS                                                   │   │
│   │     "I want to query patient_visits for Hospital X"                 │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  3. LAKEHOUSE MANAGER EVALUATES                                      │   │
│   │     • Is user authorized for Hospital X data?                       │   │
│   │     • What rows/columns can they see?                               │   │
│   │     • What is their quota?                                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  4. VEND TEMPORARY CREDENTIALS                                       │   │
│   │     • Short-lived token (e.g., 1 hour)                              │   │
│   │     • Scoped to specific tables/rows                                │   │
│   │     • Enforces column masking                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  5. USER QUERIES DATA                                                │   │
│   │     • With their BI tool of choice                                  │   │
│   │     • Or Python notebook                                            │   │
│   │     • Or custom application                                         │   │
│   │     All queries logged for audit                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why this matters:**
- Hospitals can use ANY tool (not just Looker)
- Access control is centralized, not scattered
- Every query is auditable
- Credentials expire automatically
- No data copying required

### 3.5 Why NOT Cube.dev?

In the original response, I suggested Cube.dev as an open semantic layer. Upon reflection:

| If using Dremio | Cube.dev is redundant |
|-----------------|----------------------|
| Dremio has built-in semantic layer | Cube adds another component to manage |
| Dremio's semantic layer is SQL-based | Cube requires YAML/JS configuration |
| Dremio integrates with catalog | Cube is separate from catalog |

**Revised recommendation:** Use the semantic layer built into the lakehouse manager. Add Cube.dev only if you need API-first semantic access for embedded analytics AND your lakehouse manager doesn't provide it.

### 3.6 Why NOT Vertex AI (specifically)?

The original document and my first response both assumed GCP. But a truly open lakehouse should be **ML-platform agnostic**:

| Approach | Benefit |
|----------|---------|
| Store features in Iceberg | Any ML platform can read them |
| Use open formats (Parquet) | No conversion needed |
| Expose via SQL | Spark MLlib, scikit-learn, PyTorch all work |

**Recommendation:** 
- Build a **feature store on Iceberg** (using Feast or similar)
- Train models with whatever platform makes sense (Vertex, SageMaker, Databricks ML, open-source)
- Don't lock into any vendor's ML stack

---

## 4. Revised Technology Stack

### 4.1 Recommended Components

| Layer | Component | Primary Option | Alternatives |
|-------|-----------|----------------|--------------|
| **Storage** | Object storage | GCS (if GCP) | S3, ADLS, MinIO |
| **Table format** | Open format | Apache Iceberg | Delta Lake |
| **Lakehouse manager** | Catalog + query + sharing | **Dremio** | Starburst, Tabular |
| **Transformation** | ELT | dbt-core | Dataform, custom Spark |
| **Orchestration** | Pipeline scheduling | Dagster | Airflow, Prefect |
| **CDC** | Change capture | Debezium | Striim, Datastream |
| **Streaming** | Real-time | Apache Kafka | Pub/Sub, Redpanda |
| **Data quality** | Validation | Great Expectations | Soda, dbt tests |
| **BI** | Visualization | Apache Superset | Metabase, Lightdash, Looker* |
| **ML** | Platform-agnostic | Feature store on Iceberg | Feast, custom |

*Looker can still be used as ONE option for BI, not the only option

### 4.2 What This Replaces from Original Proposal

| Original Proposal | Open Lakehouse | Benefit |
|-------------------|----------------|---------|
| BigQuery (storage) | Iceberg on object storage | Portable, no lock-in |
| BigQuery (query) | Dremio / lakehouse manager | Multiple engines possible |
| LookML | Dremio semantic layer + dbt | Open, SQL-based |
| Looker | User's choice of BI tool | No per-seat licensing |
| Cloud Composer | Dagster | Better DX, open source |
| Analytics Hub | Credential vending | Cloud-agnostic sharing |

---

## 5. Proposed Roadmap

### Phase 0: Foundation & Standards (Weeks 1-4)

| Task | Deliverable |
|------|-------------|
| Select lakehouse manager | POC with Dremio (or alternative) |
| Define Iceberg standards | Naming, partitioning, schema conventions |
| Design credential vending model | Hospital access patterns, quotas |
| Set up dbt project structure | Bronze/Silver/Gold layers |
| Define data contracts | Schema versioning, SLAs |

### Phase 1: Infrastructure & Pilot Ingestion (Weeks 5-10)

| Task | Deliverable |
|------|-------------|
| Deploy object storage | GCS buckets (landing, bronze, silver, gold) |
| Deploy lakehouse manager | Dremio cluster with Nessie catalog |
| Implement CDC pipeline | Debezium → Kafka → Iceberg for pilot tables |
| Build Bronze layer | Raw CDC data in Iceberg tables |
| Configure access control | Hospital isolation, row-level security |

### Phase 2: Transformation & Semantic Layer (Weeks 11-16)

| Task | Deliverable |
|------|-------------|
| Build Silver layer | dbt models for cleansing, standardization |
| Build Gold layer | Analytics-ready datasets, aggregations |
| Define semantic layer | Business terms, metrics in lakehouse manager |
| Implement data quality | Great Expectations checks in pipeline |
| Test credential vending | Hospital users access their data only |

### Phase 3: BI & Validation (Weeks 17-22)

| Task | Deliverable |
|------|-------------|
| Deploy BI tool | Apache Superset (and optionally Looker for comparison) |
| Build pilot dashboards | Anomaly detection use case |
| User acceptance testing | Hospital stakeholders validate |
| Performance tuning | Reflections/caching, query optimization |
| Cost model validation | Per-hospital usage tracking works |

### Phase 4: Production & Scale (Weeks 23-30)

| Task | Deliverable |
|------|-------------|
| Production deployment | Full security, monitoring, DR |
| Onboard pilot hospitals | Credential vending in production |
| Documentation | Runbooks, user guides, data catalog |
| Training | Internal team + hospital power users |
| Expand data scope | Additional source tables beyond pilot |

### Phase 5: Advanced Capabilities (Ongoing)

| Task | Deliverable |
|------|-------------|
| Real-time streaming | Kafka → Flink → Iceberg for live dashboards |
| Feature store | ML features on Iceberg |
| Data mesh patterns | Domain-oriented data products |
| Multi-cloud readiness | Test portability to other clouds |

---

## 6. Addressing Original Document's Concerns

| Document Concern | Open Lakehouse Answer |
|------------------|----------------------|
| "Need serverless scalability" | Lakehouse manager auto-scales; Spark for heavy loads |
| "Need enterprise security" | Credential vending + row-level security + audit |
| "Need semantic layer for SSOT" | Lakehouse manager semantic layer (SQL-based, open) |
| "Need real-time analytics" | Kafka + Flink + Iceberg streaming |
| "Need data sharing with hospitals" | Credential vending is first-class feature |
| "Team knows SQL" | Entire stack is SQL-native |
| "Need AI/ML capabilities" | Feature store on Iceberg, any ML platform |

---

## 7. Cost Comparison

### 7.1 BigQuery + Looker Approach

| Item | Cost Model | Risk |
|------|------------|------|
| BigQuery storage | ~$0.02/GB/month | Proprietary format |
| BigQuery compute | $6.25/TB scanned OR $2,000+/100 slots | Unpredictable |
| Looker licenses | $3,000-5,000/user/year | Scales with users |
| Future migration | Extremely high | Complete rewrite |

### 7.2 Open Lakehouse Approach

| Item | Cost Model | Advantage |
|------|------------|-----------|
| Object storage | ~$0.02/GB/month | Portable, open format |
| Lakehouse manager | License OR compute-based | Predictable |
| BI tool | Open source (Superset) | No per-user fee |
| Future migration | Near-zero | Data stays, swap engines |

### 7.3 The Real Cost: Lock-in

If Nexuzhealth needs to change platforms in 5 years:

| Scenario | BigQuery + Looker | Open Lakehouse |
|----------|-------------------|----------------|
| Switch BI tool | Rewrite all LookML | Connect new tool to same data |
| Switch warehouse | Export all data + rewrite pipelines | Add new engine, data stays |
| Multi-cloud | Pay Google everywhere (Omni) | Same Iceberg, any cloud |

---

## 8. Recommendations

### 8.1 Immediate Actions

1. **Evaluate lakehouse managers** - POC with Dremio (and optionally Starburst)
2. **Mandate Iceberg storage** - Even if using BigQuery, store as Iceberg via BigLake
3. **Design credential vending** - This is foundational for hospital access
4. **Do not commit to Looker** - Evaluate open BI tools in parallel

### 8.2 Pilot Success Criteria (Additions)

Add these to the original document's success factors:

- [ ] Data stored in Apache Iceberg format
- [ ] Queryable by multiple engines (not just one vendor)
- [ ] Credential vending working for hospital access
- [ ] Cost per hospital accurately trackable
- [ ] Semantic layer not locked to proprietary format
- [ ] Data exportable without vendor tooling

### 8.3 Questions for the Team

1. **Lakehouse manager selection:** Should we evaluate Dremio, Starburst, and Tabular?
2. **BI tool strategy:** Is Superset acceptable, or is commercial BI required?
3. **CDC tool:** Debezium (open source) or commercial (Striim)?
4. **Cloud commitment:** Are we committed to GCP, or should we be cloud-agnostic?
5. **Timeline:** Can we extend pilot to properly evaluate open alternatives?

---

## 9. Conclusion

The Data 2.0 initiative is necessary and urgent. The problems identified—query chaos, lack of SSOT, inability to scale—are real.

However, **the proposed BigQuery + Looker architecture trades one lock-in for another**. 

An **Open Lakehouse Architecture** delivers:

| Same Benefit | How It's Achieved |
|--------------|-------------------|
| Scalability | Lakehouse manager + Spark for heavy loads |
| Security | Credential vending + row-level security |
| Semantic layer | Built into lakehouse manager (SQL-based) |
| Data sharing | Credential vending (first-class feature) |
| **PLUS: Portability** | Iceberg format, swap any component |
| **PLUS: Cost control** | No per-user BI licensing |
| **PLUS: Future-proofing** | Ready for whatever comes next |

**Recommendation:** Proceed with the pilot, but use open components:
- Iceberg for storage (not BigQuery native)
- Lakehouse manager for catalog/query/sharing (evaluate Dremio)
- dbt for transformations
- Open BI tools alongside any commercial evaluation

This protects Nexuzhealth's investment regardless of how the market evolves.

---

## Appendix A: Lakehouse Manager Comparison

| Capability | Dremio | Starburst | Tabular | Unity Catalog |
|------------|--------|-----------|---------|---------------|
| Query engine | Built-in (Dremio Sonar) | Trino | External | Spark/Photon |
| Catalog | Nessie/Arctic | Hive-compatible | Iceberg REST | Unity |
| Semantic layer | Yes | Limited | No | No |
| Credential vending | Yes (Arctic) | Yes (Galaxy) | Yes | Yes |
| Reflections/caching | Yes | Materialized views | No | Delta caching |
| Open source option | Yes (community) | Yes (Trino) | No | No |
| SaaS option | Dremio Cloud | Starburst Galaxy | Yes | Databricks |

## Appendix B: Open Source Alternatives Summary

| Function | Proprietary (Document) | Open Alternative |
|----------|----------------------|------------------|
| Storage format | BigQuery native | Apache Iceberg |
| Query engine | BigQuery | Dremio, Trino, Spark |
| Semantic layer | LookML | Lakehouse manager, dbt metrics |
| BI | Looker | Apache Superset, Metabase |
| Orchestration | Cloud Composer | Dagster, Airflow |
| CDC | Datastream | Debezium |
| Data catalog | (implicit in BigQuery) | Nessie, OpenMetadata |

---

*This document advocates for an open, composable architecture that delivers the same benefits as the proposed solution while ensuring long-term flexibility and avoiding vendor lock-in.*
