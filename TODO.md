# TODO — GitHub Actions Pipeline with OIDC

## 1. Configure AWS OIDC Identity Provider

Create an IAM OIDC identity provider for GitHub Actions in your AWS account:

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`
- Thumbprint (optional — AWS can fetch it automatically)

## 2. Create an IAM Role for the Pipeline

Create an IAM role with:

- **Trust policy** allowing the OIDC provider to assume the role for your GitHub repo (`cloud-labs-vpc` / your-org-or-user).
- **Permissions policy** granting at minimum:
  - `ec2:*` (to manage VPC resources)
  - `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the state bucket (`pedrin-clabs-tfmodule-kk1`)
  - `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:DeleteItem` on the lock table (`pedrin-clterraform-locks`)

You can do this via the AWS Console, CLI, or a separate Terraform stack.

## 3. Add GitHub Actions Secret

Add a repository secret named `AWS_ROLE_ARN` with the ARN of the IAM role created above (e.g. `arn:aws:iam::123456789012:role/github-actions-terraform`).

## 4. Create the Workflow File

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy VPC

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

env:
  TF_VERSION: "1.5.7"

jobs:
  deploy:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

## 5. Commit & Push

```bash
git add .github/workflows/deploy.yml TODO.md
git commit -m "ci: add GitHub Actions pipeline with OIDC"
git push
```

## 6. Verify

- Open the repository on GitHub → Actions tab.
- Trigger the workflow manually (`workflow_dispatch`) or push to `main`.
- Check that Terraform init, plan, and apply succeed.
- Confirm the VPC is created/updated in your AWS account.

## Optional Enhancements

- [ ] Add `terraform fmt --check` and `tflint` to the pipeline.
- [ ] Use `terraform plan` output as a PR comment.
- [ ] Add a separate `apply` job that runs only after manual approval.
- [ ] Store `terraform.tfplan` as a workflow artifact between plan and apply.
- [ ] Use environment protection rules (`environment: production`) on the apply step.
