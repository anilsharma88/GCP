# AWS GitHub Actions OIDC Federation

**Document status:** Compliance portal draft  
**Original filename:** `AWSFedirationIdenity.md`  
**Filename rationale:** Corrected the misspelling and aligned the name with the AWS service, federation mechanism, and CI/CD platform.

## Purpose & Scope

This standard covers GitHub Actions use of GitHub OIDC tokens to authenticate to AWS by using an IAM OIDC provider and `sts:AssumeRoleWithWebIdentity`. It applies to AWS accounts, EKS clusters, IAM roles, Terraform and application deployment pipelines.

Out of scope: local developer credentials, AWS workforce identity, EC2 runtime credentials, and organization-specific approval periods.

## Control Objective & Compliance Alignment

- Eliminate long-lived AWS access keys from GitHub Secrets and runner images.
- Limit role trust to the intended issuer, audience, repository, branch, tag, or environment.
- Use least-privilege IAM permissions and environment-specific role/session conditions.
- Prevent direct production deployment by non-approved pipelines and protected branches.
- Maintain CloudTrail, GitHub audit, Terraform state, and deployment evidence.

## Technical Architecture & Implementation Details

### Authentication Flow

1. GitHub issues a short-lived OIDC token to an authorized workflow job.
2. AWS IAM validates the token issuer and audience through the registered GitHub OIDC provider.
3. The role trust policy validates repository/ref/environment claims and authorizes `sts:AssumeRoleWithWebIdentity`.
4. AWS STS returns temporary role credentials.
5. The workflow uses only permissions in the IAM role policy.

### IAM OIDC Provider

Register `https://token.actions.githubusercontent.com` once per AWS account. Enable GitHub certificate thumbprints according to current AWS guidance and organization baseline.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["<CURRENT-AWS-DOCUMENTED-THUMBPRINT>"]
}
```

### Least-Privilege Trust Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-app:environment:production"
      }
    }
  }]
}
```

Use environment, branch, or repository conditions appropriate to the deployment model. Do not use a wildcard subject unless an approved exception exists.

### GitHub Workflow

```yaml
permissions:
  id-token: write
  contents: read

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/github-actions-prod
    aws-region: us-east-1
```

Use protected environments, required reviewers, separate AWS roles by account/environment, and runner restrictions.

### Terraform and EKS Controls

- Store Terraform state in a remote encrypted backend with locking and access logs.
- Separate Terraform plan and apply identities where approval is required.
- Restrict `iam:PassRole`, role creation, policy modification, and OIDC-provider administration.
- Use EKS Pod Identity or IRSA only for pod runtime access; it is separate from CI/CD authentication.
- Configure Kubernetes RBAC independently of AWS IAM.

## Operational Controls, Risks & Guardrails

- Rotate and revoke all existing access keys before onboarding federation.
- Alert on role assumption by unexpected subjects, accounts, regions, or IP ranges.
- Separate infrastructure and Kubernetes deployment identities.
- Restrict production role assumptions to protected environments and approved runners.
- Review permissions boundaries, SCPs, KMS, and resource policies because they may deny STS credentials or API calls.
- Validate CloudTrail, GitHub audit logs, EKS audit logs, and Terraform apply evidence.

## Evidence & Audit Checklist

- IAM OIDC provider configuration and thumbprint approval.
- Role trust and permission policy exports.
- GitHub workflow and environment approval configuration.
- Protected branch/repository and runner evidence.
- CloudTrail assume-role and API activity logs.
- Terraform plan/apply logs and remote-state access evidence.
- EKS audit and Kubernetes RBAC evidence where applicable.
- Access review, exception approvals, and key-removal evidence.

## Scenario-Based Questions and Engineering Resolutions

### 1. GitHub Actions receives `Not authorized to perform sts:AssumeRoleWithWebIdentity`
**Question:** The workflow requests OIDC credentials but STS denies the role assumption. What is checked?  
**Answer:** Verify issuer, audience, subject, OIDC provider ARN, role trust policy, and whether the role exists in the correct account. Inspect the token claim set and CloudTrail. Correct the exact `sub` claim and trust conditions; do not add a wildcard as a shortcut.

### 2. A forked pull request assumes the production role
**Question:** A pull-request workflow can assume production. How is this prevented?  
**Answer:** Remove production credentials from pull-request workflows. Use a separate read-only role or no credentials, and restrict trust to `environment:production` or approved protected branch subjects. GitHub fork context must never be trusted for production.

### 3. Wildcard trust accepts every repository
**Question:** A trust policy uses `repo:*:*`. What is the impact and fix?  
**Answer:** Any repository satisfying the issuer/audience in the GitHub organization or host context may assume the role. Replace it with the exact repository and deployment context, and review all assumed-role events and pipeline definitions.

### 4. OIDC provider thumbprint configuration is rejected
**Question:** The IAM provider cannot validate GitHub OIDC. What is the likely cause?  
**Answer:** The provider URL, audience, or certificate thumbprint is incorrect or outdated. Follow current AWS certificate guidance, validate DNS/TLS issuer metadata, and test in a non-production account before updating production.

### 5. The workflow can assume the role but Terraform is denied
**Question:** STS succeeds but infrastructure operations fail. How is it isolated?  
**Answer:** Compare CloudTrail denied actions with the role policy, permissions boundary, SCP, and resource policy. Add only required actions and scopes. A successful STS call proves authentication, not authorization for all services.

### 6. The role is overprivileged
**Question:** A deployment role has `AdministratorAccess`. How is it reduced?  
**Answer:** Identify Terraform resource types and runtime APIs, generate an evidence-backed least-privilege policy, and separate plan, apply, and pod identities. Test in a sandbox before rotating or replacing the role.

### 7. Production and development share a role
**Question:** One role is used across accounts. What should change?  
**Answer:** Create account-specific identities and trust subjects. Grant only development resources to development and production resources to production, and validate account IDs in workflow configuration.

### 8. Terraform state contains IAM secret values
**Question:** Sensitive values are exposed in remote state. What is the response?  
**Answer:** Restrict backend access, enable encryption and locking, remove secrets from state where possible, rotate exposed values, and review state audit logs. State should be treated as sensitive infrastructure evidence.

### 9. State drift changes IAM trust
**Question:** An IAM role trust policy differs from Terraform. How is it corrected?  
**Answer:** Export CloudTrail and role configuration, determine whether the change is approved, reconcile or import the resource, and update state. Restrict console edits and alert on out-of-band role changes.

### 10. Pipeline has `iam:PassRole`
**Question:** A workflow can pass privileged roles to AWS services. Why is this sensitive?  
**Answer:** `iam:PassRole` can enable privilege escalation if combined with service role creation. Restrict exact role ARNs and service principals, or separate deployment and service-role administration into approved identities.

### 11. A GitHub environment does not enforce approvals
**Question:** The role trust uses `environment:production`, but no required reviewer is configured. What is required?  
**Answer:** Configure protected environments, required reviewers, branch restrictions, and deployment delays. Correct trust conditions alone do not prevent an authorized unreviewed workflow in that environment.

### 12. The pipeline deploys from the wrong branch
**Question:** A workflow from a test branch assumes a role intended for main. How is it fixed?  
**Answer:** Pin the trust subject to `ref:refs/heads/main` or use a GitHub Environment with approval controls. Verify the actual `sub` claim and prevent workflow_dispatch or tag contexts from matching unexpectedly.

### 13. CloudTrail records a role assumption but no deployment evidence exists
**Question:** An auditor cannot connect a role session to a GitHub run. What evidence is added?  
**Answer:** Capture workflow run ID, repository SHA, actor, environment, approvals, and deployment artifact digests. Add session tags only where supported and correlate them with CloudTrail and GitHub audit logs.

### 14. SCP denies EKS or ECR access
**Question:** Identity and role policy allow an action, but it is denied. What happens?  
**Answer:** AWS Organizations SCP and permissions boundaries can restrict effective permissions. Inspect the SCP, boundary, role policy, and resource policy. Update the narrowest approved policy layer, not merely the role policy.

### 15. A role can assume other roles
**Question:** Deployment credentials include `sts:AssumeRole`. How is privilege escalation assessed?  
**Answer:** Enumerate target roles and trust policies. Remove unnecessary assume-role permissions or constrain exact role ARNs and conditions. Review prior sessions and CloudTrail for unapproved cross-account assumptions.

### 16. Static access keys remain in GitHub Secrets
**Question:** Federation is configured but old keys still exist. What is the remediation?  
**Answer:** Identify, deactivate, and delete unused keys; remove secrets and code references; audit CloudTrail for usage; and add secret scanning and IAM-key policy controls.

### 17. A third-party workflow uses the trusted OIDC provider
**Question:** External actions are used in a privileged workflow. What guardrails apply?  
**Answer:** Pin actions to immutable commits, restrict marketplace actions, isolate secrets/tokens, use dedicated runners, and require review. A third-party action executes with the workflow's permissions.

### 18. EKS deployment succeeds but pod authentication fails
**Question:** The pipeline authenticates to AWS but pods cannot access S3. How is it diagnosed?  
**Answer:** CI/CD federation and pod runtime identity are separate. Verify IRSA or EKS Pod Identity association, Kubernetes service account, namespace, role trust, IAM policy, and pod token mount.

### 19. Kubernetes RBAC allows the wrong namespace
**Question:** The pipeline IAM role is narrow but it can modify all namespaces. How is it fixed?  
**Answer:** Configure namespace-scoped Kubernetes RBAC, cluster role bindings only for approved administrators, and separate service accounts/roles by environment. AWS IAM does not replace Kubernetes authorization.

### 20. The GitHub OIDC provider is created in only one account
**Question:** A central pipeline deploys across many accounts. What is required?  
**Answer:** Register the GitHub issuer in every target AWS account or use a controlled central federation role with explicit cross-account trust. Prefer separate environment account identities and test SCPs.

### 21. An emergency role assumption bypassed review
**Question:** Break-glass credentials were used outside an incident. What is required?  
**Answer:** Correlate GitHub, IAM, CloudTrail, approval, and incident records; revoke or expire the session; determine impact; notify control owners; and document follow-up and policy updates.

### 22. CloudTrail logging is disabled or incomplete
**Question:** An auditor cannot verify role activity. How is evidence restored?  
**Answer:** Re-enable organization CloudTrail, data events and relevant service logging, protect log storage with encryption and immutability, and configure alerts for trail or log-delivery failures.

### 23. A runner is compromised
**Question:** A GitHub runner obtains temporary AWS credentials. What containment applies?  
**Answer:** Revoke or limit active role sessions, rotate/reapply trust if needed, isolate the runner, inspect artifacts and CloudTrail, and increase isolation. Prefer ephemeral protected runners and short job lifetimes.

### 24. An auditor sees an explicit deny
**Question:** IAM policy allows S3 access but a deny is recorded. How is it resolved?  
**Answer:** Identify explicit deny in identity policy, resource policy, SCP, permissions boundary, VPC endpoint policy, or KMS policy. Remove or narrow the deny only through approved change control.

### 25. A new subject begins using the role
**Question:** CloudTrail shows a repository or environment not in the role inventory. What should happen?  
**Answer:** Investigate trust conditions and GitHub audit logs, suspend the role if unauthorized, preserve evidence, and reconcile the approved inventory. Add alerting for unexpected `sub` claims and account/region combinations.

## Assumptions and Missing Details

The source does not specify account IDs, GitHub organization, environments, branch/tag policy, approved runner model, Terraform backend, log retention, or regulatory framework. These must be supplied by the control owner.
