# GCP GitHub Actions Workload Identity Federation



## Purpose & Scope

This standard covers GitHub Actions authentication to Google Cloud through Workload Identity Federation, a Workload Identity Pool, an OIDC provider, and service-account impersonation. It applies to GCP projects, GitHub repositories, service accounts, Cloud Storage, GKE, Terraform, and CI/CD deployments.

Out of scope: human IAM, GKE runtime identity where separate, service-account JSON keys, and organization-specific log-retention policies.

## Control Objective & Compliance Alignment

- Eliminate service-account JSON keys from GitHub and runner storage.
- Validate OIDC issuer, audience, repository, branch/tag/environment, and approved principal attributes.
- Use dedicated service accounts and least-privilege IAM bindings.
- Prevent public pools/providers and broad `principalSet` impersonation.
- Preserve Cloud Audit Logs, Cloud Asset/Policy evidence, GitHub audit logs, and Terraform evidence.

## Technical Architecture & Implementation Details

### Authentication Flow

1. GitHub Actions requests an OIDC token with `permissions: id-token: write`.
2. Google STS validates issuer and audience through the pool provider.
3. Provider attribute mapping and conditions identify the external principal.
4. IAM authorizes the principal-set to impersonate a GCP service account.
5. The service account IAM policy determines project/resource permissions.

### Workload Identity Pool and Provider

```hcl
resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
  display_name               = "GitHub Actions"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-oidc"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  attribute_condition = "assertion.repository == 'my-org/my-app'"

  oidc { issuer_uri = "https://token.actions.githubusercontent.com" }
}
```

### Service-Account Impersonation

```hcl
resource "google_service_account" "github_deployer" {
  account_id   = "github-prod-deployer"
  display_name = "GitHub Production Deployer"
}

resource "google_service_account_iam_member" "workload_identity_user" {
  service_account_id = google_service_account.github_deployer.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/my-org/my-app"
}
```

For production, add exact branch/environment or repository plus ref constraints and use separate service accounts and pools/providers where trust boundaries require it.

### GitHub Workflow

```yaml
permissions:
  contents: read
  id-token: write

- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
    service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

- uses: google-github-actions/setup-gcloud@v2
```

Enable IAM Credentials, STS, and required APIs such as Container or Storage before deployment.

## Operational Controls, Risks & Guardrails

- Do not grant `roles/owner`, broad project Editor, or `objectAdmin` unless justified.
- Block public or `allUsers`/`allAuthenticatedUsers` IAM policy bindings.
- Use organization policy and VPC Service Controls for data boundaries.
- Audit workload pool/provider changes, service-account impersonation, key creation, and IAM bindings.
- Separate CI/CD deployment identity from GKE pod runtime identity.
- Store Terraform state remotely and restrict state access, state history, and apply permissions.

## Evidence & Audit Checklist

- Workload pool/provider, provider mappings, conditions, and issuer export.
- Service-account inventory, owner, key inventory, and IAM bindings.
- `roles/iam.workloadIdentityUser` principal and principal-set evidence.
- GitHub repository, workflow, environment, and protected-runner configuration.
- Cloud Audit Logs for STS and IAM Credentials impersonation.
- IAM policy, Organization Policy, VPC Service Controls, and Cloud Asset evidence.
- Terraform plan/apply and remote-state evidence.
- Key-removal and exception approvals.

## Scenario-Based Questions and Engineering Resolutions

### 1. GitHub receives `PERMISSION_DENIED` from STS
**Question:** External identity authentication is denied. What is checked?  
**Answer:** Verify provider issuer/audience/attribute mapping, exact repository/ref condition, pool/provider status, project number, service-account binding, and GitHub `id-token` permission. Compare STS audit context with the workflow claim set.

### 2. A broad `principalSet` is used
**Question:** A pool-wide principal set impersonates the service account. How is it remediated?  
**Answer:** Replace the binding with repository-, environment-, or verified-attribute-specific principal sets. Audit all existing impersonation usage and remove unused bindings.

### 3. Attribute condition is always true
**Question:** The condition omits repository restrictions. What is the risk?  
**Answer:** Every principal in the pool/provider may match. Require exact repository and approved ref/environment attributes, test with claim variants, and review audit logs for unexpected impersonation.

### 4. Workflow uses project ID instead of project number
**Question:** The provider resource name is malformed. What is the fix?  
**Answer:** Construct the provider resource name with the numeric project number, not project ID: `projects/NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER`.

### 5. Service-account IAM denies impersonation
**Question:** STS succeeds but impersonation fails. What is missing?  
**Answer:** Add `roles/iam.workloadIdentityUser` to the exact federated principal set on the target service account. Granting roles at project or pool scope does not replace the service-account IAM binding.

### 6. The service-account key is still in GitHub Secrets
**Question:** WIF is configured but JSON keys remain. What is required?  
**Answer:** Remove and disable keys, delete repository secrets, audit key usage, purge images/logs as feasible, and verify workflows exclusively use WIF.

### 7. Production deploy succeeds from a test branch
**Question:** Repository-only conditions allow unintended refs. How is this fixed?  
**Answer:** Add exact branch, tag, or GitHub Environment attributes supported by the token. Use separate service accounts and protected environments. Test actual OIDC claims before rollout.

### 8. A forked pull request accesses GCP
**Question:** How is this blocked?  
**Answer:** Do not issue production credentials to forks. Restrict provider conditions and service-account impersonation to approved repository/ref/environment, and use read-only/no credential roles for pull requests.

### 9. GKE deployment receives insufficient permissions
**Question:** The CI identity authenticates but cannot deploy. How is it isolated?  
**Answer:** Separate GCP IAM, Kubernetes API credentials, GKE IAM roles, and namespace RBAC. Grant only container/deployment actions and verify private-cluster networking and cluster credentials.

### 10. Pod runtime identity and CI/CD identity are confused
**Question:** A service account works for GitHub but pods cannot access GCS. What is missing?  
**Answer:** Configure GKE Workload Identity separately: KSA annotation, IAM Workload Identity User binding to the GSA, and node cluster settings. CI/CD impersonation does not configure pod runtime identity.

### 11. A Terraform apply creates an unintended owner binding
**Question:** How is impact contained?  
**Answer:** Remove the broad binding, search Cloud Asset Inventory and IAM policies, review Terraform state/apply, and investigate service-account activity. Replace it with a custom or resource-scoped role.

### 12. VPC Service Controls block access
**Question:** IAM permissions are valid but the API is denied. What is diagnosed?  
**Answer:** Inspect service perimeter ingress/egress, project membership, protected resources, access levels, and service-account identity. IAM alone does not override VPC Service Controls.

### 13. Organization Policy blocks service-account key creation
**Question:** Existing key still works. What should happen?  
**Answer:** The policy does not necessarily revoke existing keys. Disable/delete them, remove workflow references, and audit historical key use. Confirm the policy is enforced at the correct organization/folder/project scope.

### 14. Terraform state changes pool configuration out-of-band
**Question:** How is drift reconciled?  
**Answer:** Export Cloud Asset and provider configuration, identify the actor/change approval, reconcile or import it, and restrict console/CI permissions. Alert on provider condition or IAM binding drift.

### 15. A repository can impersonate a different project account
**Question:** IAM policy reveals several service accounts. How is blast radius prevented?  
**Answer:** Bind exact repository/principal sets to dedicated accounts per environment, remove shared identities, and use separate pools/projects for strong trust boundaries.

### 16. A third-party GitHub action receives a credential
**Question:** What controls apply?  
**Answer:** Pin actions to immutable commits, restrict marketplace actions, use minimal permissions, isolate credentials, and review action source. Federated credentials are usable by code executed in the job.

### 17. Attribute mapping is missing `google.subject`
**Question:** What fails and why?  
**Answer:** Identity uniqueness and principal resolution may fail. Map `google.subject` to `assertion.sub` or the appropriate verified claim and ensure provider mappings match deployed workload identity.

### 18. Cloud Audit Logs show an unexpected principal
**Question:** How is the incident investigated?  
**Answer:** Correlate principal name, attribute claims, service-account use, GitHub repository/run, IP/session context, and IAM bindings. Suspend impersonation if unauthorized and preserve evidence.

### 19. The service account has only `roles/storage.objectViewer`
**Question:** Deployment to GCS is denied. What is the answer?  
**Answer:** Viewer permits reads, not writes. Apply the minimal writer/creator role or custom role for only required buckets/prefixes and preserve read-role separation.

### 20. A private GKE cluster denies GitHub runner access
**Question:** How is access designed?  
**Answer:** Run a runner with approved private-network reachability or use an approved private API/reverse proxy. Separate network connectivity from IAM authorization and do not open public control-plane access by convenience.

### 21. An organization policy blocks external access
**Question:** What is the distinction between authentication and policy?  
**Answer:** External identity authentication may succeed, but IAM, Organization Policy, VPC Service Controls, or resource policy can deny the API. Diagnose each layer and use approved boundary exceptions only.

### 22. Cloud Audit logs lack impersonation activity
**Question:** What evidence is missing?  
**Answer:** Enable relevant Admin/Data Access audit logs, route logs to protected storage, ensure aggregation, and correlate STS/IAM Credentials with GitHub audit records.

### 23. A break-glass service account key is used
**Question:** How should it be handled?  
**Answer:** Correlate approval and incident, disable or delete the key, investigate service-account activity, rotate/recreate affected resources if needed, and require WIF or break-glass controls for normal operations.

### 24. The provider issuer is changed
**Question:** Existing workflows fail. How is migration performed?  
**Answer:** Create a new verified provider and test in staged projects. Update bindings and workflow variables, revoke obsolete providers/bindings, and preserve audit evidence for issuer and provider changes.

### 25. An auditor finds a service account has no owner
**Question:** What governance action follows?  
**Answer:** Assign owner, purpose, environment, review date, and authorization. Review all bindings and keys, remediate or retire unused identities, and add recurring access certification.

## Assumptions and Missing Details

The source does not specify GCP project numbers, organization/folder policy, GitHub organization, deployment branches, GKE architecture, log retention, or approved custom roles. These must be supplied by control owners.
