# Kubernetes Pod Runtime Identity (AWS, GCP & Azure)

**Pod Runtime Identity** enables Kubernetes Pods to authenticate directly with cloud object storage and APIs (**AWS S3**, **Google Cloud Storage**, and **Azure Blob Storage**) using short-lived, workload-scoped credentials bound to a Kubernetes `ServiceAccount`—eliminating static secrets, long-lived API keys, and overly permissive node-level IAM roles.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Why Terraform + Kubernetes Manifests?](#why-terraform--kubernetes-manifests)
3. [Cloud Provider Comparison](#cloud-provider-comparison)
4. [Prerequisites](#prerequisites)
5. [AWS EKS Setup (EKS Pod Identity & IRSA)](#aws-eks-setup-eks-pod-identity--irsa)
6. [GCP GKE Setup (Workload Identity)](#gcp-gke-setup-workload-identity)
7. [Azure AKS Setup (Entra Workload ID)](#azure-aks-setup-entra-workload-id)
8. [Verification & Testing](#verification--testing)
9. [Security Best Practices](#security-best-practices)

---

## Architecture Overview

```text
+-------------------------------------------------------------------+
|                        Kubernetes Cluster                         |
|                                                                   |
|  +---------------------------+     +---------------------------+  |
|  |           Pod             |     |      ServiceAccount       |  |
|  |  serviceAccountName: sa   |---->|  (Projected OIDC Token /  |  |
|  +---------------------------+     |   Pod Identity Agent)     |  |
|                |                   +---------------------------+  |
+----------------|---------------------------------|----------------+
                 |                                 |
                 | 1. Request short-lived token    | 2. Exchange identity
                 v                                 v
+-------------------------------------------------------------------+
|                   Cloud IAM / Identity Provider                   |
|   AWS IAM (STS)  |  GCP IAM (Workload Identity)  |  Entra ID      |
+-------------------------------------------------------------------+
                 |
                 | 3. Issue temporary scoped credentials
                 v
+-------------------------------------------------------------------+
|                    Cloud Object Storage Bucket                    |
|     Amazon S3      |     Google Cloud Storage    |   Azure Blob   |
+-------------------------------------------------------------------+
```

### How It Works
1. **Workload Binding:** A `Pod` specifies a dedicated `serviceAccountName` instead of using the `default` ServiceAccount.
2. **Token Projection / Agent Injection:** Kubernetes projects a short-lived, audience-bound OIDC token into the Pod (or uses the EKS Pod Identity node agent).
3. **Federated Trust Exchange:** The cloud provider's STS / IAM endpoint validates the token against a pre-configured trust relationship (`namespace` + `ServiceAccount` name).
4. **Scoped Access:** Cloud SDKs inside the container automatically exchange the token for temporary credentials scoped strictly to the target storage bucket or container.

---

## Why Terraform + Kubernetes Manifests?

Cloud runtime identity spans two control planes that must stay synchronized:

| Layer | Tooling | Responsibilities |
| :--- | :--- | :--- |
| **Cloud Infrastructure & IAM** | **Terraform** | Provisions cloud roles/identities, bucket-level permissions, and OIDC/Pod Identity trust policies. |
| **Cluster Workload Spec** | **Kubernetes YAML / Helm** | Declares the `Namespace`, `ServiceAccount` annotations, and `Pod`/`Deployment` spec. |

**Key Benefits of Using Terraform:**
- **Zero Credential Drift:** Role ARNs, Service Account emails, and Client IDs are managed declaratively and exported as outputs.
- **Least Privilege by Default:** Each microservice gets an isolated IAM identity scoped to a single bucket or prefix.
- **Auditability:** All IAM trust relationships and bucket policies are version-controlled and peer-reviewed.

---

## Cloud Provider Comparison

| Capability | AWS EKS (Pod Identity) | AWS EKS (IRSA) | GCP GKE (Workload Identity) | Azure AKS (Workload Identity) |
| :--- | :--- | :--- | :--- | :--- |
| **Cloud Identity** | IAM Role | IAM Role | Google Service Account (GSA) | User-Assigned Managed Identity |
| **Trust Mechanism** | EKS Pod Identity Association | IAM OIDC Identity Provider | IAM Policy Binding (`workloadIdentityUser`) | Federated Identity Credential |
| **SA Annotation** | None required | `eks.amazonaws.com/role-arn` | `iam.gke.io/gcp-service-account` | `azure.workload.identity/client-id` |
| **Pod Label** | None required | None required | None required | `azure.workload.identity/use: "true"` |
| **Storage Role** | Custom S3 IAM Policy | Custom S3 IAM Policy | `roles/storage.objectUser` | `Storage Blob Data Contributor` |

---

## Prerequisites

- **Terraform** `>= 1.5.0`
- **kubectl** configured for your target cluster
- Cluster features enabled:
  - **AWS EKS:** `eks-pod-identity-agent` add-on installed (or OIDC provider enabled for IRSA)
  - **GCP GKE:** Workload Identity Federation for GKE enabled on cluster and node pools
  - **Azure AKS:** `--enable-oidc-issuer` and `--enable-workload-identity` enabled on the cluster

---

## AWS EKS Setup (EKS Pod Identity & IRSA)

### 1. Terraform (`aws-s3-pod-identity.tf`)

```hcl
variable "cluster_name" { type = string }
variable "bucket_name" { type = string }
variable "namespace" { default = "app-ns" }
variable "service_account_name" { default = "s3-reader-sa" }

data "aws_iam_policy_document" "pod_identity_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }
    actions = ["sts:AssumeRole", "sts:TagSession"]
  }
}

resource "aws_iam_role" "s3_pod_role" {
  name               = "${var.cluster_name}-${var.namespace}-${var.service_account_name}"
  assume_role_policy = data.aws_iam_policy_document.pod_identity_trust.json
}

data "aws_iam_policy_document" "s3_access" {
  statement {
    effect    = "Allow"
    actions   = ["s3:ListBucket", "s3:GetBucketLocation"]
    resources = ["arn:aws:s3:::${var.bucket_name}"]
  }
  statement {
    effect    = "Allow"
    actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = ["arn:aws:s3:::${var.bucket_name}/*"]
  }
}

resource "aws_iam_role_policy" "s3_access" {
  name   = "s3-bucket-access"
  role   = aws_iam_role.s3_pod_role.id
  policy = data.aws_iam_policy_document.s3_access.json
}

resource "aws_eks_pod_identity_association" "s3_assoc" {
  cluster_name    = var.cluster_name
  namespace       = var.namespace
  service_account = var.service_account_name
  role_arn        = aws_iam_role.s3_pod_role.arn
}
```

### 2. Kubernetes Manifest (`aws-pod.yaml`)

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-ns
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: app-ns
---
apiVersion: v1
kind: Pod
metadata:
  name: s3-test-pod
  namespace: app-ns
spec:
  serviceAccountName: s3-reader-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      command: ["sleep", "3600"]
```

---

## GCP GKE Setup (Workload Identity)

### 1. Terraform (`gcp-gcs-wi.tf`)

```hcl
variable "project_id" { type = string }
variable "bucket_name" { type = string }
variable "namespace" { default = "app-ns" }
variable "ksa_name" { default = "gcs-reader-sa" }

resource "google_service_account" "gcs_gsa" {
  project      = var.project_id
  account_id   = "gcs-pod-identity-sa"
  display_name = "GKE Workload Identity for GCS"
}

resource "google_storage_bucket_iam_member" "bucket_access" {
  bucket = var.bucket_name
  role   = "roles/storage.objectUser"
  member = "serviceAccount:${google_service_account.gcs_gsa.email}"
}

resource "google_service_account_iam_member" "wi_binding" {
  service_account_id = google_service_account.gcs_gsa.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[${var.namespace}/${var.ksa_name}]"
}
```

### 2. Kubernetes Manifest (`gcp-pod.yaml`)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gcs-reader-sa
  namespace: app-ns
  annotations:
    iam.gke.io/gcp-service-account: gcs-pod-identity-sa@<PROJECT_ID>.iam.gserviceaccount.com
---
apiVersion: v1
kind: Pod
metadata:
  name: gcs-test-pod
  namespace: app-ns
spec:
  serviceAccountName: gcs-reader-sa
  containers:
    - name: gcloud
      image: google/cloud-sdk:slim
      command: ["sleep", "3600"]
```

---

## Azure AKS Setup (Entra Workload ID)

### 1. Terraform (`azure-blob-wi.tf`)

```hcl
variable "resource_group_name" { type = string }
variable "location" { type = string }
variable "aks_oidc_issuer_url" { type = string }
variable "storage_account_id" { type = string }
variable "namespace" { default = "app-ns" }
variable "ksa_name" { default = "blob-reader-sa" }

resource "azurerm_user_assigned_identity" "blob_identity" {
  name                = "aks-blob-pod-identity"
  resource_group_name = var.resource_group_name
  location            = var.location
}

resource "azurerm_role_assignment" "blob_contributor" {
  scope                = var.storage_account_id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_user_assigned_identity.blob_identity.principal_id
}

resource "azurerm_federated_identity_credential" "aks_fed" {
  name                = "aks-blob-federation"
  resource_group_name = var.resource_group_name
  parent_id           = azurerm_user_assigned_identity.blob_identity.id
  audience            = ["api://AzureADTokenExchange"]
  issuer              = var.aks_oidc_issuer_url
  subject             = "system:serviceaccount:${var.namespace}:${var.ksa_name}"
}
```

### 2. Kubernetes Manifest (`azure-pod.yaml`)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: blob-reader-sa
  namespace: app-ns
  annotations:
    azure.workload.identity/client-id: "<MANAGED_IDENTITY_CLIENT_ID>"
---
apiVersion: v1
kind: Pod
metadata:
  name: blob-test-pod
  namespace: app-ns
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: blob-reader-sa
  containers:
    - name: azure-cli
      image: mcr.microsoft.com/azure-cli:latest
      command: ["sleep", "3600"]
```

---

## Verification & Testing

Run the following commands inside each test Pod to confirm that workload identity is active and bucket operations succeed without static credentials:

### AWS EKS
```bash
kubectl -n app-ns exec -it s3-test-pod -- aws sts get-caller-identity
kubectl -n app-ns exec -it s3-test-pod -- aws s3 ls s3://<BUCKET_NAME>
```

### GCP GKE
```bash
kubectl -n app-ns exec -it gcs-test-pod -- gcloud auth list
kubectl -n app-ns exec -it gcs-test-pod -- gcloud storage ls gs://<BUCKET_NAME>
```

### Azure AKS
```bash
kubectl -n app-ns exec -it blob-test-pod -- az login --federated-token "$(cat $AZURE_FEDERATED_TOKEN_FILE)" --service-principal -u $AZURE_CLIENT_ID -t $AZURE_TENANT_ID
kubectl -n app-ns exec -it blob-test-pod -- az storage blob list --account-name <STORAGE_ACCOUNT> --container-name <CONTAINER_NAME> --auth-mode login
```

---

## Security Best Practices

- **One Identity Per Microservice:** Avoid sharing a single `ServiceAccount` or IAM Role across unrelated applications.
- **Prefix / Container Scoping:** Restrict IAM policies to specific S3/GCS key prefixes or individual Azure Blob containers rather than account-wide wildcards.
- **Disable Default Token Auto-Mounting:** Set `automountServiceAccountToken: false` on Pods that do not require Kubernetes API or cloud IAM access.
- **Keep Cloud SDKs Updated:** Ensure application container images use current AWS, GCP, or Azure SDK versions that natively support OIDC/Pod Identity credential providers.