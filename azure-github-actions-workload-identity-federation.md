# Azure GitHub Actions Workload Identity Federation



## Purpose & Scope

This standard covers Microsoft Entra Workload Identity Federation for GitHub Actions using a user-assigned managed identity or app registration and Azure RBAC. It applies to Azure subscriptions, resource groups, application deployments, Terraform, AKS, and CI/CD runners.

Out of scope: human Entra sign-in, local developer credentials, managed identities for Azure VMs or AKS pods, and enterprise-specific retention periods.

## Control Objective & Compliance Alignment

- Authenticate GitHub workflows with short-lived Entra tokens rather than client secrets or certificates.
- Bind federated credentials to exact issuer, subject, and audience claims.
- Use Azure RBAC at the narrowest supported resource scope.
- Separate environment identities and protected deployment approvals.
- Preserve Entra sign-in, audit, RBAC, workflow, and Terraform evidence.

## Technical Architecture & Implementation Details

### Authentication Flow

1. GitHub Actions requests an OIDC token for a job.
2. Entra validates the issuer, audience, and subject against a federated identity credential.
3. The app registration or user-assigned managed identity receives a token exchange.
4. Azure RBAC determines resource authorization.
5. The workflow receives only a short-lived session; no Entra secret is stored in GitHub.

### Example User-Assigned Managed Identity

```bash
az identity create --name github-prod-deployer \
  --resource-group rg-platform --location centralindia

CLIENT_ID=$(az identity show --name github-prod-deployer \
  --resource-group rg-platform --query clientId -o tsv)

PRINCIPAL_ID=$(az identity show --name github-prod-deployer \
  --resource-group rg-platform --query principalId -o tsv)

az role assignment create \
  --assignee-object-id "$PRINCIPAL_ID" \
  --assignee-principal-type ServicePrincipal \
  --role "<NARROW-CUSTOM-ROLE>" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/rg-production"
```

### Federated Identity Credential

```json
{
  "name": "github-production",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:my-org/my-app:environment:production",
  "audiences": ["api://AzureADTokenExchange"]
}
```

```bash
az identity federated-credential create \
  --name github-production \
  --identity-name github-prod-deployer \
  --resource-group rg-platform \
  --parameters federated-credential.json
```

The issuer, audience, and subject must exactly match the issued GitHub token. Allow propagation time and test in a non-production subscription.

### GitHub Workflow

```yaml
permissions:
  id-token: write
  contents: read

- uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

Use protected environments and separate identities for dev, test, and production.

### Terraform and AKS Controls

- Use remote encrypted state with locking and restricted access.
- Separate Terraform identity, application identity, and AKS pod identity.
- For AKS pods, configure a separate federated identity credential and projected service-account token.
- Enforce namespace-scoped Kubernetes RBAC independently of Azure RBAC.
- Restrict role-definition creation, role-assignment creation, and managed-identity operator permissions.

## Operational Controls, Risks & Guardrails

- Never grant subscription-wide `Owner` or `Contributor` by convenience.
- Block obsolete client secrets and certificates once federation is implemented.
- Use GitHub Environment protection for production and required approvals.
- Alert on federated credential changes, role assignments, sign-ins, and privileged actions.
- Investigate propagation, token claims, tenant, and network failures separately.
- Preserve Microsoft Entra sign-in/audit logs, Azure Activity Logs, AKS audit logs, and Terraform evidence.

## Evidence & Audit Checklist

- Managed identity/app registration inventory and owner.
- Federated credential issuer, audience, subject, and audience evidence.
- Azure RBAC role-definition and role-assignment export.
- Subscription/resource group/resource scope evidence.
- GitHub workflow, repository, environment, branch, and approval evidence.
- Entra sign-in and audit logs.
- Azure Activity and AKS audit logs where applicable.
- Terraform plan/apply, backend state, and state-access evidence.
- Secret/certificate inventory and removal confirmation.

## Scenario-Based Questions and Engineering Resolutions

### 1. Entra reports invalid federated credential
**Question:** The workflow receives a federated token exchange error. What is checked first?  
**Answer:** Compare issuer, audience, and exact subject claim with the federated credential. Inspect the GitHub token claim set and Entra sign-in/audit evidence. Test the exact repository and environment subject; do not use a wildcard if not approved.

### 2. Authentication succeeds but RBAC returns 403
**Question:** The identity is authenticated but cannot deploy resources. What is the resolution?  
**Answer:** Authentication and authorization are separate. Verify principal/object ID, role assignment scope, role definition actions, deny assignments, policy, and resource locks. Add only required permissions at the narrowest scope.

### 3. Forked pull request gets production privileges
**Question:** A pull-request workflow can access production. How is it prevented?  
**Answer:** Remove production credentials from PR workflows and use environment or branch subject restrictions. Require protected environment approval and separate untrusted build identities from deployment identities.

### 4. Federated credential trusts any repository
**Question:** The subject is `repo:my-org/*:*`. What is the risk?  
**Answer:** Authorized workflows in every matching repository may use the identity. Replace it with exact repository and context subjects and assess existing Entra sign-in/audit logs for unexpected principals.

### 5. An app registration and managed identity share one credential
**Question:** Which identity should be used for GitHub deployment?  
**Answer:** Use a dedicated managed identity or app registration per workload/environment. App registrations are appropriate for application architecture, but do not reuse a human or pod identity. Managed identities avoid stored credentials.

### 6. A production role assignment is recreated by Terraform drift
**Question:** A role assignment differs from source control. What action applies?  
**Answer:** Export Azure Activity and role-assignment data, determine if the change is authorized, reconcile state or import the resource, and restrict console/API changes for protected subscriptions.

### 7. The workflow receives `AADSTS70021`
**Question:** The federated identity credential cannot be found. How is it resolved?  
**Answer:** Verify the identity has the credential, issuer and subject match exactly, audience is correct, and changes have propagated. Confirm tenant and client ID refer to the intended identity.

### 8. `Contributor` is granted at subscription scope
**Question:** Why is this unacceptable?  
**Answer:** Contributor permits broad management operations and may enable downstream privilege escalation. Replace it with resource/resource-group scoped roles or custom roles containing only required actions.

### 9. An old client secret remains
**Question:** Federation is enabled but GitHub Secrets still store a client secret. What should happen?  
**Answer:** Remove workflow references, revoke the secret, verify sign-in logs, add secret scanning, and use the managed identity/app registration federation exclusively.

### 10. Terraform state exposes identity IDs or sensitive values
**Question:** How should state be secured and remediated?  
**Answer:** Restrict state access, enable encryption and locking, rotate actual secrets, review state logs, and remove sensitive values where possible. Treat state as controlled infrastructure evidence.

### 11. PIM bypasses approval
**Question:** A privileged role is permanently active despite PIM. What is the fix?  
**Answer:** Remove persistent assignments, configure activation duration, MFA, justification and approval, and alert on emergency activation. Use time-bound roles and periodic access review.

### 12. AKS pod identity fails despite CI/CD success
**Question:** GitHub authenticates but pods cannot access Blob Storage. How is it separated?  
**Answer:** Verify pod federation, projected service-account token, client ID, tenant, label/annotation, managed identity, container RBAC, and Blob Data role. Pipeline and pod identities are independent.

### 13. Azure role assignment propagates slowly
**Question:** The assignment was created but is temporarily denied. What should be done?  
**Answer:** Wait a bounded propagation period and retry with the correct principal and scope. If persistent, inspect role assignment, tenant, cache, networking, and resource data-plane authorization.

### 14. Blob Data Reader is absent
**Question:** The identity can manage Storage Account settings but cannot read blobs. What is wrong?  
**Answer:** Azure management-plane Contributor does not grant Blob data-plane access. Assign the appropriate Blob Data role at account/container scope and verify Entra authentication mode.

### 15. A role assignment is deleted by the wrong pipeline
**Question:** Which controls prevent this?  
**Answer:** Separate role-administration permissions, use protected environments, require approval, pin policy, review Terraform plans, and restrict service principal rights to create role assignments.

### 16. Cross-subscription access fails
**Question:** A managed identity in one subscription needs data in another. What must be verified?  
**Answer:** Verify tenant, principal object ID, target subscription scope, Blob/Key Vault RBAC, private network, policy, and resource firewall. Cross-subscription in one tenant may be allowed with RBAC; cross-tenant requires explicit tenant trust.

### 17. Federated credential subject matches the wrong environment
**Question:** GitHub environment is `production`, but subject uses `main`. What is the risk?  
**Answer:** A workflow may match an unintended branch or environment. Standardize environment naming, verify actual token claims, and use exact subjects. Protect environments with approvals.

### 18. A third-party action receives a token
**Question:** A privileged workflow uses an unreviewed marketplace action. What guardrail applies?  
**Answer:** Pin actions to immutable releases, restrict marketplace actions, reduce permissions, isolate credentials, and require security review. Third-party code executes with job permissions.

### 19. Azure sign-in logs show a blocked or unknown tenant
**Question:** Token exchange fails before Azure RBAC. What is investigated?  
**Answer:** Confirm tenant ID, issuer, Conditional Access, service principal status, managed identity state, federated credential, and audience. Distinguish exchange failure from authorization failure.

### 20. A role assignment appears on an unexpected resource
**Question:** How is scope and blast radius determined?  
**Answer:** Export all assignments for the principal and resource hierarchy, correlate with Terraform and Activity Logs, remove unauthorized assignments, and alert on scope expansion.

### 21. Cloud audit logging is disabled
**Question:** An Entra/Azure audit gap exists. What is required?  
**Answer:** Enable diagnostic settings and Entra audit/sign-in logs, route records to protected storage/SIEM, configure alerts, and document the evidence retention gap.

### 22. A break-glass identity is used for a normal deployment
**Question:** How should the event be handled?  
**Answer:** Correlate incident approval, Entra PIM, Azure Activity and GitHub logs; stop the session or role assignment if unauthorized; assess impact; and remediate the normal pipeline identity.

### 23. A runner is compromised
**Question:** What containment steps apply?  
**Answer:** Isolate runners, revoke active sessions/assignments if required, inspect logs and artifacts, rotate federated credential trust, and move to ephemeral protected runners.

### 24. A deny assignment blocks deployment
**Question:** RBAC role exists but access is denied. How is this diagnosed?  
**Answer:** Review Azure deny assignments, policy, locks, resource firewall, private connectivity, and Conditional Access. Explicit deny assignments or policy may override RBAC.

### 25. An unused federated credential remains active
**Question:** How should it be discovered and removed?  
**Answer:** Inventory credentials by identity and repository subject, confirm usage in Entra sign-in logs, obtain owner approval, delete obsolete credentials, and review dependent workflows.

## Assumptions and Missing Details

The source does not specify subscription IDs, tenants, GitHub organization, environment subjects, approved role definitions, log retention, or regulatory framework. These must be supplied by the control owner.
