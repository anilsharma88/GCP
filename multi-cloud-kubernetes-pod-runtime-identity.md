# Multi-Cloud Kubernetes Pod Runtime Identity

**Document status:** Compliance portal draft  
**Original filename:** `podruntimeidenity.md`  
**Filename rationale:** Corrected the misspelling and expanded the title to state the platform scope and Kubernetes runtime identity domain.

## Purpose & Scope

This standard covers Kubernetes pod identities for AWS EKS, GCP GKE, and Azure AKS. It includes EKS Pod Identity/IRSA, GKE Workload Identity, and AKS Workload Identity for short-lived cloud access by containers.

Out of scope: CI/CD federated deployment identity, node-level identities, human cluster access, Kubernetes control-plane hardening, and organization-specific workload labels.

## Control Objective & Compliance Alignment

- Bind each pod workload to a dedicated namespace/service-account identity and cloud identity.
- Eliminate static keys and broad node IAM roles.
- Apply least privilege at bucket/container/prefix, cloud role, and Kubernetes RBAC scopes.
- Separate workloads, environments, and teams.
- Preserve cluster audit, cloud audit, federation, IAM, and pod activity evidence.

## Technical Architecture & Implementation Details

### Cross-Cloud Mapping

| Platform | Kubernetes identity | Cloud identity | Trust mechanism | Authorization |
|---|---|---|---|---|
| EKS | Kubernetes ServiceAccount | IAM role | Pod Identity association or IRSA | IAM role policy |
| GKE | Kubernetes ServiceAccount | Google service account | IAM Workload Identity User binding | GCP IAM |
| AKS | Kubernetes ServiceAccount | Managed identity or app | Entra federated identity credential | Azure RBAC |

### AWS EKS Pod Identity

```hcl
resource "aws_iam_role" "pod_role" {
  name = "app-prod-s3-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "pods.eks.amazonaws.com" }
      Action = ["sts:AssumeRole", "sts:TagSession"]
    }]
  })
}

resource "aws_eks_pod_identity_association" "app" {
  cluster_name    = var.cluster_name
  namespace       = "app-prod"
  service_account = "app-sa"
  role_arn        = aws_iam_role.pod_role.arn
}
```

Use a dedicated namespace and service account. For IRSA, bind the IAM trust subject to `system:serviceaccount:<namespace>:<service-account>` and validate the EKS issuer.

### GKE Workload Identity

```hcl
resource "google_service_account_iam_member" "gke_identity" {
  service_account_id = google_service_account.pod_sa.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[app-prod/app-sa]"
}
```

Annotate the Kubernetes service account:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: app-prod
  annotations:
    iam.gke.io/gcp-service-account: app-prod@PROJECT_ID.iam.gserviceaccount.com
```

Enable Workload Identity on the cluster/node pool and avoid node default service accounts.

### AKS Workload Identity

1. Enable AKS OIDC issuer and Workload Identity.
2. Create a user-assigned managed identity.
3. Create a federated credential subject `system:serviceaccount:<namespace>:<service-account>`.
4. Assign Blob or Key Vault RBAC at the narrow scope.
5. Label the pod and annotate the service account with Azure client ID/tenant ID.

```yaml
metadata:
  labels:
    azure.workload.identity/use: "true"
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
```

## Operational Controls, Risks & Guardrails

- Disable automatic Kubernetes service-account token mounting where cloud identity is unnecessary.
- Prevent pods from sharing the default namespace service account.
- Audit pod security, host PID/network, metadata access, and privileged containers.
- Restrict cloud token exchange to exact namespace and service-account claims.
- Use Kubernetes RBAC for API authorization and cloud IAM/RBAC for cloud-resource authorization.
- Alert on cloud role/trust changes, KSA annotations, federated credentials, and broad bindings.

## Evidence & Audit Checklist

- Cluster feature configuration and OIDC issuer evidence.
- Namespace and Kubernetes service-account inventory.
- IAM role/IRSA association or Pod Identity association export.
- GKE IAM Workload Identity binding export.
- Entra federated credential and Azure RBAC export.
- Pod/deployment spec and image provenance.
- Kubernetes audit logs and cloud/provider audit logs.
- Storage IAM/RBAC policy and network evidence.
- Incident, exception, access-review, and Terraform evidence.

## Scenario-Based Questions and Engineering Resolutions

### 1. AWS pod assumes a role intended for another service
**Question:** Pod identity is too broad. What is wrong and how is it fixed?  
**Answer:** Use a dedicated EKS Pod Identity association or IRSA trust subject. Scope the namespace/service-account claim exactly and bind only a workload-specific IAM role. Review CloudTrail role sessions.

### 2. IRSA trust omits the namespace
**Question:** A role trust allows `system:serviceaccount:*:*`. What is the remediation?  
**Answer:** Replace wildcard trust with `system:serviceaccount:<namespace>:<service-account>` and remove broad subject patterns. Review prior role assumptions and update the policy after testing.

### 3. GKE pod uses the node service account
**Question:** Workload identity is not bound. How is it corrected?  
**Answer:** Enable GKE Workload Identity, annotate the KSA, bind the GSA with `roles/iam.workloadIdentityUser`, and remove broad node account permissions. Verify pod GSA resolution with metadata endpoints.

### 4. AKS pod receives `client credential not found`
**Question:** Federation fails. What is checked?  
**Answer:** Verify OIDC issuer, federated subject, client ID, tenant ID, pod label, service-account annotation, and projected token. Confirm the service-account name and namespace exactly match the federation subject.

### 5. Azure pod can manage storage account but cannot read blobs
**Question:** What is the likely issue?  
**Answer:** Management-plane and data-plane permissions are separate. Assign the Blob Data role at account/container scope and validate Entra authentication mode and storage firewall/private endpoint.

### 6. Pod has an AWS IAM role with `s3:*`
**Question:** How is access reduced?  
**Answer:** Replace wildcard actions with required list/read/write/delete actions; scope bucket and prefix ARNs separately; remove delete or list where unnecessary; and test with IAM policy simulation/CloudTrail.

### 7. GKE service account has project Editor
**Question:** How is it contained?  
**Answer:** Remove project Editor, grant a workload-specific GCP role or custom role on the target bucket/resource, and investigate activity from the GSA. Use Organization Policy and log review.

### 8. AKS managed identity has Contributor
**Question:** Why is this insufficient for data and overprivileged for management?  
**Answer:** Contributor may grant management operations but not data-plane access and is too broad for pod storage use. Replace it with a custom role or Blob Data role at the exact container scope.

### 9. Pod service account has no network restrictions
**Question:** Cloud IAM is least-privilege but metadata or storage access is compromised. What else applies?  
**Answer:** Restrict network, egress, metadata access, pod security, container privileges, and secrets. Use service mesh and policy controls where appropriate. IAM limits authorization but not lateral movement.

### 10. Kubernetes role binding grants cluster-admin
**Question:** How is it remediated?  
**Answer:** Remove cluster-admin, create namespace-scoped roles, validate pod/deployment service accounts, and restrict cluster-role binding administration. Audit Kubernetes audit logs for prior use.

### 11. A pod copies a cloud token from the mounted projected token
**Question:** What is the risk and response?  
**Answer:** Treat projected/federated tokens as short-lived credentials and prevent exfiltration at runtime. Revoke cloud sessions if abuse is detected, isolate pods, reduce privileges, and rotate federation conditions if needed.

### 12. Pod trust is shared across environments
**Question:** Dev and production pods share the identity. How is this separated?  
**Answer:** Use environment-specific namespaces, KSAs, cloud identities, trust subjects, IAM/RBAC policies, and cluster or account boundaries. Never permit dev pods to receive production trust claims.

### 13. Terraform recreates a pod role but forgets the binding
**Question:** Deployment fails. What is validated?  
**Answer:** Synchronize Terraform and Kubernetes manifests. Verify role association/binding and KSA annotation/subject, then apply both atomically through protected pipelines. Reconcile state drift.

### 14. State lock is disabled
**Question:** Concurrent applies alter identity associations. What guardrail applies?  
**Answer:** Enable remote state locking and versioning, protected apply, policy validation, and single-writer access. Investigate resulting identity differences and reconcile them.

### 15. Kubernetes audit logs show service-account impersonation
**Question:** How is it investigated?  
**Answer:** Correlate pod name, namespace, user, verb, resource, source IP, KSA and node, then match cloud audit events. Revoke unauthorized roles and rotate compromised workload credentials.

### 16. Cloud audit logs show a pod role used from an unexpected cluster
**Question:** What is the likely trust issue?  
**Answer:** Cross-cluster trust may be too broad or the OIDC issuer/provider is shared. Pin issuer/provider and cluster identity claims and use dedicated identities for clusters.

### 17. AWS IRSA trust has the wrong audience
**Question:** Pods cannot assume the role. How is it corrected?  
**Answer:** Configure the OIDC provider and IAM trust conditions with the EKS issuer and `aud: sts.amazonaws.com` as appropriate. Verify projected service-account token and provider thumbprint.

### 18. GKE Workload Identity is disabled on a node pool
**Question:** Some pods cannot authenticate. What is checked?  
**Answer:** Verify cluster and node-pool Workload Identity settings, metadata server, KSA annotation, GSA binding, and pod scheduling. Ensure no conflicting legacy scopes.

### 19. AKS pod uses a system-assigned identity unexpectedly
**Question:** How should the identity be controlled?  
**Answer:** Use a documented user-assigned identity per workload if lifecycle independence is required. Remove unintended system identity assignments and verify pod labels/annotations refer to the intended client ID.

### 20. Pod image contains static credentials
**Question:** What is the remediation?  
**Answer:** Remove credentials from the image and registry, revoke/rotate them, rebuild/deploy a clean image, scan image history, and convert the workload to runtime identity.

### 21. Pod is able to assume another pod identity
**Question:** What privilege escalation path is assessed?  
**Answer:** Review Kubernetes service-account permissions, AWS IAM/Pod Identity association, GCP service-account impersonation, Azure role assignment/federation, API-server RBAC, and secrets access. Restrict the exact identity trust and pod privileges.

### 22. A pod runs with hostPath or hostNetwork
**Question:** Does least-privilege IAM mitigate this?  
**Answer:** No. Add pod-security admission, prohibit privileged and host namespace access, scan manifests, and restrict workloads that can reach cloud metadata or mounted service-account tokens.

### 23. Cloud role/session logs do not link to pod identity
**Question:** What logging enhancements are required?  
**Answer:** Tag or correlate AWS session tags, GCP token subject/SA principals, Azure Entra claims, pod name/namespace, cluster name, and deployment SHA. Send logs to immutable storage/SIEM.

### 24. A namespace is deleted and recreated
**Question:** Existing federation suddenly changes. What must be checked?  
**Answer:** Recreate the KSA and exact trust binding/association/federated credential. Verify old credentials and role access are not stale or reused; do not scope trust by namespace name alone where higher isolation is required.

### 25. An auditor asks for proof of no static keys
**Question:** What evidence is sufficient?  
**Answer:** Export pod specs, KSA annotations, cloud role trust/bindings, secret inventory, image/registry scans, CloudTrail/Cloud Audit/Azure logs, and runtime verification showing SDK credentials are short-lived.

## Assumptions and Missing Details

The source does not specify cluster names, OIDC issuers, cloud account/project/subscription IDs, namespace names, storage resources, retention periods, or approved pod-security baseline. These must be supplied by control owners.
