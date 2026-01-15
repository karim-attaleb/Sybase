# Response to Data 2.0 Management Document
## A Case for an Open, Composable Lakehouse Architecture

**Author:** [DBA/Data Engineer - New Recruit]  
**Date:** January 2026  
**Subject:** Technical Review & Alternative Architecture Proposal

---

## 1. Executive Summary

The Data 2.0 document presents a compelling case for moving away from the legacy Sybase/Select*Plus environment toward a modern analytics platform. The identified problems—query chaos, lack of SSOT, pressure on the data team, and inability to scale—are valid and urgent.

However, **the proposed solution creates significant vendor lock-in with Google Cloud (BigQuery + Looker)** while claiming to support "open standards." This response proposes an alternative: a **true Open Lakehouse Architecture** using composable, open-source components that deliver the same benefits while ensuring long-term flexibility, portability, and cost control.

---

## 2. Issues Identified in the Data 2.0 Document

### 2.1 Vendor Lock-in Concerns

| Claim in Document | Reality |
|-------------------|---------|
| "BigQuery supports open standards like Iceberg" | BigQuery's native format is proprietary. Iceberg support via BigLake is read-only for external tables, not native storage. |
| "LookML provides a semantic layer" | LookML is 100% proprietary to Looker. No portability to other BI tools. |
| "BigQuery Omni enables multi-cloud" | Omni requires BigQuery licensing on AWS/Azure—you're still locked to Google. |
| "Pay-as-you-go is transparent" | BigQuery's slot-based pricing and per-TB scanned model can be unpredictable. Hospitals cannot easily forecast costs. |

**Critical Issue:** If Nexuzhealth decides to change cloud providers or BI tools in 5 years, the migration cost would be enormous:
- All LookML models would need complete rewrite
- All data would need export from BigQuery's proprietary format
- All pipelines would need reconfiguration

### 2.2 Missing Components for a True Data Platform

The document focuses heavily on BI/reporting but lacks detail on:

| Gap | Impact |
|-----|--------|
| **No open table format strategy** | Data is locked in BigQuery's proprietary columnar format |
| **No data catalog / discovery** | How do users find and understand available datasets? |
| **No explicit data quality framework** | dbt tests mentioned but no comprehensive DQ strategy |
| **No schema evolution strategy** | How to handle source schema changes without breaking pipelines? |
| **No explicit CDC architecture** | Vague mention of "CDC tools" without specifics |
| **No disaster recovery plan** | What if BigQuery region fails? No multi-region strategy |
| **No cost allocation model** | How exactly will hospitals be charged? Per query? Per TB? |

### 2.3 Architectural Weaknesses

1. **Tight Coupling:** BigQuery as both storage AND compute creates dependency
2. **No True Lakehouse:** The proposal is a cloud data warehouse, not a lakehouse
3. **Single Semantic Layer:** LookML cannot serve data science workloads (Python, Spark, etc.)
4. **Limited Streaming:** BigQuery streaming is expensive; no mention of Kafka/Pub-Sub architecture
5. **No Data Mesh Consideration:** Document mentions "data products" but architecture is centralized monolith

### 2.4 Comparison Table: Document Claims vs. Industry Best Practice

| Aspect | Document Proposes | Industry Best Practice |
|--------|-------------------|----------------------|
| Storage Format | BigQuery native | Apache Iceberg / Delta Lake |
| Compute Engine | BigQuery only | Decoupled (Spark, Trino, DuckDB, etc.) |
| Semantic Layer | LookML (proprietary) | dbt Semantic Layer / Cube.dev (open) |
| Orchestration | Cloud Composer | Dagster / Airflow / Prefect |
| Data Catalog | Not specified | Unity Catalog / DataHub / OpenMetadata |
| BI Tool | Looker only | Any tool (Superset, Metabase, PowerBI, Tableau) |
| Streaming | BigQuery Streaming | Apache Kafka + Flink/Spark Streaming |

---

## 3. Proposed Alternative: Open Lakehouse Architecture

### 3.1 Core Principles

1. **Open Table Formats:** All data stored in Apache Iceberg (or Delta Lake)
2. **Decoupled Storage & Compute:** Any engine can query the data
3. **Composable Stack:** Each component can be replaced independently
4. **Cloud-Agnostic:** Works on GCP, AWS, Azure, or on-premises
5. **Open Semantic Layer:** Business logic portable across tools

### 3.2 Proposed Technology Stack

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CONSUMPTION LAYER                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  BI Tools        │  Data Science      │  Applications      │  APIs          │
│  ───────────     │  ────────────      │  ────────────      │  ────          │
│  • Superset      │  • Jupyter         │  • Custom Apps     │  • REST        │
│  • Metabase      │  • Databricks      │  • Hospital        │  • GraphQL     │
│  • PowerBI       │  • Vertex AI       │    Portals         │                │
│  • Tableau       │  • Python/R        │                    │                │
│  • Looker*       │                    │                    │                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SEMANTIC LAYER (Open)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│  • dbt Semantic Layer (metrics, dimensions, entities)                        │
│  • Cube.dev for API-first semantic access                                    │
│  • Centralized business logic, portable across all consumers                │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           QUERY ENGINES (Decoupled)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│  Interactive       │  Batch Processing   │  ML/AI Workloads                  │
│  ───────────       │  ────────────────   │  ────────────────                 │
│  • Trino/Starburst │  • Apache Spark     │  • Spark MLlib                    │
│  • DuckDB          │  • dbt-core         │  • Vertex AI                      │
│  • BigQuery*       │  • Dataproc         │  • Custom Python                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      OPEN TABLE FORMAT (Apache Iceberg)                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  Features:                                                                   │
│  • ACID transactions          • Schema evolution                            │
│  • Time travel                • Partition evolution                         │
│  • Hidden partitioning        • Engine-agnostic                             │
│  • Row-level deletes          • Vendor-neutral                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STORAGE LAYER                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  • Google Cloud Storage (GCS) - primary                                      │
│  • S3-compatible storage (for multi-cloud)                                   │
│  • Parquet files organized by Iceberg metadata                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DATA INGESTION & STREAMING                             │
├─────────────────────────────────────────────────────────────────────────────┤
│  Batch CDC           │  Real-time Streaming    │  File Ingestion            │
│  ─────────           │  ──────────────────     │  ──────────────            │
│  • Debezium          │  • Apache Kafka         │  • Airbyte                 │
│  • Datastream        │  • Pub/Sub              │  • Fivetran                │
│  • Striim            │  • Flink                │  • Custom ETL              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          SOURCE SYSTEMS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│  • Sybase ASE (current)  →  AlloyDB / PostgreSQL (future)                   │
│  • KWS operational system                                                    │
│  • External APIs, IoT, third-party data                                     │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    CROSS-CUTTING CONCERNS                                    │
├───────────────────┬───────────────────┬─────────────────────────────────────┤
│  ORCHESTRATION    │  GOVERNANCE       │  OBSERVABILITY                      │
│  ─────────────    │  ──────────       │  ─────────────                      │
│  • Dagster        │  • Unity Catalog  │  • Monte Carlo / Soda               │
│  • Airflow        │  • OpenMetadata   │  • Great Expectations               │
│  • Prefect        │  • DataHub        │  • Prometheus + Grafana             │
│                   │  • Apache Atlas   │  • OpenTelemetry                    │
├───────────────────┴───────────────────┴─────────────────────────────────────┤
│  SECURITY: IAM (GCP/Azure AD) │ Row-Level Security │ Column Masking │ CMEK  │
└─────────────────────────────────────────────────────────────────────────────┘

* BigQuery and Looker can still be used as ONE of many options, not the only option
```

### 3.3 Why Apache Iceberg?

Apache Iceberg is the emerging industry standard for open table formats:

| Feature | Benefit for Nexuzhealth |
|---------|------------------------|
| **Engine-agnostic** | Query with Spark, Trino, Flink, BigQuery, Snowflake, Databricks—any engine |
| **ACID transactions** | Safe concurrent writes from multiple pipelines |
| **Time travel** | Query data as it was at any point in time (audit, debugging) |
| **Schema evolution** | Add/rename/drop columns without rewriting data |
| **Partition evolution** | Change partitioning strategy without migration |
| **Hidden partitioning** | Users don't need to know partition columns |
| **No vendor lock-in** | Data is yours, stored as Parquet files |

**BigQuery now supports Iceberg tables via BigLake**, so you can have the best of both worlds: open storage + BigQuery compute when needed.

### 3.4 Open Semantic Layer: dbt + Cube.dev

Instead of proprietary LookML:

| Component | Purpose |
|-----------|---------|
| **dbt-core** | Transformations (Bronze → Silver → Gold) with built-in testing |
| **dbt Semantic Layer** | Define metrics, dimensions, entities ONCE |
| **Cube.dev** | Expose semantic layer via API to ANY consumer |

**Benefits:**
- Metrics defined once, used everywhere (BI, Python, APIs)
- Open source, no vendor lock-in
- Git-based version control (same as LookML claim)
- Works with ANY database or warehouse

---

## 4. Proposed Roadmap

### Phase 0: Foundation (Month 1-2)
**Parallel to pilot—establish standards**

| Task | Deliverable |
|------|-------------|
| Define Iceberg table standards | Naming conventions, partitioning strategy |
| Set up data catalog | OpenMetadata or DataHub instance |
| Establish dbt project structure | Monorepo with Bronze/Silver/Gold layers |
| Define data contracts | Schema versioning policy, SLAs |
| Security framework | RBAC model, row-level security design |

### Phase 1: Pilot with Open Architecture (Month 2-5)
**Same use case (anomaly detection), different architecture**

| Task | Details |
|------|---------|
| Set up GCS buckets | Landing zone, Bronze, Silver, Gold |
| Deploy Iceberg catalog | Use BigLake Metastore or AWS Glue-compatible |
| Implement CDC pipeline | Debezium → Kafka → Iceberg (or Datastream → GCS → Iceberg) |
| Build dbt models | Bronze → Silver → Gold transformations |
| Deploy query engine | Trino cluster OR BigQuery with external Iceberg tables |
| Build dashboards | Apache Superset (open) + optional Looker |
| Data quality | Great Expectations integrated in dbt |

### Phase 2: Production & Governance (Month 5-8)

| Task | Details |
|------|---------|
| Data catalog population | All datasets documented with lineage |
| Row-level security | Hospital isolation implemented at Iceberg level |
| Cost monitoring | Per-hospital usage tracking |
| Self-service enablement | Hospital analysts trained on Superset/approved tools |
| Disaster recovery | Multi-region replication for Iceberg tables |

### Phase 3: Scale & Advanced Use Cases (Month 8-12)

| Task | Details |
|------|---------|
| Real-time streaming | Kafka + Flink for near-real-time dashboards |
| ML/AI integration | Feature store on Iceberg, Vertex AI integration |
| Data sharing | Iceberg REST catalog for external partners |
| Data mesh patterns | Domain-oriented data products |

### Phase 4: Continuous Evolution (Ongoing)

| Task | Details |
|------|---------|
| Evaluate new engines | DuckDB for edge, Snowflake for specific workloads |
| Optimize costs | Right-size compute, implement caching |
| Advanced governance | Data contracts, automated quality checks |

---

## 5. Cost Comparison

### 5.1 BigQuery-Only Approach (Document Proposal)

| Cost Element | Risk |
|--------------|------|
| BigQuery storage | Proprietary format, $0.02/GB/month |
| BigQuery compute | $6.25/TB scanned OR $2,000+/month per 100 slots |
| Looker licenses | ~$3,000-5,000/user/year |
| Migration cost (future) | Extremely high—complete rewrite |

### 5.2 Open Lakehouse Approach

| Cost Element | Advantage |
|--------------|-----------|
| GCS storage (Parquet) | $0.02/GB/month (same) but portable |
| Compute (Trino/Spark) | Pay for actual compute, scale to zero possible |
| BI (Superset) | Open source, no per-user licensing |
| Migration cost | Near-zero—data is in open format |

### 5.3 Hidden Cost: Opportunity Cost of Lock-in

If in 3 years a better/cheaper solution emerges (e.g., Snowflake Iceberg, Databricks, new open-source tool), the BigQuery+Looker approach requires:
- Complete data migration
- Complete semantic layer rewrite
- Retraining all users

The Open Lakehouse approach requires:
- Add new query engine (data stays in place)
- Connect new BI tool to existing semantic layer
- Minimal retraining

---

## 6. Addressing Document's Concerns

| Document Concern | Open Lakehouse Answer |
|------------------|----------------------|
| "Need serverless scalability" | Trino on Kubernetes auto-scales; BigQuery can still be used via BigLake |
| "Need enterprise security" | Iceberg + OPA policies + IAM = same security model |
| "Need semantic layer for SSOT" | dbt Semantic Layer + Cube.dev = portable SSOT |
| "Need real-time analytics" | Kafka + Flink + Iceberg = true streaming lakehouse |
| "Google has better AI/ML" | Vertex AI works with ANY data in GCS, not just BigQuery |
| "Team knows SQL" | Trino/Spark/dbt all use SQL; no new language needed |

---

## 7. Recommendations

### 7.1 Immediate Actions

1. **Do not commit** to Looker multi-year licensing until pilot proves value
2. **Require Iceberg storage** as a pilot success criterion
3. **Evaluate open BI tools** (Superset, Metabase) alongside Looker
4. **Implement dbt** regardless of final architecture—it's already mentioned in document

### 7.2 Pilot Modifications

| Original Pilot Plan | Suggested Modification |
|--------------------|----------------------|
| Store in BigQuery native | Store in Iceberg tables (query via BigLake) |
| LookML semantic layer | dbt Semantic Layer + optional Looker consumption |
| Looker dashboards only | Superset + Looker comparison |
| Cloud Composer | Dagster (better developer experience, open source) |

### 7.3 Success Criteria Additions

Add these to the pilot's critical success factors:

- [ ] Data queryable by multiple engines (not just BigQuery)
- [ ] Semantic layer portable (not Looker-dependent)
- [ ] Cost per hospital accurately measurable and predictable
- [ ] Data exportable without Google tooling
- [ ] Schema changes handled without pipeline rewrites

---

## 8. Conclusion

The Data 2.0 document correctly identifies the problems and the need for a modern data platform. However, the proposed solution **trades one form of lock-in (Sybase) for another (Google Cloud)**.

A true **Open Lakehouse Architecture** delivers:

| Benefit | How |
|---------|-----|
| **Same scalability** | Iceberg + Trino/BigQuery hybrid |
| **Same security** | Same IAM, RLS, encryption—standard practices |
| **Same semantic layer** | dbt + Cube instead of LookML |
| **PLUS portability** | Change any component without migration |
| **PLUS cost control** | No per-user BI licensing, flexible compute |
| **PLUS future-proofing** | Ready for whatever comes next |

**My recommendation:** Proceed with the pilot, but mandate open table formats (Iceberg) and evaluate open semantic layers alongside the proposed Google stack. This protects Nexuzhealth's investment regardless of which direction the industry moves.

---

## Appendix A: Open Source Alternatives Reference

| Category | Proprietary (Document) | Open Alternative |
|----------|----------------------|------------------|
| Storage Format | BigQuery native | Apache Iceberg, Delta Lake |
| Query Engine | BigQuery | Trino, Apache Spark, DuckDB |
| Semantic Layer | LookML | dbt Semantic Layer, Cube.dev |
| BI / Dashboards | Looker | Apache Superset, Metabase, Lightdash |
| Orchestration | Cloud Composer | Dagster, Apache Airflow, Prefect |
| Data Catalog | (not specified) | OpenMetadata, DataHub, Apache Atlas |
| Data Quality | dbt tests | Great Expectations, Soda, Monte Carlo |
| Streaming | BigQuery Streaming | Apache Kafka, Apache Flink |
| CDC | Datastream | Debezium, Striim |

## Appendix B: Key Industry References

- [Apache Iceberg Documentation](https://iceberg.apache.org/)
- [dbt Semantic Layer](https://docs.getdbt.com/docs/build/semantic-layer)
- [Cube.dev](https://cube.dev/)
- [Tabular (Iceberg creators)](https://tabular.io/)
- [BigLake Iceberg Tables](https://cloud.google.com/bigquery/docs/biglake-iceberg)
- [Open Data Lakehouse Principles](https://www.databricks.com/glossary/data-lakehouse)

---

*This document is intended to promote constructive dialogue about architecture decisions. The goal is to ensure Nexuzhealth builds a data platform that serves its needs for the next decade, not just the next project.*
