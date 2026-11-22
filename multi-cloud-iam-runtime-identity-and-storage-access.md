# Multi-Cloud IAM Runtime Identity and Storage Access

**Document status:** Compliance portal draft  
**Original filename:** `iam-role-olicy.md`  
**Filename rationale:** The original name contains an apparent spelling error, does not describe the technology and control domain, and uses an ambiguous acronym. The standardized name identifies the scope as multi-cloud IAM runtime identity and storage access.

## Purpose & Scope

This control standard defines how AWS, Microsoft Azure, and Google Cloud Platform (GCP) virtual-machine workloads obtain short-lived credentials and access object storage without embedding static keys. It covers:

- AWS EC2 IAM roles, instance profiles, IAM policies, S3, and KMS.
- Azure virtual machines, managed identities, Microsoft Entra ID, Azure RBAC, and Blob Storage.
- GCP Compute Engine service accounts, IAM role bindings, metadata credentials, and Cloud Storage.
- Terraform-managed provisioning, runtime verification, monitoring, and audit evidence.

**Out of scope:** workstation identities, human identity lifecycle, secret-manager implementation, network egress controls, and internal password or privileged-access standards. Those controls may be documented separately.

## Control Objective & Compliance Alignment

The control objective is to ensure workload access is authenticated through dedicated, short-lived identities, authorized with least privilege, and consistently logged and reviewed.

- **Least privilege:** Grant only required object and prefix/container actions at the narrowest supported scope.
- **Separation of duties:** Separate workload identity administration, storage administration, key management, Terraform deployment, and audit review.
- **Access governance:** Require ownership, environment, purpose, expiration or review date, and periodic access certification.
- **Auditability:** Record identity attachment, role assignments/policy changes, storage API activity, metadata or token activity, and Terraform state changes.
- **Short-lived credentials:** Use cloud runtime credentials or workload federation instead of access keys, account keys, SAS tokens, or downloaded service-account keys.
- **Non-interactive workload identity:** Do not assign interactive administrator roles to VM identities or reuse identities across unrelated environments.

Implementation details and retention periods are based on common engineering practice; organization-specific approval, retention, and severity requirements must be supplied by policy owners.

## Technical Architecture & Implementation Details

### Cross-Cloud Identity Mapping

| Concept | AWS | Azure | GCP |
|---|---|---|---|
| Workload identity | IAM role in an instance profile | System- or user-assigned managed identity | Service account |
| Permission policy | IAM identity policy, resource policy, SCP, permissions boundary | Azure RBAC role definition and role assignment | IAM role binding and organizational constraints |
| Resource scope | Account, ARN, bucket, prefix, key | Management group, subscription, resource group, storage account, container | Organization, folder, project, bucket, object |
| Runtime credential delivery | EC2 Instance Metadata Service v2 | Azure Instance Metadata Service / Entra OAuth token | GCE metadata server / Application Default Credentials |
| Privilege attachment | Attach instance profile to EC2 | Assign managed identity to VM; assign RBAC to identity | Attach service account to VM; grant IAM binding |

### AWS — EC2 IAM Role

1. Create an IAM role with a trust policy limited to `ec2.amazonaws.com`.
2. Create a permission policy restricted to the target bucket and object prefix; use separate statements for bucket and object resources.
3. Attach a permissions boundary and organization SCP where required.
4. Create an instance profile and attach it to the EC2 instance.
5. Require IMDSv2, block IMDS token forwarding, and limit network access to metadata where possible.
6. Enable CloudTrail, S3 data events, KMS key usage logs, GuardDuty, and Config.

Reference trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

Reference permissions policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {"Effect": "Allow", "Action": "s3:ListBucket", "Resource": "arn:aws:s3:::example-app-prod"},
    {"Effect": "Allow", "Action": ["s3:GetObject", "s3:PutObject"], "Resource": "arn:aws:s3:::example-app-prod/app/*"}
  ]
}
```

For SSE-KMS, additionally grant narrowly scoped `kms:Decrypt` and `kms:GenerateDataKey` to both IAM and the KMS key policy. Do not grant `s3:*`, `kms:*`, or wildcard object resources in production without a documented exception.

### Azure — Managed Identity and RBAC

1. Prefer a user-assigned managed identity for independent lifecycle management.
2. Attach it to the intended virtual machine.
3. Assign a data-plane role such as `Storage Blob Data Reader` or `Storage Blob Data Contributor` at the storage account or container scope.
4. Use `DefaultAzureCredential`, `az login --identity`, or an Azure SDK without account keys or SAS tokens.
5. Disable shared-key and SAS access when not required; configure private connectivity and storage firewall controls where applicable.
6. Enable Microsoft Entra sign-in logs, Azure Activity Logs, Storage Data Plane Logs, Defender for Storage, and Azure Policy.

Example role assignment:

```bash
az role assignment create \
  --assignee-object-id <MANAGED-IDENTITY-PRINCIPAL-ID> \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope <STORAGE-CONTAINER-RESOURCE-ID>
```

Azure `Contributor` or `Owner` is a management-plane role and does not grant blob data access by itself. A principal may receive control-plane rights and data-plane rights separately.

### GCP — Service Account and IAM

1. Create a dedicated service account per workload and environment.
2. Grant a bucket or project role directly to `serviceAccount:<email>`.
3. Attach that service account to Compute Engine with `scopes = ["cloud-platform"]` when required, while controlling authorization with IAM rather than legacy OAuth scopes.
4. Prefer `roles/storage.objectViewer`, `roles/storage.objectCreator`, or an appropriately restricted custom role over `roles/storage.admin` or `roles/storage.objectAdmin`.
5. Prevent automatic or user-managed service-account keys.
6. Enable Cloud Audit Logs, Data Access logs for Storage, Security Command Center, and Organization Policy.

Example IAM binding:

```bash
gcloud storage buckets add-iam-policy-binding gs://example-app-prod \
  --member="serviceAccount:app-prod@example-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

### Terraform Guardrails

Terraform declarations should separate identity, permissions, VM attachment, and storage resources by environment. State files must be remote, encrypted, access-controlled, versioned, and excluded from source control. CI/CD applies should use workload identity federation rather than persisted cloud credentials.

Recommended Terraform controls include:

- Separate state backends or state namespaces for AWS accounts, Azure subscriptions, and GCP projects.
- Pull-request review for IAM and role-assignment changes.
- Automated policy validation for wildcard resources/actions, public bindings, and key creation.
- Drift detection and reconciliation in managed pipelines.
- Plan/apply separation or protected environments for production.
- Explicit user or service identities for state-lock and backend access.

Example AWS attachment:

```hcl
resource "aws_iam_instance_profile" "app" {
  name = "app-prod-profile"
  role = aws_iam_role.app.name
}

resource "aws_instance" "app" {
  iam_instance_profile = aws_iam_instance_profile.app.name

  metadata_options {
    http_tokens                 = "required"
    http_endpoint               = "enabled"
    http_put_response_hop_limit = 1
  }
}
```

## Operational Controls, Risks & Guardrails

| Risk or misconfiguration | Guardrail and remediation |
|---|---|
| Hardcoded AWS keys, Azure keys/SAS, or GCP key JSON | Revoke, rotate, remove from images and repositories, then replace with runtime identity. Search Git history and artifact registries. |
| Overbroad role or wildcard resource | Replace with workload-specific custom policy/role and environment-specific scope; review generated policy with IAM policy analyzers. |
| Default or shared VM identity | Create dedicated identities; prohibit default service accounts/identities outside approved exceptions. |
| Incorrect data/control-plane role | AWS: separate KMS and S3 policy evaluation. Azure: add a data-plane role. GCP: verify both IAM role binding and legacy OAuth scope. |
| Metadata SSRF credential theft | AWS require IMDSv2; Azure require `Metadata: true`; GCP require `Metadata-Flavor: Google`; also patch SSRF, restrict hops, and minimize exposed metadata. |
| Cross-account/project access | Explicitly authorize both caller and target sides in AWS; validate Entra tenant and RBAC scope in Azure; bind the principal in GCP and assess org-policy constraints. |
| Terraform state leakage or drift | Encrypt and restrict remote state, redact secrets, require state locks, perform drift checks, and reject unreviewed infrastructure changes. |
| Privilege escalation during deployment | Restrict role-pass, managed-identity operator, service-account impersonation, and role-grant permissions to deployment personnel and protected pipelines. |
| External or public service account binding | Block public/allUsers/allAuthenticatedUsers bindings; review domain and principal types. |
| Logging gaps | Enable control-plane and data-plane audit data; route logs to immutable storage; alert on denied requests, role changes, key creation, and policy drift. |

## Evidence & Audit Checklist

A reviewer should retain the following evidence, appropriate to the organization’s approved retention schedule:

- Approved identity inventory with owner, workload, environment, and review date.
- IAM role, managed identity, or service-account configuration export.
- Trust policy, permission policy, RBAC assignment, or IAM policy binding export.
- VM attachment or Terraform resource configuration.
- AWS SCP and permissions-boundary exports where used.
- Azure role definitions, assignments, tenant scope, and storage data-plane configuration.
- GCP IAM policy, service-account key inventory, organization policy, and access scope export.
- CloudTrail, Azure Activity/Storage/Microsoft Entra logs, or GCP Cloud Audit Logs for relevant events.
- Security group/firewall/private-endpoint evidence for storage paths.
- Terraform plan, apply, state-access, lock, and state-encryption evidence.
- Key/SAS/static credential scan results and remediation tickets.
- Exception approvals and periodic access certification records.
- Verification showing direct key/secret acquisition is blocked and runtime credentials are used.

## Scenario-Based Questions and Engineering Resolutions

### 1. AWS EC2 access fails after attaching `AmazonS3FullAccess`
**Question:** An EC2 role has `AmazonS3FullAccess`, but a KMS-encrypted object cannot be read or written. What is the resolution?  
**Answer:** IAM alone is insufficient. Reproduce the denied action with CloudTrail and KMS. Add only required `kms:Decrypt` and, for writes, `kms:GenerateDataKey` plus relevant key condition context to the IAM policy and grant the role in the customer-managed KMS key policy. Correct S3 bucket encryption and deny statements may also block access. Use IAM Access Analyzer or policy simulation to validate all layers.

### 2. Azure managed identity has Contributor but blob upload fails
**Question:** A VM managed identity has `Contributor` on a storage account yet receives `AuthorizationPermissionMismatch`. What is wrong?  
**Answer:** `Contributor` is generally a control-plane role and does not grant Blob Data Plane access. Add `Storage Blob Data Contributor` or `Storage Blob Data Reader` at the storage account/container scope. Verify that Entra authentication is selected with `--auth-mode login` and remove any conflicting public-network or firewall control, if appropriate.

### 3. GCP VM role exists but gcloud reports insufficient scopes
**Question:** The service account has `roles/storage.objectAdmin`, but a VM still receives an insufficient-authentication-scope error. What should change?  
**Answer:** Align the instance legacy OAuth scopes. Set the VM service-account scopes to `cloud-platform` and control authorization in Cloud IAM. Then verify whether the service account is the VM-attached account and whether organizational policies block access. IAM roles alone cannot overcome a restrictive OAuth scope.

### 4. An SSRF reaches instance metadata
**Question:** An application vulnerability attempts to retrieve VM credentials through metadata. How is exposure reduced in each cloud?  
**Answer:** In AWS, enforce IMDSv2 and an IMDS response hop limit of 1, but patch the SSRF and restrict the application identity. Azure IMDS requires `Metadata: true`; GCP requires `Metadata-Flavor: Google`. These headers are defenses, not substitutes for secure application code. Revoke exposed credentials, investigate logs, and reduce attached permissions.

### 5. Two environments share one identity
**Question:** Dev and production VMs share the same AWS role or Azure identity or GCP service account. What is the fix?  
**Answer:** Create separate identities per workload/environment and scope their storage permissions to environment-specific resources. Update the VM attachment and Terraform modules, then revoke and retire the shared identity after confirming no workloads depend on it.

### 6. An AWS cross-account bucket request is denied
**Question:** Account A's EC2 role needs Account B's bucket. The IAM role policy appears complete. What else is required?  
**Answer:** Account B must allow the role ARN in its bucket policy or use another supported resource policy. AWS organizations/SCPs, permissions boundaries, VPC endpoint policies, account block public access settings, and KMS policies can still deny access. Test with AWS IAM Policy Simulator and CloudTrail.

### 7. Azure cross-subscription access fails
**Question:** A managed identity in Subscription A needs Blob Reader in Subscription B. What must be verified?  
**Answer:** Confirm the identity's object/principal ID, tenant, the exact data-plane role assignment scope, and propagation time. Verify the application calls Blob Data Plane with Entra authentication, not storage account keys. Cross-tenant access requires explicit tenant trust/authorization and is not solved by Azure Lighthouse alone.

### 8. GCP cross-project service-account access fails
**Question:** A VM in Project A uses a service account to read a Project B bucket, but access is denied. How is it corrected?  
**Answer:** Bind `roles/storage.objectViewer` or a custom equivalent on the bucket or appropriate project resource to `serviceAccount:sa@project-a.iam.gserviceaccount.com`. Verify the VM is attached to that service account, Project B IAM is not overridden by VPC Service Controls or an organization policy, and uniform bucket-level access is configured consistently.

### 9. Terraform attaches the wrong AWS role
**Question:** A production EC2 instance is launched with a development instance profile after a Terraform variable change. How is impact contained?  
**Answer:** Stop unauthorized access by detaching or replacing the profile, constrain `iam:PassRole` to approved role ARN patterns, review CloudTrail and S3 data events, and recreate or update the instance through reviewed Terraform. Add validation and environment guards to the module and remote-state workflow.

### 10. Azure Terraform role assignment applies to the wrong principal
**Question:** A role assignment created in Terraform has the wrong `principal_id`, granting access to another workload. What is the response?  
**Answer:** Delete or replace the assignment, audit all assignments created by that apply, identify who consumed the privilege, and verify identity object IDs, not display names. Add a principal-name/object-ID validation rule and review outputs and role-assignment diffs in protected pipelines.

### 11. GCP Terraform grants `roles/owner` to a service account
**Question:** A deployed Compute Engine service account unexpectedly receives project Owner through Terraform. How should this be remediated?  
**Answer:** Remove the broad project binding and replace it with a custom or storage-specific role. Search for all IAM bindings using state and Cloud Asset Inventory, review the state history and apply logs, and rotate credentials or investigate workload activity if the account was exposed.

### 12. A CI/CD workload can impersonate production identities
**Question:** A build pipeline can call `sts:AssumeRole`, Azure federated token exchange, or GCP service-account impersonation for production without clear approval. What is required?  
**Answer:** Configure environment-specific trust subjects and token claims, pin federated OIDC issuers and audiences, add subject/repository/ref conditions, and require protected-runner and approval controls. Remove broad impersonation/AssumeRole grants and deliver short-lived credentials only to the intended pipeline environment.

### 13. AWS role trust allows every account or principal
**Question:** An IAM role trust policy contains `*` in a principal or lacks external-ID restrictions for third-party access. What should be done?  
**Answer:** Replace wildcard principals with explicit account/role trust ARNs. For third parties, require external ID and, when supported, account and session-context conditions. Review CloudTrail for assumed-role usage and revert any broad trust added without approval.

### 14. Azure federation accepts tokens from any repository
**Question:** An application or pipeline federated credential uses only the issuer and not a repository or environment subject restriction. What is the risk and fix?  
**Answer:** Any workflow or issuer-controlled token matching the weak subject may obtain the identity. Restrict federated credentials to exact `subject` values such as repository/environment or enterprise application conditions, use separate identities by environment, and review Entra sign-in and federated credential audit logs.

### 15. GCP workload identity pool lets an untrusted identity impersonate a service account
**Question:** A pool has a weak attribute condition or a broad principalSet. How is it hardened?  
**Answer:** Restrict the Workload Identity Pool provider and IAM binding to exact issuer, repository, service, namespace, or other verified attributes. Remove `principalSet://.../*` grants, require explicit `attribute.repository` and `attribute.ref` conditions, and review service-account token usage and impersonation logs.

### 16. Break-glass access is used for an unauthorized change
**Question:** An emergency identity or role was used outside its approved incident. What evidence is required?  
**Answer:** Require a recorded incident, approval, identity owner, start/end time, and actions. Correlate AWS CloudTrail, Azure Activity/PIM/Microsoft Entra audit logs, or GCP Cloud Audit Logs and access-approval events with the incident. Revoke or expire the session, rotate any involved credentials, and document incident follow-up.

### 17. Azure PIM approval is bypassed or still over-privileged
**Question:** A break-glass or temporary role remains eligible or active indefinitely. What control should apply?  
**Answer:** Configure activation duration limits, justification and approval, MFA, and alerting through Privileged Identity Management. Use separate emergency identities, enforce periodic eligibility review, and continuously monitor activation and assignment logs. Persistent active assignment should be an exception requiring documented approval.

### 18. A data-plane denial occurs with valid control-plane permissions
**Question:** Azure blob access fails while the role can manage the storage account. How is this diagnosed?  
**Answer:** Separate management and data authorization. Confirm the VM identity has `Storage Blob Data *` RBAC at container/account scope and uses Entra OAuth rather than account keys/SAS. Check storage network rules, shared-key disablement, and Defender or policy settings. Use Azure sign-in and Storage data logs to locate the denied principal and action.

### 19. A VM identity can query or modify KMS/Secrets Manager unexpectedly
**Question:** A VM with storage access can list or decrypt unrelated secrets or keys. What action is required?  
**Answer:** Review all attached roles and resource policies. In AWS, remove broad `kms:*`, `secretsmanager:*`, and wildcard resources; in Azure, separate Key Vault RBAC and use application-specific vault scope; in GCP, remove unrelated Secret Manager/KMS bindings. Rotate exposed values and verify logs for prior use.

### 20. Static credentials are found in a VM image or source repository
**Question:** An AWS access key, Azure storage key/SAS, or GCP JSON key is discovered in Git or an image. What is the incident resolution?  
**Answer:** Immediately disable or revoke the credential, identify dependents, rotate and replace it with a runtime identity, purge secrets from source history and images where feasible, and assess whether the repository or registry is public. Review audit logs for unauthorized use and add secret-scanning and pre-deployment key-blocking controls.

### 21. Terraform state exposes identity details or secrets
**Question:** Remote state contains identity data or sensitive fields, and the state backend is accessible to broad engineering groups. How should it be secured?  
**Answer:** Restrict read/write access by team and environment, enable encryption and versioning, require state locking, use backend audit logs, redact or remove secrets where possible, and rotate any secret that is actually written to state. Separate state backends for sensitive environments and review historical state exports.

### 22. Terraform drift changes storage permissions
**Question:** IAM bindings or role assignments exist outside Terraform and do not appear in a plan. What is the response?  
**Answer:** Inventory the drift using CloudTrail/Activity Logs/Cloud Asset Inventory, determine whether it is authorized, and reconcile state with reviewed Terraform or explicitly import/manage the resource. Alert on repeated out-of-band changes and restrict console/API permission for teams that should deploy only through CI/CD.

### 23. Public or external storage principals appear unexpectedly
**Question:** An auditor discovers `allUsers`, `allAuthenticatedUsers`, broad domain values, or unknown external accounts on storage policies. What steps follow?  
**Answer:** Remove public/external bindings, inspect Cloud Audit logs and data access logs to assess exposure, classify affected data, rotate or delete sensitive objects as incident policy requires, and apply organization-level public-access prevention or policy controls. Document affected scope and remediation approval.

### 24. Storage access is denied only from private networks
**Question:** A workload VM has the correct IAM role but private-endpoint or firewall traffic is denied. How is it distinguished from IAM?  
**Answer:** Inspect network flow, DNS resolution, private endpoint/links, storage firewall rules, service endpoints, and VPC/VNet policies separately from identity authorization. Use a denied flow or access log to identify whether the failure is network or IAM. Do not add IAM privilege to compensate for a network restriction.

### 25. Cloud audit logs show a successful access denied by policy
**Question:** A reviewer sees both allow and deny results for the same identity. How should the final evaluation be determined?  
**Answer:** AWS evaluates explicit deny across applicable identity, resource, KMS, SCP, and boundary policies. Azure combines Entra authentication, Azure RBAC assignments, deny assignments, and resource firewall rules. GCP combines IAM allow, deny policy, organizational policy, VPC Service Controls, and access scopes. Reconstruct all applicable policy layers and enforce the identity at the narrowest effective scope.

## Assumptions and Missing Details

The source is a conceptual and Terraform guide and does not state organization-specific roles, environments, region, tenant/project IDs, regulatory framework, retention period, approval workflow, or incident-severity criteria. Those items must be populated by the control owner rather than inferred. Terraform provider versions and exact account identifiers are examples and should be validated against the organization’s supported baseline.
