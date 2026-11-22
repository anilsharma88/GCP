from pathlib import Path

path = Path("/mnt/data/README.md")
content = r"""# Federated Workload Identity and GitHub Actions: AWS, Azure, and GCP

This guide compares cloud-native Workload Identity Federation (WIF) and shows how GitHub Actions can provision infrastructure and deploy a containerized application to one Kubernetes namespace in dev, test, and prod.

Cloud targets:
- AWS: Amazon EKS
- Azure: Azure Kubernetes Service (AKS)
- Google Cloud: Google Kubernetes Engine (GKE)

## 1. AWS equivalent of Workload Identity Federation

AWS uses **IAM OIDC federation**. GitHub Actions requests an OIDC token, and AWS Security Token Service (STS) exchanges it for short-lived credentials through `AssumeRoleWithWebIdentity`.

There is no direct AWS resource equivalent to a GCP Workload Identity Pool. AWS trust is configured using an IAM OIDC provider and an IAM role trust policy.

### Cloud comparison

| Capability | AWS | Azure | GCP |
|---|---|---|---|
| Federation | IAM OIDC federation | Microsoft Entra Workload Identity Federation | Workload Identity Federation |
| Trust configuration | IAM OIDC provider + IAM role trust policy | Federated identity credential on UAMI or App Registration | Workload Identity Pool + OIDC provider |
| Cloud identity | IAM role | Managed identity or service principal | GCP service account |
| Token exchange | AWS STS `AssumeRoleWithWebIdentity` | Microsoft Entra token exchange | Google STS |
| Authorization | IAM policies | Azure RBAC | GCP IAM |
| GitHub Action | `aws-actions/configure-aws-credentials` | `azure/login` | `google-github-actions/auth` |

**Interview shortcut:** GCP uses a Workload Identity Pool and provider; Azure uses a federated identity credential on a managed identity or App Registration; AWS uses an IAM OIDC provider and IAM role trust policy. All three avoid long-lived cloud keys in GitHub.

## 2. GitHub repository structure

```text
platform-app/
├── .github/workflows/
│   ├── deploy-aws.yml
│   ├── deploy-azure.yml
│   └── deploy-gcp.yml
├── infra/
│   └── environments/
│       ├── dev/
│       ├── test/
│       └── prod/
├── k8s/
│   ├── namespace.yaml
│   ├── serviceaccount.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── app/
    └── Dockerfile
```

Create GitHub Environments named `dev`, `test`, and `prod`. Configure environment-specific variables and protection rules under **Settings → Environments**. Require reviewers for production and restrict which branches or tags can deploy.

Suggested promotion model:

| Environment | Trigger | Approval |
|---|---|---|
| dev | Push to `develop` | Automatic |
| test | Approved promotion or push to `main` | Optional |
| prod | Release tag or approved promotion | Required reviewer |

Build and scan the application image once, then promote the same immutable image digest through environments.

## 3. AWS EKS: configure GitHub OIDC

### 3.1 Create the AWS IAM OIDC provider

Create the provider once per AWS account if it does not already exist:

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

Typical provider ARN:

```text
arn:aws:iam::<AWS_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

### 3.2 IAM role trust policy

This example trusts only the `main` branch of one repository:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:my-org/platform-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Replace the account ID, organization, and repository. If using a GitHub Environment, the subject claim is different; for example:

```text
repo:my-org/platform-app:environment:prod
```

Configure the trust policy to match the exact subject GitHub issues. Avoid broad wildcard subjects.

### 3.3 IAM permissions

Attach least-privilege permissions to the role. Depending on your design, the role may need permissions for:
- Terraform-managed AWS resources.
- ECR image push and image metadata operations.
- EKS cluster description and access integration.
- Supporting services used by the Terraform configuration.

Avoid unrestricted `AdministratorAccess`. In a mature platform, separate infrastructure provisioning and application deployment roles.

### 3.4 GitHub Actions AWS login

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Configure AWS credentials using OIDC
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/github-eks-deployer
      aws-region: ap-south-1

  - name: Verify AWS identity
    run: aws sts get-caller-identity
```

Cloud authentication does not automatically grant Kubernetes access. Configure an EKS access entry (or the supported cluster access mechanism) for the IAM role, then grant only the required Kubernetes namespace permissions.

## 4. Azure AKS: Entra Workload Identity Federation

Azure's equivalent is **Microsoft Entra Workload Identity Federation**. Configure a federated identity credential on a user-assigned managed identity (UAMI) or App Registration.

Example credential for a GitHub branch:

```json
{
  "name": "github-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:my-org/platform-app:ref:refs/heads/main",
  "description": "GitHub Actions main branch",
  "audiences": ["api://AzureADTokenExchange"]
}
```

GitHub Actions login and AKS context:

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Azure login using OIDC
    uses: azure/login@v2
    with:
      client-id: ${{ vars.AZURE_CLIENT_ID }}
      tenant-id: ${{ vars.AZURE_TENANT_ID }}
      subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

  - name: Get AKS credentials
    uses: azure/aks-set-context@v4
    with:
      resource-group: ${{ vars.AKS_RESOURCE_GROUP }}
      cluster-name: ${{ vars.AKS_CLUSTER_NAME }}
```

Assign only the Azure roles required for infrastructure and AKS access. Azure subscription Contributor does not automatically grant Kubernetes namespace permissions. For a private AKS API, use a self-hosted runner with connectivity to the private endpoint.

## 5. GCP GKE: Workload Identity Federation

GCP uses a Workload Identity Pool and OIDC provider to trust GitHub Actions, then allows the federated principal to impersonate a Google service account.

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: actions/checkout@v4

  - name: Authenticate to Google Cloud using WIF
    uses: google-github-actions/auth@v2
    with:
      workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
      service_account: ${{ vars.GCP_DEPLOYER_SERVICE_ACCOUNT }}

  - name: Get GKE credentials
    uses: google-github-actions/get-gke-credentials@v2
    with:
      cluster_name: ${{ vars.GKE_CLUSTER_NAME }}
      location: ${{ vars.GKE_LOCATION }}
      project_id: ${{ vars.GCP_PROJECT_ID }}
```

Grant the service account only the IAM permissions required for Terraform and GKE access. Kubernetes RBAC is a separate authorization layer. For a private GKE control plane, the runner needs network access to the endpoint.

## 6. Kubernetes application in one namespace

These are cloud-neutral Kubernetes manifests. The example deploys NGINX in a namespace called `app-team`.

### Namespace: `k8s/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: app-team
  labels:
    app.kubernetes.io/part-of: platform-app
```

### Runtime service account: `k8s/serviceaccount.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-runtime
  namespace: app-team
automountServiceAccountToken: false
```

If the application needs cloud API access, configure a separate pod runtime identity:
- EKS: EKS Pod Identity or IRSA.
- AKS: Microsoft Entra Workload ID.
- GKE: Workload Identity Federation for GKE.

Do not reuse the GitHub Actions deployment identity as the application's runtime identity.

### Deployment: `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app-team
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      serviceAccountName: app-runtime
      containers:
        - name: web
          image: nginx:1.27
          ports:
            - name: http
              containerPort: 80
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 15
            periodSeconds: 20
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            runAsUser: 101
            capabilities:
              drop: ["ALL"]
```

For your own application, replace the sample image with the immutable image built by CI. Confirm that the image supports non-root execution; adapt the security context if it does not.

### Service: `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: app-team
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - name: http
      port: 80
      targetPort: http
```

### Namespace-scoped RBAC example

Bootstrap this using a platform administrator or a controlled platform process. The subject below is a placeholder: replace it with the authenticated identity string presented by your cluster's cloud authentication integration.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: application-deployer
  namespace: app-team
rules:
  - apiGroups: ["", "apps"]
    resources: ["configmaps", "services", "serviceaccounts", "deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: application-deployer
  namespace: app-team
subjects:
  - kind: User
    name: REPLACE_WITH_AUTHENTICATED_DEPLOYER_IDENTITY
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: application-deployer
  apiGroup: rbac.authorization.k8s.io
```

Tailor the Role's resources and verbs to the manifests the pipeline manages. Avoid cluster-admin and broad `edit` permissions when a narrower Role is sufficient. Prefer pre-creating namespaces so the application pipeline does not need cluster-wide namespace creation privileges.

## 7. Example GitHub Actions workflow: Terraform + EKS application deployment

This is a learning/reference workflow. It assumes:
- `infra/environments/dev`, `test`, and `prod` contain working Terraform configurations.
- Each environment uses a remote state backend and its own state key.
- The AWS OIDC provider and IAM roles already exist.
- The IAM role is allowed to provision the resources in that environment and access the intended EKS cluster.
- The namespace and namespace-scoped RBAC are bootstrapped.
- The sample NGINX image is used; an application build-and-push stage is not included below.

Create `.github/workflows/deploy-aws.yml`:

```yaml
name: Deploy infrastructure and application to EKS

on:
  push:
    branches: [develop, main]
  workflow_dispatch:
    inputs:
      environment:
        description: Target environment
        type: choice
        required: true
        options: [dev, test, prod]

permissions:
  contents: read
  id-token: write

concurrency:
  group: eks-deploy-${{ github.workflow }}-${{ inputs.environment || github.ref_name }}
  cancel-in-progress: false

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment || (github.ref_name == 'develop' && 'dev' || 'test') }}

    env:
      AWS_REGION: ap-south-1
      TF_IN_AUTOMATION: "true"
      NAMESPACE: app-team

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Select environment
        id: target
        shell: bash
        run: |
          if [[ -n "${{ inputs.environment }}" ]]; then
            ENV_NAME="${{ inputs.environment }}"
          elif [[ "${GITHUB_REF_NAME}" == "develop" ]]; then
            ENV_NAME="dev"
          elif [[ "${GITHUB_REF_NAME}" == "main" ]]; then
            ENV_NAME="test"
          else
            echo "Unsupported branch: ${GITHUB_REF_NAME}" >&2
            exit 1
          fi
          echo "name=${ENV_NAME}" >> "$GITHUB_OUTPUT"

      - name: Configure AWS credentials using OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Verify AWS identity
        run: aws sts get-caller-identity

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform init
        working-directory: infra/environments/${{ steps.target.outputs.name }}
        run: terraform init

      - name: Terraform validate
        working-directory: infra/environments/${{ steps.target.outputs.name }}
        run: terraform validate

      - name: Terraform plan
        working-directory: infra/environments/${{ steps.target.outputs.name }}
        run: terraform plan -out=tfplan

      - name: Terraform apply
        working-directory: infra/environments/${{ steps.target.outputs.name }}
        run: terraform apply -auto-approve tfplan

      - name: Configure kubectl for EKS
        run: |
          aws eks update-kubeconfig \
            --region "$AWS_REGION" \
            --name "${{ vars.EKS_CLUSTER_NAME }}"

      - name: Deploy application to one namespace
        run: |
          kubectl apply -n "$NAMESPACE" -f k8s/serviceaccount.yaml
          kubectl apply -n "$NAMESPACE" -f k8s/deployment.yaml
          kubectl apply -n "$NAMESPACE" -f k8s/service.yaml

      - name: Verify rollout
        run: |
          kubectl rollout status deployment/web \
            --namespace "$NAMESPACE" --timeout=180s
          kubectl get deployments,pods,services \
            --namespace "$NAMESPACE"
```

### Environment variables

Create GitHub Environments `dev`, `test`, and `prod`. Configure these variables separately in each environment:

| Variable | Example |
|---|---|
| `AWS_DEPLOY_ROLE_ARN` | Environment-specific IAM role ARN |
| `EKS_CLUSTER_NAME` | `eks-platform-dev`, `eks-platform-test`, or `eks-platform-prod` |

Use the same variable names in each environment but different values and role ARNs. Set the production environment to require reviewers and restrict allowed deployment branches/tags.

**Important production improvements:**
- The workflow above is intentionally simplified. For production, use separate infrastructure and application deployment roles/jobs.
- Add a required approval between production Terraform plan and apply.
- Keep separate remote state keys per environment and serialize applies.
- Build, scan, push, and deploy an immutable image digest; do not use mutable tags such as `latest`.
- Pin third-party GitHub Actions to full commit SHAs in production.
- Do not let routine application deployments create namespaces or modify cluster-wide RBAC.
- Use a self-hosted runner with private network connectivity for private Kubernetes API endpoints.
- Avoid letting Terraform and `kubectl` manage the same Kubernetes objects.

## 8. Azure and GCP Kubernetes deployment steps

The Terraform and Kubernetes deployment concepts remain the same. Replace the cloud login and kubeconfig steps.

### Azure AKS

```yaml
- name: Azure login using OIDC
  uses: azure/login@v2
  with:
    client-id: ${{ vars.AZURE_CLIENT_ID }}
    tenant-id: ${{ vars.AZURE_TENANT_ID }}
    subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

- name: Get AKS credentials
  uses: azure/aks-set-context@v4
  with:
    resource-group: ${{ vars.AKS_RESOURCE_GROUP }}
    cluster-name: ${{ vars.AKS_CLUSTER_NAME }}

- name: Deploy to app-team
  run: |
    kubectl apply -n app-team -f k8s/serviceaccount.yaml
    kubectl apply -n app-team -f k8s/deployment.yaml
    kubectl apply -n app-team -f k8s/service.yaml
    kubectl rollout status deployment/web -n app-team --timeout=180s
```

### GCP GKE

```yaml
- name: Authenticate to Google Cloud using WIF
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: ${{ vars.GCP_WIF_PROVIDER }}
    service_account: ${{ vars.GCP_DEPLOYER_SERVICE_ACCOUNT }}

- name: Get GKE credentials
  uses: google-github-actions/get-gke-credentials@v2
  with:
    project_id: ${{ vars.GCP_PROJECT_ID }}
    cluster_name: ${{ vars.GKE_CLUSTER_NAME }}
    location: ${{ vars.GKE_LOCATION }}

- name: Deploy to app-team
  run: |
    kubectl apply -n app-team -f k8s/serviceaccount.yaml
    kubectl apply -n app-team -f k8s/deployment.yaml
    kubectl apply -n app-team -f k8s/service.yaml
    kubectl rollout status deployment/web -n app-team --timeout=180s
```

Use separate identities, Terraform state, clusters, and registry permissions for each environment. For private AKS/GKE control planes, the runner must have the appropriate network path.

## 9. Recommended enterprise pipeline

```text
Pull request
  ├── Terraform fmt / validate / plan
  ├── IaC security scan
  ├── Unit tests / SAST / dependency scan
  └── Container build and image scan
          |
Merge to develop
  ├── OIDC authentication
  ├── Terraform plan and apply (dev)
  ├── Push immutable image to registry
  ├── Deploy to dev/app-team
  └── Rollout and smoke tests
          |
Promote same image digest to test
  ├── Approval / policy checks
  ├── Deploy to test/app-team
  └── Integration tests
          |
Promote same image digest to prod
  ├── Required GitHub Environment approval
  ├── Review and apply Terraform changes if needed
  ├── Deploy to prod/app-team
  └── Health checks and rollback decision
```

Use Terraform for cloud infrastructure and platform foundations. Use Kubernetes manifests, Helm, or Kustomize for application releases.

## 10. Troubleshooting

| Symptom | Likely cause | Check |
|---|---|---|
| AWS `AssumeRoleWithWebIdentity` denied | OIDC provider or trust claim mismatch | Provider ARN, `aud`, exact `sub`, `id-token: write` |
| Azure federated login fails | Federated credential mismatch | Issuer, subject, audience, client ID, tenant ID |
| GCP WIF fails | Provider or service-account binding mismatch | Provider name, audience, attribute mapping/condition, impersonation binding |
| Cloud login works but Kubernetes returns Forbidden | Kubernetes authorization missing | EKS access entry/RBAC, AKS RBAC, GKE IAM and Kubernetes RBAC |
| Cannot connect to cluster | Private endpoint/network issue | Runner network, DNS, routes, firewall |
| Terraform state conflict | Concurrent apply or backend issue | Remote backend, locking, workflow concurrency |
| Image pull failure | Registry identity or image reference issue | Node/pod pull permissions, repository, tag/digest, network |
| Rollout timeout | Failed probes, crash, image pull, insufficient capacity | Pod events, describe, logs, resource requests |

Useful commands:

```bash
kubectl get all -n app-team
kubectl get events -n app-team --sort-by=.lastTimestamp
kubectl describe deployment web -n app-team
kubectl describe pods -n app-team
kubectl logs deployment/web -n app-team --all-containers=true
kubectl auth can-i patch deployments -n app-team
kubectl rollout history deployment/web -n app-team
```

## 11. Interview questions and model answers

### Q1. What is the AWS equivalent of GCP Workload Identity Federation?

**Answer:** AWS uses IAM OIDC federation. We configure an IAM OIDC provider for GitHub and an IAM role trust policy that validates the token issuer, audience, and subject. GitHub Actions uses `aws-actions/configure-aws-credentials` to exchange the OIDC token through AWS STS and receive temporary credentials.

### Q2. Compare WIF in AWS, Azure, and GCP.

**Answer:** GCP uses a Workload Identity Pool and provider; Azure uses a federated identity credential on a managed identity or App Registration; AWS uses an IAM OIDC provider and IAM role trust policy. Each maps an external identity to a cloud identity, then uses the cloud authorization system to grant permissions.

### Q3. Why is OIDC better than storing cloud access keys in GitHub Secrets?

**Answer:** OIDC avoids long-lived static credentials. The cloud issues short-lived credentials only when token claims satisfy the configured trust conditions. This reduces secret leakage and rotation risk and lets us restrict trust to a repository, branch, or environment.

### Q4. How do you restrict a GitHub Actions role to production only?

**Answer:** Use a protected GitHub Environment named `prod`, require reviewers, restrict deployment branches/tags, and configure the cloud trust policy to accept only the exact GitHub Environment subject. Give the role only production-scoped permissions.

### Q5. Does successful cloud authentication mean the workflow can deploy to Kubernetes?

**Answer:** No. Cloud authentication and Kubernetes authorization are separate. An AWS role can authenticate to EKS but still require an EKS access entry and Kubernetes RBAC. The equivalent authorization layers must also be configured for AKS and GKE.

### Q6. How do you restrict a pipeline to one Kubernetes namespace?

**Answer:** Pre-create the namespace and bind the deployment identity to a namespace-scoped Kubernetes Role using a RoleBinding. Grant only the resources and verbs required by the deployment. Do not grant cluster-admin to the pipeline.

### Q7. How do you deploy the same application to dev, test, and prod?

**Answer:** Build and scan the image once, push it to a registry, and promote the same immutable image digest through environments. Use separate GitHub Environments, cloud identities, Terraform state, clusters, and approvals.

### Q8. How do you manage Terraform state across environments?

**Answer:** Use a remote backend with locking and separate state keys per environment. Restrict backend access, enable encryption and versioning where supported, and serialize applies to prevent concurrent state changes.

### Q9. How do you handle a private Kubernetes API endpoint?

**Answer:** Run the workflow on a self-hosted runner with network connectivity to the private cluster. Configure DNS, routes, and firewall rules. Do not expose the Kubernetes API publicly only to accommodate a hosted runner.

### Q10. What do you do if deployment fails after applying manifests?

**Answer:** Check rollout status, pod events, probes, image pull status, and application logs. If the previous ReplicaSet is healthy, perform a controlled rollback to it and investigate the cause. Ensure database changes are backward compatible before relying on application rollback.

### Q11. How do you prevent developers from deploying directly to production?

**Answer:** Enforce branch protection, CODEOWNERS and pull-request reviews; protect the production GitHub Environment with reviewers; restrict cloud trust to approved refs/environments; and use least-privilege deployment identities.

### Q12. How do you secure containers and Kubernetes workloads?

**Answer:** Scan dependencies and images, use trusted minimal base images, pin image digests, run as non-root, drop Linux capabilities, set resource requests/limits and probes, apply network policies, use namespace RBAC, and store secrets in an approved secret manager.

### Q13. What is the difference between the GitHub deployment identity and the pod runtime identity?

**Answer:** The deployment identity is used by CI/CD to provision infrastructure and submit Kubernetes resources. The pod runtime identity is used by the application inside the cluster to access cloud services. They should be separate identities with different least-privilege permissions.

### Q14. Would you use Terraform or kubectl to deploy an application?

**Answer:** Terraform is well suited to cloud infrastructure and platform foundations. Kubernetes manifests, Helm, or Kustomize are generally better for frequent application releases. Avoid managing the same Kubernetes object with both Terraform and kubectl.

### Q15. How do you design rollback and zero-downtime deployment?

**Answer:** Use rolling updates with multiple replicas, readiness probes, graceful shutdown, suitable `maxUnavailable` and `maxSurge`, and backward-compatible changes. Verify rollout and smoke tests. Roll back to the previous immutable image digest if health checks fail.

### Q16. How do you prevent two environments from using the same cloud permissions?

**Answer:** Use separate IAM roles, managed identities, or service accounts for dev, test, and prod. Restrict each trust policy to the matching GitHub Environment or approved ref, scope permissions to that environment's resources, and use separate state backends/keys.

### Q17. How do you handle Terraform drift?

**Answer:** Run scheduled or pre-deployment plans, review drift, determine whether the out-of-band change is valid, and either update code or reconcile the infrastructure. Do not blindly apply a plan that could destroy an intentional emergency fix.

### Q18. How do you deploy a new image without rebuilding it for every environment?

**Answer:** Build once, scan, push to the registry, record the image digest, and promote that exact digest through dev, test, and prod. This gives artifact consistency and traceability.

## 12. Official documentation

- AWS: [Configure AWS credentials for GitHub Actions](https://github.com/aws-actions/configure-aws-credentials)
- AWS: [IAM OIDC identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- AWS: [EKS access entries](https://docs.aws.amazon.com/eks/latest/userguide/access-entries.html)
- Azure: [Workload identity federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- Azure: [Azure Login GitHub Action](https://github.com/Azure/login)
- Azure: [AKS identity concepts](https://learn.microsoft.com/en-us/azure/aks/concepts-identity)
- GCP: [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- GCP: [Get GKE credentials GitHub Action](https://github.com/google-github-actions/get-gke-credentials)
- GitHub: [OpenID Connect in GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- Kubernetes: [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- Kubernetes: [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
"""
path.write_text(content, encoding="utf-8")
print(f"Created {path} ({path.stat().st_size:,} bytes)")
