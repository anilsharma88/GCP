from pathlib import Path

path = Path("/mnt/data/README.md")
content = """# Workload Identity Federation (WIF): GCP and Azure

## 1. What is Workload Identity Federation?

Workload Identity Federation (WIF) lets an external workload prove its identity to a cloud provider using a trusted identity provider (for example, GitHub Actions OIDC) and obtain short-lived cloud credentials.

It avoids storing long-lived service-account keys, client secrets, or other static credentials in CI/CD systems.

Typical flow:

1. A workflow requests an OIDC token from GitHub Actions.
2. The cloud provider validates the token's issuer, audience, and subject claims.
3. The provider maps the trusted external identity to a cloud identity.
4. The workflow receives short-lived credentials and accesses only the resources allowed by IAM/RBAC.

## 2. Azure equivalent of GCP WIF

The Azure equivalent is **Microsoft Entra Workload Identity Federation**.

There is no exact one-to-one Azure resource equivalent to a GCP Workload Identity Pool. In Azure, you configure a **federated identity credential** on either:

- A **User-assigned Managed Identity (UAMI)** — commonly used for automation and workloads.
- An **App Registration** — commonly used when an application/service principal is the chosen identity.

The federated identity credential establishes trust with an external identity provider such as GitHub Actions. Azure RBAC then grants access to Azure resources.

### GCP-to-Azure mapping

| GCP | Azure |
|---|---|
| Workload Identity Federation | Microsoft Entra Workload Identity Federation |
| Workload Identity Pool + OIDC Provider | Federated identity credential on a UAMI or App Registration |
| GCP service account | Azure managed identity or service principal |
| `roles/iam.workloadIdentityUser` binding | Federated identity trust configuration (not a direct RBAC role equivalent) |
| GCP IAM roles | Azure RBAC roles |
| GitHub Actions OIDC | GitHub Actions OIDC with `azure/login` |

## 3. Configure Azure WIF for GitHub Actions

This example trusts only the `main` branch of one GitHub repository.

### Prerequisites

- An Azure subscription.
- Permission to create a user-assigned managed identity and federated credential.
- Permission to assign Azure RBAC roles at the intended scope.
- A GitHub repository where the workflow will run.

Set the values for your environment:

```bash
RESOURCE_GROUP="rg-platform"
LOCATION="centralindia"
IDENTITY_NAME="github-deployer"
GITHUB_ORG="my-org"
GITHUB_REPO="my-repo"
```

### Step 1: Create a user-assigned managed identity

```bash
az identity create \\
  --name "$IDENTITY_NAME" \\
  --resource-group "$RESOURCE_GROUP" \\
  --location "$LOCATION"
```

Get its client ID and principal ID:

```bash
CLIENT_ID=$(az identity show \\
  --name "$IDENTITY_NAME" \\
  --resource-group "$RESOURCE_GROUP" \\
  --query clientId -o tsv)

PRINCIPAL_ID=$(az identity show \\
  --name "$IDENTITY_NAME" \\
  --resource-group "$RESOURCE_GROUP" \\
  --query principalId -o tsv)

TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
```

### Step 2: Create the federated identity credential

Create `federated-credential.json`:

```json
{
  "name": "github-main-branch",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:my-org/my-repo:ref:refs/heads/main",
  "description": "Trust GitHub Actions from the main branch",
  "audiences": [
    "api://AzureADTokenExchange"
  ]
}
```

Replace `my-org/my-repo` with the exact GitHub organization and repository.

Create the credential:

```bash
az identity federated-credential create \\
  --name "github-main-branch" \\
  --identity-name "$IDENTITY_NAME" \\
  --resource-group "$RESOURCE_GROUP" \\
  --parameters federated-credential.json
```

The subject must match the token's subject claim exactly. For example, a branch-based subject is different from a pull-request or GitHub Environment subject.

### Step 3: Assign Azure RBAC permissions

Grant only the permissions needed by the workflow. For example, to allow Contributor access to one resource group:

```bash
az role assignment create \\
  --assignee-object-id "$PRINCIPAL_ID" \\
  --assignee-principal-type ServicePrincipal \\
  --role "Contributor" \\
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP"
```

`Contributor` is broad. In production, prefer a narrower built-in or custom role and the smallest practical scope.

## 4. GitHub Actions workflow

Add these GitHub repository variables:

- `AZURE_CLIENT_ID` — client ID of the user-assigned managed identity.
- `AZURE_TENANT_ID` — Entra tenant ID.
- `AZURE_SUBSCRIPTION_ID` — Azure subscription ID.

Example `.github/workflows/azure-login.yml`:

```yaml
name: Azure login using Workload Identity Federation

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Azure login using OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Verify Azure identity
        run: |
          az account show
          az group show --name rg-platform
```

The workflow does not need a client secret. `id-token: write` allows GitHub Actions to request the OIDC token used for federation.

## 5. GCP vs. Azure vs. AWS

| Capability | GCP | Azure | AWS |
|---|---|---|---|
| Federation service | Workload Identity Federation | Entra Workload Identity Federation | IAM OIDC federation |
| Trust configuration | Workload Identity Pool and Provider | Federated identity credential | IAM role trust policy |
| Cloud identity | Service account | Managed identity or service principal | IAM role |
| Authorization | GCP IAM | Azure RBAC | AWS IAM policies |
| GitHub Actions integration | `google-github-actions/auth` | `azure/login` | `aws-actions/configure-aws-credentials` |

## 6. Important distinction: CI/CD federation vs. Kubernetes workload identity

Do not confuse GitHub Actions federation with pod identity:

- **GitHub Actions to Azure:** configure Entra Workload Identity Federation on a UAMI or App Registration.
- **Pods running in AKS:** use **AKS Workload Identity**, which integrates Kubernetes service-account tokens with Microsoft Entra ID.
- **Pods running in GKE:** use **GKE Workload Identity Federation for GKE**.
- **Applications running on Azure VMs or supported Azure services:** use Managed Identity where possible.

## 7. Security and troubleshooting checklist

- Match the federated credential's issuer, subject, and audience exactly to the OIDC token.
- Restrict trust to the required repository, branch, tag, or GitHub Environment.
- Do not grant subscription-wide Contributor if resource-group or resource-level access is sufficient.
- Keep `id-token: write` limited to jobs that need federation.
- Check that the workflow uses the same managed identity whose federated credential was configured.
- Confirm the identity has the required Azure RBAC role at the correct scope.
- Allow a short propagation period after creating federated credentials or role assignments.
- Check the GitHub Actions run logs and Azure sign-in/audit logs when authentication or authorization fails.

## 8. Interview answer

**Question: What is the Azure equivalent of GCP Workload Identity Federation?**

**Answer:** Microsoft Entra Workload Identity Federation. In GCP, we configure a Workload Identity Pool and OIDC provider and map an external identity to a GCP service account. In Azure, we configure a federated identity credential on a user-assigned managed identity or an App Registration. GitHub Actions obtains an OIDC token, Entra validates its issuer, subject, and audience, and the workflow authenticates without storing a client secret. We then use Azure RBAC to grant least-privilege access to the required resources.

## 9. Official documentation

- [Microsoft Entra Workload ID federation](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation)
- [Configure a federated identity credential on a user-assigned managed identity](https://learn.microsoft.com/en-us/entra/workload-id/workload-identity-federation-create-trust-user-assigned-managed-identity)
- [Azure Login GitHub Action](https://github.com/Azure/login)
- [Google Cloud Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
"""
path.write_text(content, encoding="utf-8")
print(f"Created {path} ({path.stat().st_size} bytes)")
