# Lambda Deployer Reusable Workflow

This repository contains a reusable GitHub Actions workflow designed to simplify the deployment of Lambda functions on AWS via CloudFormation. It allows other Lambda repositories to reuse this process to create and update Lambdas in three distinct environments: `dev`, `staging`, and `prod`.

## Overview

The `lambda_deployer.yml` workflow automates the following steps:
1. **Release Verification**: Confirms that the specified version exists as a release on GitHub.
2. **Tagged Commit Checkout**: Checks out the exact commit associated with the release tag to ensure consistency.
3. **Packaging**: Creates a ZIP file with the Lambda code and its dependencies.
4. **S3 Upload**: Uploads the package to a shared S3 bucket.
5. **CloudFormation Deployment**: Creates or updates a CloudFormation stack that deploys the Lambda.

This workflow is called from a main workflow in the Lambda repository, enabling consistent deployments and rollbacks based on published releases.

## Prerequisites

Before using this workflow, ensure the following is set up:

### 1. S3 Bucket for Lambdas
- Create an S3 bucket named `ec-lambdas-bucket` in your AWS account.
- The bucket must have permissions allowing the GitHub Actions role to upload objects.
- Expected structure: Packages are uploaded to paths like `s3://ec-lambdas-bucket/dev/lambda-name-version.zip`.

### 2. Environment Variables in the Calling Repository
In the repository containing the Lambda (e.g., `lambdatest1`), configure the following variables in **Settings > Secrets and variables > Actions > Variables**:
- `AWS_ACCOUNT`: Your AWS account ID (e.g., `396401407315`).
- `ENV`: The default environment (e.g., `dev`). It must match the workflow's `env` input.

### 3. IAM Role for GitHub Actions
- Create an IAM role named `gh-actions-role` in your AWS account.
- Assign permissions for:
  - `s3:PutObject` and `s3:GetObject` on `arn:aws:s3:::ec-lambdas-bucket/*`.
  - `cloudformation:*` to create/update stacks.
  - `lambda:*` to manage Lambda functions.
  - `iam:CreateRole`, `iam:PassRole` if necessary.
- Configure the trust policy for GitHub OIDC:
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Effect": "Allow",
              "Principal": {
                  "Federated": "arn:aws:iam::YOUR_ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                  "StringEquals": {
                      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                  },
                  "StringLike": {
                      "token.actions.githubusercontent.com:sub": "repo:YOUR_ORG/*"
                  }
              }
          }
      ]
  }
  ```

### 4. Files in the Lambda Repository
The repository must contain:
- `lambda.py`: The main Lambda code.
- `requirements.txt`: Python dependencies.
- `build/cloudformation.yml`: CloudFormation template for the Lambda (see example in this repo).

## How to Use the Reusable Workflow

### 1. Configure the Main Workflow
In your Lambda repository, create a `.github/workflows/release.yml` file similar to this:

```yaml
name: Release and Deploy

on:
  workflow_dispatch:
    inputs:
      environment:
        description: "Target environment"
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
      version:
        description: "Version to deploy and release"
        required: true
        type: string

jobs:
  Release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Create release
        run: |
          if gh release view "${{ inputs.version }}" >/dev/null 2>&1; then
            echo "Release ${{ inputs.version }} already exists; skipping creation."
          else
            gh release create "${{ inputs.version }}" --generate-notes
          fi
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  Deploy:
    needs: Release
    uses: pedro-cale/PedroCalvo-Envioclick-Challenge/.github/workflows/lambda_deployer.yml@main
    with:
      component_name: YOUR_LAMBDA_NAME  # e.g., lambdatest1
      version: ${{ inputs.version }}
      env: ${{ inputs.environment }}
    permissions:
      id-token: write
      contents: write
```

### 2. Run the Workflow
- Go to the **Actions** tab in your repo.
- Select the "Release and Deploy" workflow.
- Click **Run workflow**.
- Choose the `environment` and specify the `version` (e.g., `v1.0.0`).
- Run it.

### 3. Rollbacks
To rollback to a previous version:
- Run the workflow again with the desired version.
- The workflow will verify the release exists and deploy the exact code from that tagged commit.

## Technical Details

- **Release Verification**: The workflow fails if the version does not exist as a GitHub release, preventing deployments of unpublished versions.
- **Code Consistency**: By checking out the release tag, the deployed code is guaranteed to match the release exactly.
- **Environments**: The `dev`, `staging`, `prod` environments are mapped to "Development", "Staging", "Production" for approvals and OIDC.
- **CloudFormation**: The template expects `Env`, `JobName`, and `CodeVersion` parameters. Adjust as needed.

## Troubleshooting

- **S3 Error**: Check the role permissions and bucket existence.
- **CloudFormation Error**: Review AWS logs for stack details.
- **Release Not Found**: Ensure the release is published on GitHub.
- **OIDC Failed**: Confirm the IAM role trust policy.
