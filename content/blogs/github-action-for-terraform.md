---
date: '2025-04-05T18:14:36+05:30'
draft: false
title: 'GitHub Action to Automate Terraform Plan and Apply'
tags: ["github", "cicd", "terraform"]
cover: 
    image: images/gha-for-terraform.jpg
    responsiveImages: true
    linkFullImages: true
---

Managing infrastructure as code with Terraform brings consistency and scalability to DevOps workflows, but manually running the Terraform plan and application can be time-consuming and error-prone. This is a GitOps-inspired workflow: infrastructure changes are driven by pull requests, validated automatically via terraform plan, and applied on merge (with optional manual approval).

While this isn’t a strict GitOps setup (which often uses a dedicated operator like Flux or ArgoCD to sync state continuously), it follows GitOps core tenets:

- Declarative infrastructure (Terraform code in Git).
- Version-controlled changes (via PRs).
- Automated enforcement (through GitHub Actions).

In this blog, we’ll build a GitHub Actions workflow to automate terraform plan on PRs and terraform apply on merges—bringing reliability and auditability to Terraform deployments. Let’s dive in!

First, Lets understand the folder structure of the GitHub repository.
```bash
.
├── .github
│   └── workflows
│       └── plan.yaml
├── .gitignore
├── modules
└── terraform
    ├── production
    ├── sandbox
    └── uat
```
The `.github` folder is for the GitHub workflows, PR templates, and other stuff,
The `modules` folder is for terraform modules, and the `terraform` folder will have sub-folders which is used for different environments.

Now, for the GitHub Action to perform the terraform plan for any environment, we must first know where the change was made and run the terraform plan for that particular folder. For the I am using a [`dorny/paths-filter`](https://github.com/dorny/paths-filter) which can find the files that were changed based on given filters.
```yaml
- uses: dorny/paths-filter@v3
  id: changes
  with:
    list-files: shell
    filters: |
    terraform:
        - 'terraform/**'

- name: Extract directory paths
  if: steps.changes.outputs.terraform == 'true'
  id: terraform_dirs
  env:
    FILES: "${{ steps.changes.outputs.terraform_files }}"
  run: |
    # Extract directory paths using dirname and convert to JSON array
    DIRS_JSON=$(echo "$FILES" | xargs -n 1 dirname | sort -u | jq -R -s -c 'split("\n")[:-1]')
    echo "Directories: $DIRS_JSON"
    echo "terraform_dirs_json=$DIRS_JSON" >> $GITHUB_OUTPUT
```
Here the first step finds all the files that were modified in the terraform folder and the second step extracts the folder path from the list of modified files.

Now we have the folder where we need to run the terraform plan.
```yaml
- name: Terraform plan
  uses: dflook/terraform-plan@v1
  with:
    path: path_to_folder
```
Here, we are using [`dflook/terraform-plan`](https://github.com/dflook/terraform-plan) to run the terraform plan and put the terraform plan in the PR message for display.

There were just important bits for the GitHub workflow. Now let's see what the full GitHub workflow looks like.
```yaml
name: Terraform format and plan

on: [pull_request]

jobs:
  detect_changes:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: changes
        with:
          list-files: shell
          filters: |
            terraform:
              - 'terraform/**'

      - name: Extract directory paths
        if: steps.changes.outputs.terraform == 'true'
        id: terraform_dirs
        env:
          FILES: "${{ steps.changes.outputs.terraform_files }}"
        run: |
          # Extract directory paths using dirname and convert to JSON array
          DIRS_JSON=$(echo "$FILES" | xargs -n 1 dirname | sort -u | jq -R -s -c 'split("\n")[:-1]')
          echo "Directories: $DIRS_JSON"
          echo "terraform_dirs_json=$DIRS_JSON" >> $GITHUB_OUTPUT

    outputs:
      terraform_dirs_json: ${{ steps.terraform_dirs.outputs.terraform_dirs_json }}
      has_terraform_changes: ${{ steps.changes.outputs.terraform }}

  fmt_plan:
    needs: detect_changes
    if: needs.detect_changes.outputs.has_terraform_changes == 'true'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    strategy:
      fail-fast: false
      matrix:
        dir: ${{ fromJson(needs.detect_changes.outputs.terraform_dirs_json) }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set AWS role
        run: |
          if [[ "${{ matrix.dir }}" == *"production"* ]]; then
            echo "role_arn=arn:aws:iam::xxxxxxxxxxxx:role/arc-terraform-action" >> $GITHUB_ENV
          else
            echo "role_arn=arn:aws:iam::xxxxxxxxxxxx:role/arc-terraform-action" >> $GITHUB_ENV
          fi

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.role_arn }}
          aws-region: us-east-2

      - name: Terraform fmt-check
        uses: dflook/terraform-fmt-check@v1
        with:
          path: ${{ matrix.dir }}

      - name: Terraform plan
        uses: dflook/terraform-plan@v1
        with:
          path: ${{ matrix.dir }}
```
This might look confusing but let me give you the gist of it. We have 2 jobs in workflow, first, `detect_changes` which finds where the changes were made and outputs the folder, then `fmt_plan` which takes the output from the first step and uses matrix strategy and performs terraform format and plans on the output folder path.

Let's see this workflow in action now. I will create a new file `terraform/sandbox/s3.tf` with the following code:
```tf
provider "aws" {
  region = "us-east-2"
}

module "s3_bucket" {
  source      = "../../modules/s3_bucket"
  bucket_name = "test-s3-bucket-for-terraform-2025"
  versioning  = "Enabled"
}
```
I will commit and push a code to branch s3 and create PR. This will trigger a GitHub action `Terraform format and plan` which will perform terraform format and plan and show the plan if the PR is output. Terraform format [`dflook/terraform-apply`](https://github.com/dflook/terraform-github-actions/tree/main/terraform-apply) is optional but it is always good practice to perform format checks to keep the code format linear.
![pull request](/images/tf-pr.png)
To apply the changes I am using similar workflow as previous one with one change i.e. I am using [`Terraform fmt-check`](https://github.com/dflook/terraform-github-actions/tree/main/terraform-fmt-check) to perform terraform apply. Here is the full workflow.
```yaml
name: Terraform Apply

on:
  push:
    branches:
      - main

jobs:
  detect_changes:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - uses: dorny/paths-filter@v3
        id: changes
        with:
          list-files: shell
          filters: |
            terraform:
              - 'terraform/**'

      - name: Extract directory paths
        if: steps.changes.outputs.terraform == 'true'
        id: terraform_dirs
        env:
          FILES: "${{ steps.changes.outputs.terraform_files }}"
        run: |
          # Extract directory paths using dirname and convert to JSON array
          DIRS_JSON=$(echo "$FILES" | xargs -n 1 dirname | sort -u | jq -R -s -c 'split("\n")[:-1]')
          echo "Directories: $DIRS_JSON"
          echo "terraform_dirs_json=$DIRS_JSON" >> $GITHUB_OUTPUT

    outputs:
      terraform_dirs_json: ${{ steps.terraform_dirs.outputs.terraform_dirs_json }}
      has_terraform_changes: ${{ steps.changes.outputs.terraform }}

  apply:
    needs: detect_changes
    if: needs.detect_changes.outputs.has_terraform_changes == 'true'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    strategy:
      fail-fast: false
      matrix:
        dir: ${{ fromJson(needs.detect_changes.outputs.terraform_dirs_json) }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set AWS role
        run: |
          if [[ "${{ matrix.dir }}" == *"production"* ]]; then
            echo "role_arn=arn:aws:iam::xxxxxxxxxxxx:role/arc-terraform-action" >> $GITHUB_ENV
          else
            echo "role_arn=arn:aws:iam::xxxxxxxxxxxx:role/arc-terraform-action" >> $GITHUB_ENV
          fi

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.role_arn }}
          aws-region: us-east-2

      - name: Terraform Apply
        uses: dflook/terraform-apply@v1
        with:
          path: ${{ matrix.dir }}
```
Once the PR is merged `Terraform Apply` workflow is triggered which performs terraform apply.
![pr merged](/images/merged-pr.png)
By automating Terraform plans on PRs and applies on merges via GitHub Actions, teams gain both efficiency and reliability in infrastructure management. This GitOps-inspired approach enforces version-controlled changes, reduces manual errors, and provides built-in audit trails—all while maintaining the safety of peer reviews. The solution balances automation with control, making it practical for production environments yet simple enough to implement immediately. For teams adopting IaC, this workflow delivers the core benefits of GitOps without requiring complex operators, creating a foundation for scalable, collaborative infrastructure management.