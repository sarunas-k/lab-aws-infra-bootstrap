# lab-aws-infra-bootstrap

CloudFormation seed stacks for the lab AWS organization. CloudFormation is used here only for the
chicken-and-egg resources that Terraform cannot create for itself.

| Stage | Account | Contents | Status |
|---|---|---|---|
| `stage1/` | lab-management | Terraform state bucket, GitHub OIDC provider, plan/apply roles for `lab-aws-infra-organization` | ready |
| `stage2/` | management (StackSets) | `member-account.yaml`: OIDC provider + CI roles per workload account. `shared-state.yaml`: platform state bucket | ready, deploy after `lab-aws-infra-organization` |

## Stage 1

Prerequisites: AWS CLI v2, an admin login to the management account (SSO or a temporary admin profile), MFA on root.

```bash
aws cloudformation deploy \
  --stack-name lab-bootstrap-stage1 \
  --template-file stage1/seed.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides GitHubOwner=<your-github-user-or-org>
```

Outputs: state bucket name and the two role ARNs. Use them in the `lab-aws-infra-organization` backend config and workflow.

- `lab-github-org-plan`: read-only, assumable only from `pull_request` runs of `lab-aws-infra-organization`.
- `lab-github-org-apply`: write access, assumable only from the protected GitHub Environment `org-apply`.
- The state bucket is versioned, encrypted, private, TLS-only and retained if the stack is deleted.
- If the GitHub OIDC provider already exists in the account, pass `CreateOidcProvider=false`.

The apply role grants Organizations, IAM Identity Center, CloudTrail, Budgets and Account service permissions.
It is broad by design for an org-management pipeline, so protect it with the environment approval and
the strict branch ruleset on `lab-aws-infra-organization`.

## Validate locally

```bash
pip install cfn-lint && cfn-lint stage1/seed.yaml
```

## Stage 2

Run after `lab-aws-infra-organization` has created the accounts and OUs. Service-managed StackSets need trusted access
(`member.org.stacksets.cloudformation.amazonaws.com`, enabled by `lab-aws-infra-organization`).

```bash
# Workloads OU: OIDC provider and CI roles. IAM is global, so deploy to the home region only.
aws cloudformation create-stack-set --stack-set-name lab-member-account \
  --template-body file://stage2/member-account.yaml \
  --permission-model SERVICE_MANAGED --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameters ParameterKey=GitHubOwner,ParameterValue=<owner> \
    ParameterKey=DevAccountId,ParameterValue=<id> ParameterKey=TestAccountId,ParameterValue=<id> \
    ParameterKey=ProdAccountId,ParameterValue=<id> ParameterKey=SharedServicesAccountId,ParameterValue=<id>
aws cloudformation create-stack-instances --stack-set-name lab-member-account \
  --deployment-targets OrganizationalUnitIds=<workloads-ou-id> --regions eu-central-1

# Infrastructure OU: platform state bucket.
aws cloudformation create-stack-set --stack-set-name lab-shared-state \
  --template-body file://stage2/shared-state.yaml --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=false --parameters ParameterKey=OrganizationId,ParameterValue=<o-xxxx> \\
    ParameterKey=DevAccountId,ParameterValue=<id> ParameterKey=TestAccountId,ParameterValue=<id> ParameterKey=ProdAccountId,ParameterValue=<id>
aws cloudformation create-stack-instances --stack-set-name lab-shared-state \
  --deployment-targets OrganizationalUnitIds=<infrastructure-ou-id> --regions eu-central-1
```

Roles created in each workload account (the environment is derived from the account id):

| Role | Assumable from | Permissions |
|---|---|---|
| `lab-github-platform-plan` | `lab-aws-infra-environments` pull requests | ReadOnlyAccess + state lock |
| `lab-github-cost-guard` | `lab-aws-infra-environments` `main` (scheduled and manual runs) | ReadOnlyAccess |
| `lab-github-platform-apply` | `lab-aws-infra-environments` GitHub Environment `<env>` | AdministratorAccess (single-purpose account, SCPs, approval gate) |
| `lab-github-ecr-push` | `lab-aws-backend`, `lab-aws-web`, Environment `<env>` | Push to `lab-*` ECR repositories |
| `lab-github-assets-sync` | `lab-aws-web`, Environment `<env>` | Sync `images/` of `lab-<env>-assets` |

State isolation: the shared bucket policy lets each environment account use only its own `envs/<env>/` prefix
(by `aws:PrincipalAccount`), so the dev apply role cannot read or modify test or prod state. Principals are the `lab-github-platform-*` roles and the IAM Identity Center (`AWSReservedSSO_*`) roles of the operator, who runs `scripts/destroy.sh` locally. Terraform backends in
`lab-aws-infra-environments` must therefore use keys like `envs/<env>/<stack>/terraform.tfstate`. The `lab-shared-state`
StackSet also needs `DevAccountId`, `TestAccountId` and `ProdAccountId` parameters.
