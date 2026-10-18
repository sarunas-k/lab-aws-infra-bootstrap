# lab-bootstrap

CloudFormation seed stacks for the lab AWS organization. CloudFormation is used here only for the
chicken-and-egg resources that Terraform cannot create for itself.

| Stage | Account | Contents | Status |
|---|---|---|---|
| `stage1/` | lab-management | Terraform state bucket, GitHub OIDC provider, plan/apply roles for `lab-infra-org` | ready |
| `stage2/` | management (StackSet) | OIDC provider + deployer role in every member account, platform state bucket | after `lab-infra-org` |

## Stage 1

Prerequisites: AWS CLI v2, an admin login to the management account (SSO or a temporary admin profile), MFA on root.

```bash
aws cloudformation deploy \
  --stack-name lab-bootstrap-stage1 \
  --template-file stage1/seed.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides GitHubOwner=<your-github-user-or-org>
```

Outputs: state bucket name and the two role ARNs. Use them in the `lab-infra-org` backend config and workflow.

- `lab-github-org-plan`: read-only, assumable only from `pull_request` runs of `lab-infra-org`.
- `lab-github-org-apply`: write access, assumable only from the protected GitHub Environment `org-apply`.
- The state bucket is versioned, encrypted, private, TLS-only and retained if the stack is deleted.
- If the GitHub OIDC provider already exists in the account, pass `CreateOidcProvider=false`.

The apply role grants Organizations, IAM Identity Center, CloudTrail, Budgets and Account service permissions.
It is broad by design for an org-management pipeline, so protect it with the environment approval and
the strict branch ruleset on `lab-infra-org`.

## Validate locally

```bash
pip install cfn-lint && cfn-lint stage1/seed.yaml
```
