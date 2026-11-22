# Multi-Cloud On-Premises to Cloud Application Migration

**Document status:** Compliance portal draft  
**Original filename:** `on_premises_to_cloud_application_migration_guide.md`  
**Filename rationale:** Replaced underscores with the compliance-ready kebab-case convention and clarified that the migration spans AWS, Azure, and GCP.

## Purpose & Scope

This standard defines governance and implementation for migrating applications and data from on-premises environments to AWS, Azure, and GCP. It covers discovery, 6R strategy, landing zones, identity, networking, data migration, cutover, rollback, validation, and post-migration operations.

Out of scope: data-center physical decommissioning, application redesign funding, workforce restructuring, and organization-specific RTO/RPO values.

## Control Objective & Compliance Alignment

- Protect confidentiality, integrity, availability, and sovereignty during migration.
- Use least-privilege identities and controlled hybrid connectivity.
- Preserve separation of duties, audit logs, change approvals, and rollback evidence.
- Validate data lineage, encryption, replication lag, access, and recovery readiness.
- Prevent unplanned cutover, public exposure, credential loss, or destructive rollback.

## Technical Architecture & Implementation Details

### 1. Discovery and Assessment

Inventory application tiers, dependencies, API/endpoints, scheduled jobs, databases, storage, certificates, network rules, performance, RTO/RPO, licensing, data classification, residency, and exceptions.

Outputs include dependency diagrams, data-flow diagrams, migration candidates, 6R decisions, target architecture, owner, controls, and accepted residual risk.

### 2. Strategy Mapping

| Strategy | Use case | Typical risk control |
|---|---|---|
| Rehost | Minimal change, short deadline | Baseline configuration and image hardening |
| Replatform | Managed service adoption | Schema, identity, and service-control validation |
| Refactor | Cloud-native redesign | Architecture/security review and incremental rollout |
| Repurchase | SaaS replacement | Contract, export, privacy, and identity integration |
| Retire | Redundant workload | Evidence-backed decommission and restore hold |
| Retain | Hardware, legal, or technical dependency | Compensating controls and documented exception |

### 3. Landing Zone

Provision networking, identity, encryption, logging, policy, secrets, and observability using version-controlled Terraform/OpenTofu:

- AWS: accounts, Organizations/SCP, VPC, IAM, KMS, CloudTrail, Config, GuardDuty.
- Azure: management groups/subscriptions, Entra, RBAC, Policy, VNets, Key Vault, Activity/diagnostic logs.
- GCP: organizations/folders/projects, IAM, VPC, KMS, Cloud Audit Logs, Organization Policy, SCC.

Require separate dev/test/prod identities, encrypted state, state locking, protected apply, pull-request approval, and drift detection.

### 4. Data and Compute Migration

Use full or incremental database migration based on downtime and consistency requirements:

- AWS: DMS, native replication, backup/restore, EBS/S3 lifecycle controls.
- Azure: Database Migration Service, native replication, backup/restore, Blob/Files controls.
- GCP: Database Migration Service, native replication, backup/restore, Cloud Storage controls.

For files, synchronize with checksums and allow read-only target validation. Encrypt in transit and at rest and restrict migration endpoints and identities.

### 5. Cutover and Rollback

1. Freeze writes or apply a controlled maintenance window.
2. Verify final replication lag and data integrity.
3. Lower DNS TTL where DNS is used.
4. Deploy and smoke-test the target.
5. Shift traffic using blue-green or canary controls.
6. Monitor SLOs, database replication, errors, security, and business transactions.
7. Initiate rollback against agreed thresholds and preserve source compatibility.
8. Confirm stabilization and decommission only after approval.

## Operational Controls, Risks & Guardrails

- Block migration credentials in source images, shell history, repositories, and logs.
- Separate migration, application, database, and approval identities.
- Validate DNS, firewall, private connectivity, certificates, and routing before cutover.
- Treat large or sensitive data as protected by retention, residency, and classification controls.
- Test rollback repeatedly and maintain source and target recovery points.
- Alert on replication lag, backup failures, IAM changes, public exposure, and policy drift.

## Evidence & Audit Checklist

- Approved discovery inventory and dependency maps.
- RTO/RPO, data classification, legal/residency, and licensing approvals.
- 6R decision record and architecture diagrams.
- Landing-zone policy/IaC/plan/apply evidence.
- Database and file migration plans, checksums, lag, and reconciliation evidence.
- Cutover approval, DNS/load-balancer changes, smoke test, and monitoring evidence.
- Rollback test and incident criteria.
- IAM, network, encryption, logging, and vulnerability evidence.
- Post-migration optimization, decommission approval, and residual-risk acceptance.

## Scenario-Based Questions and Engineering Resolutions

### 1. Discovery misses a database dependency
**Question:** Cutover reveals a hardcoded application dependency to an unlisted database. How should this be prevented?  
**Answer:** Expand discovery with traffic, DNS, package, configuration, database, connection, and network-flow analysis. Perform dependency scoring and pilot testing. Add an outage gate and rollback plan for unknown dependencies.

### 2. RTO/RPO are not validated
**Question:** Business requested zero downtime but only backup/restore is planned. What action applies?  
**Answer:** Challenge the strategy and obtain explicit RTO/RPO approval. Add CDC/replication or a zero-downtime architecture only if tested. Document residual recovery risk and escalation criteria.

### 3. Migration uses the wrong 6R strategy
**Question:** A legacy app with unsupported OS is rehosted despite no migration path. What is done?  
**Answer:** Reassess support, security, compatibility, and lifetime cost. Choose replatform/refactor/retain/repurchase and document the exception and upgrade plan rather than merely lifting unsupported infrastructure.

### 4. Cloud landing zone has no SCP/policy controls
**Question:** The target account/project is exposed to uncontrolled resources. How is it hardened?  
**Answer:** Apply organization policy, SCP, Azure Policy, restricted identities, network baselines, encryption, logging, and protected state. Reconcile existing resources before migration.

### 5. Terraform state is public or shared
**Question:** State can expose secrets and infrastructure details. What is the remediation?  
**Answer:** Restrict backend access, encryption and state locks, remove public access, rotate exposed secrets, and review historical state. Add logs and separate state backends by environment.

### 6. Migration runner has broad cloud privileges
**Question:** A migration identity can reach all accounts or subscriptions. How is blast radius reduced?  
**Answer:** Use dedicated source/target identities, time-bound migration credentials, exact resource scopes, and separate plan/deploy/rollback roles. Review CloudTrail/Activity/Cloud Audit logs.

### 7. DNS TTL is not lowered early enough
**Question:** Traffic remains on the old environment after cutover. What is done?  
**Answer:** Stop the cutover if needed, correct DNS ownership/TTL, validate global propagation and health, then shift with controlled canary or rollback. Define TTL timing in the runbook.

### 8. Replication lag is non-zero at cutover
**Question:** The database source continues to change. What is required?  
**Answer:** Do not promote until lag reaches the approved threshold and source writes are frozen or otherwise reconciled. Re-run reconciliation and retain rollback.

### 9. Database schema differs after migration
**Question:** Application queries fail on the target. How is validation performed?  
**Answer:** Compare schema, objects, versions, extensions, collation, users/grants, and test queries. Run application integration tests and reconcile DDL or application compatibility defects before traffic shifts.

### 10. Storage sync omits recent files
**Question:** Target data is incomplete. What is checked?  
**Answer:** Re-run incremental sync, compare counts/checksums/object inventories, verify exclusions and source locks, and record reconciliation. Block cutover on unresolved data differences.

### 11. Private endpoint DNS is wrong
**Question:** Apps use public endpoints or fail name resolution. How is it resolved?  
**Answer:** Correct private DNS zones/resolvers and endpoint routes, test source/target resolution, and verify firewall/security groups. Do not bypass with public connectivity if the policy requires private access.

### 12. Firewall rules allow production to all networks
**Question:** Migration introduces overbroad network access. What is the remediation?  
**Answer:** Replace broad CIDR rules with approved source/destination/service identities, private connectivity, and application ports. Review flow logs and remove temporary cutover rules.

### 13. TLS certificates use wrong names or expire
**Question:** Cutover traffic gets certificate errors. How is it handled?  
**Answer:** Validate names, chain, trust stores, expiry, HSM/KMS integration, and load-balancer listeners before cutover. Renew or rotate through approved certificate control.

### 14. Static database credentials are embedded in migration scripts
**Question:** What is required?  
**Answer:** Move secrets to AWS Secrets Manager, Azure Key Vault, or GCP Secret Manager; use short-lived or workload identity where supported; revoke exposed credentials; and scan Git history/artifacts.

### 15. Monitoring baseline was not established
**Question:** Post-cutover latency cannot be compared. How is it prevented?  
**Answer:** Deploy logs, metrics, traces, synthetic checks, alerts, and baseline dashboards before migration. Compare target p50/p95/p99, errors, saturation, database, and business metrics to source baselines.

### 16. Canary traffic reveals severe errors
**Question:** What rollback criteria apply?  
**Answer:** Stop traffic expansion or shift back to source based on approved error/SLO thresholds. Preserve target logs/data, investigate, and retry only after root-cause remediation and approval.

### 17. Rollback cannot reverse database writes
**Question:** The target accepted writes and source is stale. How is rollback performed?  
**Answer:** Do not use a blind traffic reversal. Stop writes, reconcile new writes to source or preserve an export, promote the correct database state, and execute a tested data-reversal procedure.

### 18. Decommission happens before stabilization
**Question:** The source is destroyed immediately after cutover. What should change?  
**Answer:** Retain source data/systems for an approved stabilization period with backups and isolation. Require success, rollback-window expiry, and owner/security approval before decommission.

### 19. Public object storage is exposed during testing
**Question:** Migration files become publicly accessible. What is required?  
**Answer:** Remove public access, block public ACL/policy exposure, classify affected data, review access logs, rotate or delete sensitive artifacts, and apply organization/public-access controls.

### 20. A migration tool lacks audit evidence
**Question:** Operations are not correlated to approvals. How is evidence restored?  
**Answer:** Enforce approved pipelines, immutable logs, change tickets, runner identities, artifact signatures, Terraform plans, database logs, and source/target checksum evidence.

### 21. FinOps right-sizing starts before stabilization
**Question:** Resources are resized during active cutover. What is the risk?  
**Answer:** Maintain the stabilization baseline and freeze architecture changes. Use evidence-based right-sizing after SLO stability and capacity sampling, with rollback or autoscaling safeguards.

### 22. Application licensing is incompatible in the target
**Question:** The migration is blocked after provisioning. How is this avoided?  
**Answer:** Complete licensing, OS, database, processor, geography, and SaaS entitlement review during discovery. Re-host or repurchase according to contract and approval.

### 23. A data residency condition is violated
**Question:** The target region or backup replicates data outside the approved geography. What action applies?  
**Answer:** Stop affected transfers, inventory replicas/backups, correct region/policy settings, consult data owners/legal, and document remediation or an approved exception.

### 24. Cutover succeeds but rollback testing was not completed
**Question:** Stabilization begins anyway. What is required?  
**Answer:** Do not assume rollback readiness. Complete a table-top or technical rollback drill, measure recovery time/data impact, retain source systems/backups, and obtain approval for the observed gap.

### 25. Post-migration cost rises unexpectedly
**Question:** How is cost corrected without risking reliability?  
**Answer:** Attribute usage to workloads, inspect idle resources, storage tiers, backups, replication, network egress, commitment coverage, and autoscaling. Optimize in phases with SLO and architecture review.

## Assumptions and Missing Details

The source does not specify target providers, application names, RTO/RPO, data classification, residency, migration tools, cutover duration, stabilization period, or regulatory framework. These must be supplied by control owners.
