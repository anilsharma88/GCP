# Google Cloud Workload Identity Federation (WIF)

This guide configures **GitHub Actions to authenticate to Google Cloud
without a long-lived service-account JSON key**.

It uses:

-   A Google Cloud Workload Identity Pool (WIP)
-   An OIDC provider that trusts GitHub Actions
-   A Google Cloud service account
-   IAM impersonation restricted to one GitHub repository
-   Terraform for the Google Cloud identity configuration

> **Terminology:** A Workload Identity Pool is the pool resource.
> Workload Identity Federation (WIF) is the overall authentication
> mechanism. The pool contains one or more identity providers.

## 1. Authentication flow

1.  A GitHub Actions job requests an OIDC token from GitHub.
2.  GitHub issues a short-lived token identifying the repository and
    workflow context.
3.  Google Cloud validates the token using the configured OIDC provider.
4.  The provider maps GitHub claims to Google attributes and checks the
    attribute condition.
5.  The matching GitHub identity is allowed to impersonate a Google
    service account.
6.  The service account's IAM permissions determine which Google Cloud
    resources the job can access.

No service-account private key is stored in GitHub.

## 2. Prerequisites

-   A Google Cloud project with billing enabled, where required by the
    APIs/services you use.
-   Permission to manage Workload Identity Pools, service accounts, and
    IAM policies.
-   Terraform installed.
-   A GitHub repository in an organization or personal account.
-   A GitHub Actions workflow.
-   The numeric Google Cloud project number (not the project ID).

Enable the required APIs:

``` bash
gcloud services enable \
  iam.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  --project=YOUR_PROJECT_ID
```

For GKE deployments, enable the relevant GKE API as well:

``` bash
gcloud services enable container.googleapis.com \
  --project=YOUR_PROJECT_ID
```

## 3. Terraform configuration

Suggested directory:

``` text
gcp-wif/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars.example
```

### `main.tf`

``` hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

resource "google_service_account" "github_deployer" {
  account_id   = "github-deployer"
  display_name = "GitHub Actions deployer"
  description  = "Service account impersonated by the approved GitHub repository"
}

resource "google_iam_workload_identity_pool" "github" {
  workload_identity_pool_id = "github-pool"
  display_name               = "GitHub Actions Pool"
  description                = "Federation pool for GitHub Actions"
}

resource "google_iam_workload_identity_pool_provider" "github" {
  workload_identity_pool_id          = google_iam_workload_identity_pool.github.workload_identity_pool_id
  workload_identity_pool_provider_id = "github-oidc"
  display_name                       = "GitHub Actions OIDC"

  attribute_mapping = {
    "google.subject"       = "assertion.sub"
    "attribute.repository" = "assertion.repository"
    "attribute.ref"        = "assertion.ref"
  }

  # Restrict federation to the exact repository.
  # For production, consider also restricting the branch, tag,
  # or GitHub Environment using claims appropriate to your workflow.
  attribute_condition = "assertion.repository == '${var.github_repository}'"

  oidc {
    issuer_uri = "https://token.actions.githubusercontent.com"
  }
}

resource "google_service_account_iam_member" "github_can_impersonate" {
  service_account_id = google_service_account.github_deployer.name
  role               = "roles/iam.workloadIdentityUser"

  member = "principalSet://iam.googleapis.com/${google_iam_workload_identity_pool.github.name}/attribute.repository/${var.github_repository}"
}

# Example permission for uploading objects to one bucket.
# Remove this if your pipeline does not need Cloud Storage access.
resource "google_storage_bucket_iam_member" "github_bucket_access" {
  bucket = var.bucket_name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.github_deployer.email}"
}
```

If you are not granting Cloud Storage access, remove the
`google_storage_bucket_iam_member` resource. Grant only the roles your
deployment actually requires.

### `variables.tf`

``` hcl
variable "project_id" {
  description = "Google Cloud project ID"
  type        = string
}

variable "region" {
  description = "Default Google Cloud region"
  type        = string
  default     = "asia-south1"
}

variable "github_repository" {
  description = "GitHub repository in owner/repository format"
  type        = string
}

variable "bucket_name" {
  description = "Existing Cloud Storage bucket used by the example"
  type        = string
}
```

### `outputs.tf`

``` hcl
output "project_number" {
  value = data.google_project.current.number
}

data "google_project" "current" {
  project_id = var.project_id
}

output "workload_identity_provider" {
  value = "projects/${data.google_project.current.number}/locations/global/workloadIdentityPools/${google_iam_workload_identity_pool.github.workload_identity_pool_id}/providers/${google_iam_workload_identity_pool_provider.github.workload_identity_pool_provider_id}"
}

output "service_account_email" {
  value = google_service_account.github_deployer.email
}
```

### `terraform.tfvars.example`

``` hcl
project_id        = "my-gcp-project"
region            = "asia-south1"
github_repository = "my-org/my-repo"
bucket_name       = "my-existing-deployment-bucket"
```

The bucket in this example must already exist. Alternatively, manage its
creation in Terraform.

## 4. Deploy the Terraform configuration

``` bash
terraform init
terraform fmt -check
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
```

Get the provider resource name and service-account email:

``` bash
terraform output -raw workload_identity_provider
terraform output -raw service_account_email
```

Save these two values for the GitHub Actions workflow.

## 5. GitHub Actions workflow

Create `.github/workflows/deploy.yml` in the repository:

``` yaml
name: Authenticate to Google Cloud

on:
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  authenticate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

      - name: Set up gcloud
        uses: google-github-actions/setup-gcloud@v2

      - name: Verify identity
        run: |
          gcloud auth list
          gcloud config list project
```

In GitHub, add these **repository Actions variables**:

  ---------------------------------------------------------------------
  Variable                           Value
  ---------------------------------- ----------------------------------
  `GCP_WORKLOAD_IDENTITY_PROVIDER`   Terraform output
                                     `workload_identity_provider`

  `GCP_SERVICE_ACCOUNT`              Terraform output
                                     `service_account_email`
  ---------------------------------------------------------------------

The workflow needs `id-token: write` so GitHub can issue the OIDC token.
It does not need a service-account JSON key.

## 6. Security recommendations

-   Restrict the provider's `attribute_condition` to the exact
    repository.
-   For production, restrict the identity further to an approved branch,
    tag, or GitHub Environment.
-   Grant `roles/iam.workloadIdentityUser` only to the intended
    principal set.
-   Give the service account only the resource-level permissions
    required by the pipeline.
-   Avoid broad project-level roles such as Owner or Editor.
-   Protect production branches and GitHub Environments with required
    reviewers.
-   Pin third-party GitHub Actions to reviewed commit SHAs in
    production.
-   Do not print OIDC tokens or credentials in workflow logs.
-   Review IAM bindings periodically and remove unused identities.

## 7. Troubleshooting

### `Permission denied` while impersonating the service account

Check:

-   The pool and provider resource names are correct.
-   The service account IAM policy contains
    `roles/iam.workloadIdentityUser`.
-   The `principalSet` path uses the pool's full resource name and the
    mapped attribute.
-   The repository claim matches `owner/repository` exactly.
-   The GitHub workflow has `permissions: id-token: write`.

### `The attribute condition failed`

Check the provider's condition and the claims in the GitHub OIDC token.
Ensure the repository name and any branch or environment restrictions
match the workflow that is running.

### Authentication succeeds, but deployment is denied

WIF authentication and resource authorization are separate. Check the
roles granted to the impersonated service account on the target
resource, such as a specific bucket or GKE cluster.

### API errors

Confirm that the IAM, Security Token Service, and IAM Service Account
Credentials APIs are enabled in the correct project.

## 8. WIF vs GKE Workload Identity Federation

  -----------------------------------------------------------------------
  Topic                   External WIF            GKE Workload Identity
                                                  Federation
  ----------------------- ----------------------- -----------------------
  Typical identity        GitHub Actions, AWS,    Kubernetes workload
                          Azure, on-premises      identity

  Main use                Let an external         Let a GKE workload
                          workload access Google  access Google Cloud
                          Cloud                   

  Long-lived              Not required            Not required
  service-account key                             

  Identity source         External OIDC, SAML, or GKE Kubernetes
                          AWS identity            ServiceAccount
  -----------------------------------------------------------------------

For an application running in **AWS EKS** that needs Cloud Storage
access, configure an AWS identity provider in a Workload Identity Pool
and grant the AWS principal permission to impersonate a narrowly scoped
Google service account. The GitHub OIDC provider in this README is a
separate example and is not used for EKS authentication.

## 9. Official documentation

-   [Workload Identity Federation
    overview](https://cloud.google.com/iam/docs/workload-identity-federation)
-   [Configure GitHub Actions to use Workload Identity
    Federation](https://github.com/google-github-actions/auth)
-   [Terraform Google provider: Workload Identity
    Pool](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool)
-   [Terraform Google provider: Workload Identity Pool
    Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool_provider)
-   [Google Cloud IAM
    roles](https://cloud.google.com/iam/docs/understanding-roles)

------------------------------------------------------------------------

**Important:** This is a starting example. Review provider versions,
GitHub claims, IAM permissions, and organization security requirements
before using it in production.

## 10. Deploy an application to one GKE namespace using GitHub Actions

This section extends the WIF setup so a GitHub Actions workflow can
deploy an application to **one namespace only** in a GKE cluster.

### 10.1 Deployment architecture

``` text
GitHub repository
  |
  | GitHub Actions requests an OIDC token
  v
Google Workload Identity Pool
  |
  | Validates GitHub repository identity
  v
OIDC Provider
  |
  | Allows impersonation
  v
Google service account: github-deployer
  |
  | roles/container.clusterViewer (read cluster metadata)
  | Kubernetes RoleBinding (edit access in one namespace)
  v
GKE cluster
  |
  +-- namespace: app-team
        |
        +-- Deployment: sample-web
        +-- Service: sample-web
```

The important security separation is:

-   Google Cloud IAM lets the pipeline discover/connect to the GKE
    cluster.
-   Kubernetes RBAC limits what the authenticated service account can do
    inside the cluster.
-   A namespace-scoped `RoleBinding` grants permissions only in
    `app-team`, not across the whole cluster.

### 10.2 Example values

Replace these example values with your environment:

  ----------------------------------------------------------------------------------------------
  Setting                             Example
  ----------------------------------- ----------------------------------------------------------
  GCP project ID                      `my-gcp-project`

  GKE cluster                         `gke-app-cluster`

  GKE location                        `asia-south1`

  Namespace                           `app-team`

  GitHub repository                   `my-org/my-repo`

  Google service account              `github-deployer@my-gcp-project.iam.gserviceaccount.com`
  ----------------------------------------------------------------------------------------------

For a regional GKE cluster, `GKE_LOCATION` is the region. For a zonal
cluster, use its zone.

### 10.3 Grant the service account permission to retrieve cluster credentials

Add this resource to the Terraform configuration from the previous
section. It grants read access to GKE cluster metadata; it does not
grant namespace-wide Kubernetes permissions.

``` hcl
resource "google_project_iam_member" "github_gke_cluster_viewer" {
  project = var.project_id
  role    = "roles/container.clusterViewer"
  member  = "serviceAccount:${google_service_account.github_deployer.email}"
}
```

Do not grant `roles/container.admin` to the deployment service account
for this example. Use the narrowest Google Cloud IAM permissions that
meet your requirements.

Apply the Terraform change:

``` bash
terraform fmt
terraform validate
terraform plan
terraform apply
```

### 10.4 Create the namespace and namespace-scoped RBAC

A cluster administrator should create the namespace and RoleBinding
once. Do this through an administrator identity, not through the GitHub
deployment identity.

Save as `k8s/rbac/app-team-deployer.yaml`, and replace the
service-account email:

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-team
  labels:
    name: app-team
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-deployer-edit
  namespace: app-team
subjects:
  - kind: User
    name: github-deployer@my-gcp-project.iam.gserviceaccount.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

Apply it using an administrator context:

``` bash
kubectl apply -f k8s/rbac/app-team-deployer.yaml
```

The built-in `edit` ClusterRole is namespace-scoped when referenced by a
`RoleBinding`. It allows common application deployment operations in
that namespace, but does not grant permission to manage RoleBindings or
cluster-scoped resources.

For stricter production access, define a custom `Role` containing only
the required API groups and verbs instead of using the built-in `edit`
role.

> The namespace and RoleBinding are bootstrap resources. The GitHub
> deployment service account should not be responsible for creating its
> own permissions.

### 10.5 Add the Kubernetes application manifests

Example repository layout:

``` text
.
├── .github/
│   └── workflows/
│       └── deploy-gke.yml
└── k8s/
    └── app/
        ├── deployment.yaml
        └── service.yaml
```

`k8s/app/deployment.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-web
  namespace: app-team
  labels:
    app: sample-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-web
  template:
    metadata:
      labels:
        app: sample-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

`k8s/app/service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-web
  namespace: app-team
spec:
  type: ClusterIP
  selector:
    app: sample-web
  ports:
    - name: http
      port: 80
      targetPort: http
```

This example deploys NGINX as a simple test application. For a real
application, replace the image with your versioned image in Artifact
Registry or another trusted registry.

### 10.6 Configure GitHub repository variables

In the GitHub repository, open **Settings → Secrets and variables →
Actions → Variables** and create:

  -----------------------------------------------------------------------------------------------------------------------------------
  Variable                            Example value
  ----------------------------------- -----------------------------------------------------------------------------------------------
  `GCP_WORKLOAD_IDENTITY_PROVIDER`    `projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-oidc`

  `GCP_SERVICE_ACCOUNT`               `github-deployer@my-gcp-project.iam.gserviceaccount.com`

  `GCP_PROJECT_ID`                    `my-gcp-project`

  `GKE_CLUSTER`                       `gke-app-cluster`

  `GKE_LOCATION`                      `asia-south1`

  `GKE_NAMESPACE`                     `app-team`
  -----------------------------------------------------------------------------------------------------------------------------------

Use the exact provider and service-account values output by Terraform.
These are identifiers, not private keys, so they can be repository
variables.

### 10.7 GitHub Actions workflow

Create `.github/workflows/deploy-gke.yml`:

``` yaml
name: Deploy application to GKE

on:
  push:
    branches:
      - main
    paths:
      - "k8s/app/**"
      - ".github/workflows/deploy-gke.yml"
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    name: Deploy to app-team namespace
    runs-on: ubuntu-latest

    # Optional: configure this GitHub Environment and add required reviewers
    # before using it for production deployments.
    environment: production

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud using WIF
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ vars.GKE_CLUSTER }}
          location: ${{ vars.GKE_LOCATION }}
          project_id: ${{ vars.GCP_PROJECT_ID }}

      - name: Verify namespace permissions
        run: |
          kubectl auth can-i get deployments --namespace="${{ vars.GKE_NAMESPACE }}"
          kubectl auth can-i create deployments --namespace="${{ vars.GKE_NAMESPACE }}"
          kubectl auth can-i create services --namespace="${{ vars.GKE_NAMESPACE }}"
          kubectl auth can-i create namespaces

      - name: Deploy application
        run: |
          kubectl apply \
            --namespace="${{ vars.GKE_NAMESPACE }}" \
            -f k8s/app/

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/sample-web \
            --namespace="${{ vars.GKE_NAMESPACE }}" \
            --timeout=180s

      - name: Verify deployment
        run: |
          kubectl get deployments,pods,services \
            --namespace="${{ vars.GKE_NAMESPACE }}"
```

The `kubectl auth can-i create namespaces` check is expected to return
`no`. It demonstrates that the deployment identity cannot create
namespaces. If you want the workflow to fail only when required
namespaced permissions are missing, use explicit shell checks:

``` bash
kubectl auth can-i create deployments -n "$GKE_NAMESPACE" | grep -qx yes
kubectl auth can-i patch deployments -n "$GKE_NAMESPACE" | grep -qx yes
kubectl auth can-i create services -n "$GKE_NAMESPACE" | grep -qx yes
```

You can replace the verification step in the workflow with these checks.

### 10.8 Verify the deployment

After the GitHub Actions workflow succeeds, an administrator or
authorized namespace user can run:

``` bash
kubectl get pods -n app-team
kubectl get deployments -n app-team
kubectl get services -n app-team
kubectl rollout status deployment/sample-web -n app-team
```

Expected deployment state:

``` text
NAME         READY   UP-TO-DATE   AVAILABLE
sample-web   2/2     2            2
```

To verify the deployment identity is restricted to the intended
namespace, test access to another namespace:

``` bash
kubectl auth can-i get deployments -n kube-system
```

It should return `no`, assuming the service account has no other broader
IAM or Kubernetes RBAC grants.

### 10.9 Important networking note for private GKE clusters

The GitHub-hosted `ubuntu-latest` runner cannot directly reach a GKE
private control-plane endpoint unless the required network path is
available.

For a private GKE cluster, use a self-hosted GitHub Actions runner in a
network connected to the GKE VPC, or another approved private
connectivity design. Configure firewall rules, DNS, and control-plane
authorized networks as appropriate.

### 10.10 Production recommendations

-   Use a dedicated service account for each deployment environment or
    trust boundary.
-   Restrict the WIF provider to the exact repository and, where
    practical, the approved branch or GitHub Environment.
-   Keep the namespace RoleBinding separate from application deployment.
-   Prefer a custom namespace `Role` over the broad built-in `edit` role
    when feasible.
-   Use immutable image tags or image digests rather than mutable tags
    such as `latest`.
-   Add image scanning and deployment policy checks before applying
    manifests.
-   Use GitHub Environment approvals for production.
-   Pin GitHub Actions to reviewed commit SHAs in production.
-   Keep cluster administration and namespace RBAC changes in a
    separate, tightly controlled workflow.
## Read

from pathlib import Path
import pypandoc

readme_path = Path("/mnt/data/README.md")
if readme_path.exists():
    base = readme_path.read_text()
else:
    base = "# GCP Workload Identity Federation (WIF)\n"

interview_section = r"""

# GCP Workload Identity Federation (WIF) – Interview Preparation

This section provides interview questions, real-world problem statements, scenario-based answers, and a hands-on exercise for **Google Cloud Workload Identity Federation (WIF)**.

## 1. Terminology

- **WIP – Workload Identity Pool:** A Google Cloud IAM resource that organizes external identities.
- **WIF – Workload Identity Federation:** The mechanism that allows external workloads to authenticate to Google Cloud without service-account keys.
- **OIDC provider:** Validates identity tokens issued by an external identity provider, such as GitHub Actions.
- **Service account:** The Google Cloud identity whose permissions are used to access GCP resources.
- **Attribute mapping:** Maps claims from an external token to Google attributes.
- **Attribute condition:** Restricts which external identities are trusted.

## 2. Why do we use WIF?

### Real-world problem statement

An organization has 50 GitHub repositories. Each repository has a GitHub Actions pipeline that deploys applications to GKE.

Previously, the DevOps team created Google Cloud service-account JSON keys and stored them in GitHub Secrets.

This creates several security and operational risks:

| Problem | Risk |
|---|---|
| Long-lived JSON keys | A stolen key can be reused until revoked or expired |
| Shared service-account keys | Difficult to attribute usage to a specific workload |
| Key rotation | Operational overhead and risk of missed rotation |
| Developer access to secrets | Credentials can be exposed |
| Keys committed accidentally | Potential cloud compromise |
| Many repositories | Difficult credential management |

### WIF solution

Instead of distributing service-account keys:

1. GitHub Actions requests an OIDC token from GitHub.
2. Google Cloud validates the token using a configured Workload Identity Pool provider.
3. GCP checks the token claims and configured attribute conditions.
4. The approved federated identity is allowed to impersonate a Google service account.
5. The service account accesses GCP resources using its IAM permissions.

**Benefits:**

- No long-lived service-account JSON key.
- Short-lived federated credentials.
- Centralized identity federation.
- Repository, branch, and environment-level trust restrictions.
- Least-privilege IAM permissions.
- Better auditability and simpler credential lifecycle management.

## 3. Authentication and authorization flow

```text
GitHub repository
       |
       | GitHub Actions requests OIDC token
       v
GitHub OIDC issuer
       |
       | Signed identity token
       v
Google Workload Identity Pool
       |
       v
OIDC Provider
       |
       | Validate issuer, audience, signature and claims
       | Apply attribute mapping and attribute condition
       v
Federated principal
       |
       | roles/iam.workloadIdentityUser binding
       v
Google Cloud service account
       |
       | Google Cloud IAM permissions
       v
GKE / Cloud Storage / other GCP resources

For Kubernetes API operations:
Service account identity
       |
       v
Kubernetes RBAC RoleBinding
       |
       v
Only the authorized namespace
```

**Important distinction:** WIF authenticates the external workload. Google Cloud IAM authorizes access to Google Cloud APIs. Kubernetes RBAC authorizes operations against Kubernetes resources. Network connectivity is a separate requirement.

## 4. Basic interview questions and answers

### Q1. What is Workload Identity Federation in GCP?

Workload Identity Federation allows external workloads to access Google Cloud resources without maintaining service-account private keys. It uses an external identity provider, such as GitHub Actions, AWS, Azure, or another OIDC/SAML provider, to authenticate workloads.

### Q2. What is a Workload Identity Pool?

A Workload Identity Pool is a Google Cloud IAM resource that manages external identities. It contains one or more identity providers and is used to establish trust between external identities and Google Cloud.

### Q3. What is the difference between WIP and WIF?

| WIP | WIF |
|---|---|
| Workload Identity Pool | Workload Identity Federation |
| A resource in GCP IAM | The federation mechanism |
| Organizes external identities and providers | Establishes trust with external identities |
| Example: `github-pool` | Example: GitHub Actions authenticating to GCP |

### Q4. What is an OIDC provider?

An OIDC provider establishes trust with an external identity issuer. For example, GitHub Actions issues an OIDC token. Google Cloud validates that token through the configured provider.

### Q5. Why is WIF better than a service-account JSON key?

A JSON key is a long-lived credential that must be stored, rotated and revoked securely. WIF uses short-lived federated credentials and avoids distributing long-lived service-account private keys.

### Q6. Can AWS workloads authenticate to GCP using WIF?

Yes. Configure an AWS identity provider in a Google Cloud Workload Identity Pool, map the required AWS identity attributes, and grant the approved federated principal permission to impersonate a Google service account.

### Q7. Can Azure DevOps authenticate to GCP using WIF?

Yes. Azure DevOps can issue an OIDC token through a supported service connection. Configure a corresponding OIDC provider in GCP, validate the issuer and claims, and grant the approved identity service-account impersonation permission.

## 5. Scenario-based interview questions

### Scenario 1: GitHub Actions deploying to GKE

**Problem statement**

Your organization has 20 microservices deployed in GKE. Each microservice has a separate GitHub repository. All pipelines currently use the same GCP service-account JSON key stored in GitHub Secrets.

The security team wants to eliminate service-account keys. How would you implement this?

**Expected answer**

I would implement Workload Identity Federation:

1. Create a Workload Identity Pool.
2. Configure GitHub as an OIDC provider.
3. Configure attribute mappings for repository and branch claims.
4. Add an attribute condition to restrict approved repositories and branches.
5. Create a dedicated GCP service account for the deployment identity.
6. Grant the approved federated principal `roles/iam.workloadIdentityUser` on that service account.
7. Grant the service account the required GCP IAM permissions.
8. Configure namespace-scoped Kubernetes RBAC.
9. Use `google-github-actions/auth` in the GitHub Actions workflow.
10. Retrieve GKE credentials and deploy using `kubectl` or Helm.

**Follow-up: How do you prevent one repository from deploying to another team's namespace?**

Use Kubernetes RBAC with separate namespace-scoped RoleBindings. Also restrict the WIF provider using repository and branch claims. Ensure the deployment identity has no other broad Kubernetes permissions.

### Scenario 2: Prevent unauthorized repositories

**Problem statement**

You configured GitHub OIDC with GCP. You want to prevent unapproved repositories in your organization from authenticating through the provider. How would you do it?

**Expected answer**

Use attribute mapping and an attribute condition.

Example Terraform provider configuration:

```hcl
attribute_mapping = {
  "google.subject"       = "assertion.sub"
  "attribute.repository" = "assertion.repository"
  "attribute.ref"        = "assertion.ref"
}

attribute_condition = "assertion.repository == 'my-org/my-repo' && assertion.ref == 'refs/heads/main'"
```

This example restricts federation to the specified repository and branch. Confirm that the claims are present in the token issued by the workflow and use the appropriate audience configuration.

### Scenario 3: One GitHub pipeline, multiple GKE namespaces

**Problem statement**

Your organization has Development, QA, and Production teams. All teams deploy applications to one GKE cluster.

How would you implement access control?

**Expected answer**

Use separate service accounts or deployment identities for each environment or trust boundary. Grant each identity only the necessary GCP IAM permissions, then use Kubernetes RoleBindings scoped to the corresponding namespace.

| Team | Namespace | Access |
|---|---|---|
| Development | `dev` | Deploy and update |
| QA | `qa` | Deploy and update |
| Production | `prod` | Deploy after approval |

Use GitHub Environment protection and required reviewers for production. Keep namespace RBAC administration separate from normal application deployment.

### Scenario 4: Authentication succeeds, deployment fails

**Problem statement**

GitHub Actions successfully authenticates to GCP using WIF, but `kubectl apply -f deployment.yaml` returns:

```text
Error from server (Forbidden):
User cannot create resource deployments
```

What would you investigate?

**Expected answer**

Authentication and authorization are separate. I would check:

1. Whether GKE credentials were retrieved successfully.
2. Whether the expected Google service account identity is being used.
3. Whether the service account has the required GKE access.
4. Whether Kubernetes RBAC has a RoleBinding for that identity.
5. Whether the RoleBinding references the correct namespace.
6. Whether the role allows creation or modification of Deployments.

Useful command:

```bash
kubectl auth can-i create deployments -n app-team
```

### Scenario 5: GitHub Actions deployment to private GKE

**Problem statement**

Your GKE cluster has a private control-plane endpoint. GitHub Actions authenticates successfully but cannot connect to the Kubernetes API server. How would you solve this?

**Expected answer**

WIF solves authentication, not network connectivity. I would use a self-hosted GitHub Actions runner in the GKE VPC or a network connected to it. I would configure private DNS, routing, firewall rules and GKE control-plane access appropriately.

### Scenario 6: Excessive service-account permissions

**Problem statement**

A security review finds that an external workload can impersonate a service account with excessive permissions. How would you reduce the risk?

**Expected answer**

1. Review and remove unnecessary service-account IAM roles.
2. Restrict provider attribute conditions.
3. Remove unauthorized principal bindings.
4. Separate identities by application and environment.
5. Review Cloud Audit Logs for suspicious activity.
6. Revoke or disable compromised access where necessary.
7. Use resource-level IAM permissions instead of broad project-level permissions where possible.

## 6. Advanced interview questions

| Question | Expected discussion |
|---|---|
| How does GCP validate an OIDC token? | Issuer, signature, audience, expiry and claims |
| What is attribute mapping? | Mapping external token claims to Google attributes |
| What is an attribute condition? | Restricting which external identities are trusted |
| What is `roles/iam.workloadIdentityUser`? | Allows a federated principal to impersonate a service account |
| Difference between `roles/iam.workloadIdentityUser` and `roles/iam.serviceAccountTokenCreator`? | Different impersonation/token-generation permission models and use cases |
| How do you restrict WIF to a GitHub branch? | Map `assertion.ref` and configure an attribute condition |
| Can multiple providers exist in one pool? | Yes |
| Can one service account be accessed by multiple pools? | Yes, through appropriate IAM bindings |
| How do you audit WIF authentication? | Cloud Audit Logs and relevant IAM Credentials activity |
| Does WIF provide Kubernetes namespace authorization? | No; Kubernetes RBAC controls Kubernetes API operations |
| Does WIF provide network access to a private GKE endpoint? | No; network routing and firewall access are separate |

## 7. Hands-on interview exercise: Deploy NGINX to one GKE namespace

**Problem statement**

Build a secure deployment pipeline that deploys NGINX to a GKE namespace named `app-team` using GitHub Actions and WIF.

### Expected implementation

1. Create a GCP Workload Identity Pool.
2. Create a GitHub OIDC provider.
3. Restrict the provider to the approved GitHub repository and branch.
4. Create a dedicated GCP service account.
5. Grant the federated principal service-account impersonation permission.
6. Grant the service account the required GKE cluster discovery permission.
7. Have a cluster administrator create the `app-team` namespace and a namespace-scoped RoleBinding.
8. Configure GitHub Actions to authenticate using OIDC.
9. Retrieve GKE credentials.
10. Deploy an NGINX Deployment and Service into `app-team`.
11. Verify the rollout.
12. Confirm that the deployment identity cannot deploy into `kube-system`.

### Suggested repository structure

```text
gke-wif-deployment/
├── .github/
│   └── workflows/
│       └── deploy-gke.yml
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── k8s/
    ├── rbac/
    │   └── app-team-deployer.yaml
    └── app/
        ├── deployment.yaml
        └── service.yaml
```

### Example namespace-scoped RoleBinding

A cluster administrator applies this once:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-team
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github-deployer-edit
  namespace: app-team
subjects:
  - kind: User
    name: github-deployer@my-gcp-project.iam.gserviceaccount.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
```

The built-in `edit` ClusterRole is namespace-scoped when referenced by a RoleBinding. For production, consider a custom Role with only the required resources and verbs. Do not let the deployment identity create its own RoleBindings.

### Example NGINX Deployment

`k8s/app/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-web
  namespace: app-team
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-web
  template:
    metadata:
      labels:
        app: sample-web
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
```

### GitHub Actions workflow

`.github/workflows/deploy-gke.yml`:

```yaml
name: Deploy application to GKE

on:
  push:
    branches:
      - main
    paths:
      - "k8s/app/**"
      - ".github/workflows/deploy-gke.yml"
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Checkout source
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud using WIF
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ vars.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ vars.GCP_SERVICE_ACCOUNT }}

      - name: Get GKE credentials
        uses: google-github-actions/get-gke-credentials@v2
        with:
          cluster_name: ${{ vars.GKE_CLUSTER }}
          location: ${{ vars.GKE_LOCATION }}
          project_id: ${{ vars.GCP_PROJECT_ID }}

      - name: Verify namespace permissions
        env:
          GKE_NAMESPACE: ${{ vars.GKE_NAMESPACE }}
        run: |
          set -euo pipefail
          kubectl auth can-i create deployments -n "$GKE_NAMESPACE" | grep -qx yes
          kubectl auth can-i patch deployments -n "$GKE_NAMESPACE" | grep -qx yes
          kubectl auth can-i create services -n "$GKE_NAMESPACE" | grep -qx yes

      - name: Deploy application
        env:
          GKE_NAMESPACE: ${{ vars.GKE_NAMESPACE }}
        run: |
          kubectl apply -n "$GKE_NAMESPACE" -f k8s/app/

      - name: Wait for rollout
        env:
          GKE_NAMESPACE: ${{ vars.GKE_NAMESPACE }}
        run: |
          kubectl rollout status deployment/sample-web \
            -n "$GKE_NAMESPACE" \
            --timeout=180s

      - name: Verify deployment
        env:
          GKE_NAMESPACE: ${{ vars.GKE_NAMESPACE }}
        run: |
          kubectl get deployments,pods,services -n "$GKE_NAMESPACE"
          kubectl auth can-i create namespaces
```

The final `kubectl auth can-i create namespaces` command should return `no` when the identity has no cluster-scoped namespace creation permission. It is an informational check, not a deployment requirement.

### GitHub repository Actions variables

Configure these under **Settings → Secrets and variables → Actions → Variables**:

| Variable | Example |
|---|---|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | `projects/123456789/locations/global/workloadIdentityPools/github-pool/providers/github-oidc` |
| `GCP_SERVICE_ACCOUNT` | `github-deployer@my-gcp-project.iam.gserviceaccount.com` |
| `GCP_PROJECT_ID` | `my-gcp-project` |
| `GKE_CLUSTER` | `gke-app-cluster` |
| `GKE_LOCATION` | `asia-south1` |
| `GKE_NAMESPACE` | `app-team` |

Use the exact provider and service-account identifiers from Terraform outputs. They are identifiers, not private keys.

### Verify the deployment

```bash
kubectl get pods -n app-team
kubectl get deployments -n app-team
kubectl get services -n app-team
kubectl rollout status deployment/sample-web -n app-team
```

Check that the deployment identity cannot access another namespace:

```bash
kubectl auth can-i get deployments -n kube-system
```

This should return `no`, assuming the identity has no other broad Kubernetes RBAC grants.

### Private GKE networking note

A GitHub-hosted runner cannot directly reach a private GKE control-plane endpoint unless an approved network path exists. For a private cluster, use a self-hosted GitHub Actions runner in the GKE VPC or a connected network, and configure DNS, routing and firewall access.

## 8. Quick revision: Authentication vs authorization vs networking

| Layer | Responsibility | Example |
|---|---|---|
| GitHub OIDC | Issues external identity token | GitHub Actions job |
| WIF / WIP provider | Validates and trusts external identity | `github-pool` |
| Service account IAM | Controls Google Cloud API access | `roles/container.clusterViewer` |
| Kubernetes RBAC | Controls Kubernetes resources | RoleBinding in `app-team` |
| Network | Provides connectivity to API endpoint | Runner to private GKE control plane |

A strong interview answer should explicitly explain that successful WIF authentication does not automatically grant Kubernetes permissions or private network connectivity.

## 9. 60-second interview answer

> In my project, we use GitHub Actions to deploy applications to GKE. Earlier, pipelines used service-account JSON keys stored in GitHub Secrets. This created risks around credential leakage, rotation and long-lived credentials.
>
> To improve security, I implemented Workload Identity Federation. We created a Workload Identity Pool and configured GitHub as an OIDC provider. We mapped repository and branch claims and restricted access using attribute conditions.
>
> GitHub Actions requests a short-lived OIDC token, and Google Cloud validates it. The approved GitHub identity then impersonates a dedicated GCP service account.
>
> We grant that service account only the required Google Cloud permissions and use Kubernetes namespace-scoped RBAC to restrict deployments to the relevant application namespace.
>
> This removes the need for long-lived service-account keys and provides better security and access control.

## 10. Official documentation

- [Workload Identity Federation overview](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Configure GitHub Actions to use Workload Identity Federation](https://github.com/google-github-actions/auth)
- [GKE authentication](https://cloud.google.com/kubernetes-engine/docs/how-to/cluster-access-for-kubectl)
- [Kubernetes RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Google Cloud IAM roles](https://cloud.google.com/iam/docs/understanding-roles)

---

**Interview preparation tip:** Focus on the difference between authentication, IAM authorization, Kubernetes RBAC and network connectivity. Senior DevOps interviewers often use these scenarios to check whether you understand the complete deployment path rather than just the Terraform resources.
"""

updated = base.rstrip() + "\n\n" + interview_section
pypandoc.convert_text(
    updated,
    "md",
    format="md",
    outputfile=str(readme_path),
    extra_args=["--standalone"]
)

print(f"Updated interview-preparation README: {readme_path}")
