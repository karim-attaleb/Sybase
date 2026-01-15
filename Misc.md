# Data 2.0 - Summary for DBA/Data Engineers

## Executive Overview
Nexuzhealth is transitioning from a legacy Sybase-based OLTP data delivery model (Select*Plus, stored queries) to a modern cloud-based OLAP platform using **Google BigQuery** + **Looker**. This document outlines the strategic, technical, and operational aspects of this migration.

---

## Current State Problems
- **No real OLAP/DWH environment** - data delivered as byproduct of KWS (OLTP system)
- **Query chaos** - multiple versions of same queries, inconsistent logic, potential data leaks
- **Pressure on data team** - constant ad-hoc requests, manual/repetitive work
- **No Single Source of Truth (SSOT)** - fragmented data across systems
- **Limited scalability** - difficult to implement AI/ML, self-service analytics, or benchmarking

---

## Target Architecture

### Tech Stack
| Component | Technology | Purpose |
|-----------|------------|---------|
| Data Warehouse | **Google BigQuery** | Serverless, columnar storage, MPP |
| Semantic Layer | **Looker + LookML** | Centralized business logic, SSOT |
| ETL/ELT | **dbt (data build tool)** | Transformations, lineage, testing |
| Orchestration | **Cloud Composer** | Pipeline scheduling, alerting |
| Staging | **Google Cloud Storage (GCS)** | Raw file landing zone |
| Version Control | **Git** | LookML and dbt code management |

### Data Flow
```
Source Systems (Sybase ASE → future AlloyDB/MS SQL)
        ↓
    GCS (staging/landing zone)
        ↓
    BigQuery (Bronze → Silver → Gold layers)
        ↓
    Looker (semantic layer via LookML)
        ↓
    Dashboards / Self-Service / APIs
```

---

## Key Technical Considerations

### Data Modeling
- Shift from normalized OLTP to **denormalized** structures for analytics performance
- **Partitioning** and **clustering** are critical for cost optimization
- Separation of **compute and storage** - different optimization strategies than traditional DB servers
- Data quality gates: Bronze → Silver → Gold promotion only after validation

### Security & Governance
| Feature | Implementation |
|---------|----------------|
| Access Control | Google Cloud IAM, RBAC, ABAC |
| Row/Column Security | BigQuery RLS & CLS, Looker row-level security |
| Encryption | At-rest & in-transit (default), CMEK for key control |
| Audit | Cloud Audit Logs, Access Transparency |
| Data Masking | Dynamic Data Masking, tokenization, pseudonymization |
| Compliance | GDPR-ready, Data Boundary (EU), Assured Workloads |

### Cost Management (Critical!)
- **Pay-as-you-go** or **reserved slots** model
- Costs driven by: query volume, storage, streaming
- Must implement:
  - Budget alerts per project
  - Query cost limits per user/project
  - Usage dashboards
  - Lifecycle policies (cold storage vs archive)
- **Risk**: Full table scans, uncontrolled ad-hoc queries, uncleared staging data

### Integration Points
- CDC/replication tools for source systems
- VPN/Interconnect for on-prem connectivity
- REST APIs and standard SQL for external access
- Native integrations: Pub/Sub, Dataflow, Vertex AI

---

## Migration Approach

### Pilot Project: Anomaly Detection (Access Logging)
7-phase rollout:
1. **Initiation & Analysis** - scope, KPIs, architecture
2. **BigQuery Setup** - projects, datasets, IAM, ELT pipelines
3. **LookML Modeling** - semantic layer, naming conventions, Git
4. **Dashboards & Validation** - build, UAT, RBAC config
5. **Go-Live Prep** - docs, training, final checks
6. **Go-Live** - monitoring, issue resolution
7. **Optimization** - additional datasets, governance procedures

### Data Migration Tasks
- Reload historical data to cloud
- Bulk loads via GCS or parallel ingestion
- Validation & reconciliation against source
- Plan for future source change (Sybase → AlloyDB/MS SQL)

---

## Operational Changes

### New Daily Operations
- **Pipeline monitoring** - health checks, failures, retries, SLA tracking
- **Data quality** - automated row counts, checksums, freshness checks
- **Cost monitoring** - compute & storage tracking, anomaly detection
- **Lineage tracking** - dbt generates automatic documentation

### New Skills Required
| Skill Area | Topics |
|------------|--------|
| SQL | BigQuery SQL, distributed SQL patterns |
| Data Modeling | Columnar/distributed warehouses, star schema, data vault |
| Tooling | dbt, LookML, Cloud Composer |
| Security | RBAC/ABAC, tokenization, encryption, KMS |
| DevOps | CI/CD for data pipelines, Git workflows |

---

## Key Roles (BI Department)
- **BI Manager** - strategy, budget, stakeholder communication
- **Data Architect** - warehouse design, standards, tech decisions
- **Data Engineer / ETL-ELT Developer** - pipelines, integrations, data quality
- **BI Developer / Analytics Engineer** - dashboards, data models, self-service
- **Data Steward / Data Owner** - quality, business rules, metadata
- **Security & Compliance Officer** - IAM, encryption, auditing
- **DevOps / Platform Engineer** - CI/CD, monitoring, availability

---

## Why BigQuery over Microsoft Fabric?
| Criterion | BigQuery | Fabric |
|-----------|----------|--------|
| Maturity | Since 2010, proven at scale | 2023, still evolving |
| Scalability | Petabyte-scale, serverless auto-scaling | Less optimized for large analytics |
| Cost Model | Transparent pay-per-query or slots | F-capacities generate cost even when idle |
| Open Standards | Iceberg support, multi-cloud (Omni) | Primarily OneLake/MS ecosystem |
| AI/ML | BigQuery ML, Vertex AI integration | Less mature integration |
| Nexuzhealth fit | No existing MS ecosystem dependency | Would only benefit if using Power BI/Azure |

---

## Critical Success Factors
- Executive sponsorship and clear governance
- Defined use cases with measurable KPIs
- Stable, well-documented data models
- Automated data quality checks in pipelines
- Query optimization discipline (partitioning, clustering)
- CI/CD for LookML and dbt code
- User training and change management
- Clear cost controls and budget ownership

---

## Risks & Mitigations
| Risk | Mitigation |
|------|------------|
| Cost overrun | Budget alerts, query limits, reserved slots, usage dashboards |
| Adoption resistance | Training, key-user involvement, change management |
| Data leaks | RLS/CLS, RBAC, hospital data segregation |
| Pipeline failures | Cloud Composer alerting, retry policies, self-healing |
| Vendor lock-in | CMEK, Data Boundary, open format support |

---

## Action Items for DBA/Data Engineers
1. Review current Sybase extraction points for CDC/replication planning
2. Define partitioning and clustering strategy for large tables
3. Establish naming conventions and data modeling standards
4. Plan dbt project structure and testing framework
5. Set up GCS lifecycle policies
6. Configure IAM roles and row-level security rules
7. Build cost monitoring dashboards
8. Document data lineage requirements
9. Prepare training plan for BigQuery SQL and dbt
