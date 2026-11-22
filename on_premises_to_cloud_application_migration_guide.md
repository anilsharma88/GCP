# On-Premises to Cloud Application Migration Guide

A structured, end-to-end framework for planning, executing, and validating an application migration from on-premises infrastructure to the cloud (AWS, GCP, or Azure).

---

## 1. Phase 1: Discovery & Assessment

Before provisioning any cloud infrastructure, establish complete visibility into the existing application footprint:

* **Application & Dependency Mapping:** Identify all tiers (web, application, database, caches, message queues, scheduled cron jobs) and map upstream/downstream integrations, third-party APIs, and hardcoded IP addresses.
* **Performance Baselines & Right-Sizing:** Collect historical CPU, memory, disk IOPS, storage growth, and network throughput metrics. Avoid lifting and shifting over-provisioned peak hardware specs; right-size for cloud elasticity.
* **Business Constraints (RTO & RPO):**
  * **RTO (Recovery Time Objective):** Maximum acceptable downtime during the cutover window.
  * **RPO (Recovery Point Objective):** Maximum acceptable data loss (determines whether batch backup/restore or real-time replication is required).
* **Security, Compliance & Licensing:** Verify data residency requirements (e.g., GDPR, HIPAA, local regulations) and review software licensing terms (Windows Server, SQL Server, Oracle BYOL vs. license-included).

---

## 2. Phase 2: Migration Strategy ("The 6 Rs")

Select the migration path that aligns with business timelines, budget, and technical debt:

| Strategy | Description | Best Used When |
| :--- | :--- | :--- |
| **Rehost** *(Lift & Shift)* | Move VMs directly to cloud VMs (EC2 / Compute Engine / Azure VM) with minimal or no code changes. | Strict deadlines (e.g., data center exit) or legacy systems where source code cannot be altered. |
| **Replatform** *(Lift, Tinker & Shift)* | Retain core application code, but adopt managed cloud services (e.g., self-hosted DB to RDS/Cloud SQL, or containerizing onto EKS/GKE/AKS). | You want immediate operational relief and managed availability without a full rewrite. |
| **Refactor / Re-architect** | Redesign the application into cloud-native microservices, containers, or serverless/event-driven architectures. | Monolithic bottlenecks prevent scaling or rapid feature delivery. |
| **Repurchase** | Replace custom on-premises software with a commercial SaaS product (e.g., migrating on-prem CRM to Salesforce). | Commodity business capabilities with no competitive differentiation. |
| **Retire** | Decommission unused applications, legacy endpoints, or zombie services discovered during assessment. | 10–20% of an on-premises estate is typically redundant or no longer in active use. |
| **Retain** | Keep specific workloads on-premises temporarily or permanently. | Workloads tied to specialized physical hardware, mainframes, or strict local compliance. |

---

## 3. Phase 3: Target Architecture & Cloud Landing Zone

Provision the cloud foundation using **Infrastructure as Code (Terraform / OpenTofu)** for repeatability and auditability:

* **Networking & Hybrid Connectivity:**
  * Design VPC/VNet topology with isolated public and private subnets across multiple Availability Zones.
  * Establish secure hybrid connectivity (**Site-to-Site VPN** or dedicated **Direct Connect / Cloud Interconnect / ExpressRoute**) so on-premises and cloud components can communicate securely during migration.
* **Identity, Security & Secrets:**
  * Enforce least-privilege IAM roles and service accounts.
  * Migrate hardcoded credentials and config files into managed vaults (AWS Secrets Manager, GCP Secret Manager, Azure Key Vault).
  * Enforce encryption in transit (TLS) and at rest (KMS/CMEK).
* **Observability Baseline:** Configure centralized logging, metrics, alerting, and distributed tracing *before* cutover so cloud behavior can be directly compared against on-premises baselines.

---

## 4. Phase 4: Migration Execution (Compute & Data)

Separate stateless application tiers from stateful data tiers:

### A. Pilot & Non-Production Validation
* Migrate a non-production environment (`dev`/`staging`) first to uncover hidden firewall rules, DNS resolution gaps, latency bottlenecks, and authentication issues.

### B. Database Migration (Minimal Downtime)
* **Small Databases / Flexible Downtime Window:** Perform a native backup/dump on-premises and restore it into the managed cloud database.
* **Large Databases / Near-Zero Downtime:**
  1. Take a consistent initial snapshot and restore it to the target cloud database.
  2. Enable continuous **Change Data Capture (CDC)** replication (via **AWS DMS**, **GCP Database Migration Service**, **Azure Database Migration Service**, or native logical replication) to stream ongoing writes in real time until cutover.

### C. Application & File Storage Migration
* Sync static files and shared network drives to cloud object storage (S3 / GCS / Azure Blob) or managed file shares using `rsync`, AWS DataSync, Storage Transfer Service, or AzCopy.
* Deploy stateless application instances/containers in the cloud configured against the synced storage and read-only target replica for pre-cutover verification.

---

## 5. Phase 5: Testing, Cutover & Rollback Plan

### Pre-Cutover Validation
* Execute functional smoke tests, end-to-end integration tests, security/vulnerability scans, and load/stress testing against the cloud stack.

### Cutover Execution
1. **Lower DNS TTLs:** Reduce DNS record TTLs (e.g., to `60` seconds) 24–48 hours prior to cutover to ensure rapid traffic propagation.
2. **Freeze Writes & Sync:** Place the on-premises application into a brief maintenance/read-only window and confirm CDC replication lag reaches `0`.
3. **Promote Target:** Promote the cloud database to primary read/write mode.
4. **Shift Traffic:** Switch DNS or load balancer routing to the cloud environment (either via **Blue-Green full switch** or **Canary weighted routing**: 5% $\rightarrow$ 25% $\rightarrow$ 100%).
5. **Verify:** Monitor error rates, latency ($p_{95}/p_{99}$), database connections, and business transactions.

### Rollback Plan
* Keep the on-premises environment warm (and reverse-replicate new cloud writes if feasible) for a defined stabilization window (e.g., 48–72 hours) so traffic can be reverted immediately if a critical P1 issue arises.

---

## 6. Phase 6: Post-Migration Optimization (Day 2 Operations)

* **FinOps & Right-Sizing:** Analyze actual cloud resource utilization over the first 2–4 weeks, tune autoscaling policies, configure storage lifecycle tiers, and purchase Savings Plans, Reserved Instances, or Committed Use Discounts (CUDs).
* **Decommissioning:** Once stability targets are met, safely archive final backups, tear down temporary VPN/migration pipelines, and decommission on-premises servers.

---

## 7. Quick Interview Summary Pitch

> *"I approach an on-premises to cloud migration in six structured phases: first, **assessing** application dependencies, performance baselines, and RTO/RPO targets to pick the right **6R strategy**; second, building an IaC-driven **landing zone** with secure hybrid networking and observability; third, migrating **stateful databases via snapshot + continuous CDC replication** alongside stateless compute; fourth, executing a **low-TTL DNS or canary cutover** backed by a tested rollback plan; and finally, running **post-migration right-sizing and FinOps optimization** before decommissioning legacy hardware."*