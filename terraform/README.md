# Terraform AWS Shared Pipeline Library

## 1. Overview

The `terraform/aws.yml` file is a reusable GitLab CI/CD shared pipeline template for Terraform-based AWS infrastructure provisioning.

It is maintained centrally in:

```text
infra_team/platform-pipelines

and consumed by Terraform application/live repositories using GitLab include.

This pipeline provides a standard enterprise workflow for:

Terraform formatting
Terraform validation
Terraform planning
Manual apply on main
GitLab-managed Terraform state
AWS authentication using CI/CD variables
Optional destroy workflow
Optional Terraform output handoff for AWX inventory sync

2. Why This Shared Pipeline Exists

Without a shared pipeline library, every Terraform repository would define its own CI/CD logic.

That creates problems:

Different teams run Terraform differently
Tool versions drift
Apply/destroy controls are inconsistent
Security checks may be missed
Pipeline updates must be repeated across many repos

This shared pipeline solves that by centralizing the Terraform execution model.

3. Enterprise Design Principle
Terraform repositories = infrastructure code only
Shared pipeline library = CI/CD execution logic
Platform images = approved runtime tools
GitLab variables = secrets and runtime configuration

This creates separation of responsibility:

Layer	Responsibility
Terraform repo	Terraform code
platform-pipelines	CI/CD logic
platform-images	Terraform/AWS CLI runtime
GitLab CI/CD variables	Secrets and environment values
GitLab Runner	Execution engine
4. Pipeline Behavior
Branch Behavior
Branch	Jobs Executed
develop / feature branches	fmt, validate, plan
main	fmt, validate, plan, apply
main with destroy extension	destroy available manually
5. Default Stages

The shared pipeline defines:

stages:
  - fmt
  - validate
  - plan
  - apply
  - configure

The configure stage is included for optional extensions, such as AWX inventory sync.

6. Runtime Image

The pipeline uses the approved Terraform platform image:

registry.midhtech.local:5050/infra_team/platform-images/terraform-aws:1.0.0

This image includes:

Terraform
AWS CLI
jq
git
curl

The image version can be overridden using:

TERRAFORM_IMAGE_TAG

Example:

TERRAFORM_IMAGE_TAG=1.0.1
7. Terraform State Backend

This pipeline uses GitLab-managed Terraform state through the HTTP backend.

Each Terraform repository must include:

terraform {
  backend "http" {}
}

The pipeline dynamically sets:

TF_HTTP_ADDRESS
TF_HTTP_LOCK_ADDRESS
TF_HTTP_UNLOCK_ADDRESS
TF_HTTP_LOCK_METHOD
TF_HTTP_UNLOCK_METHOD
TF_HTTP_USERNAME
TF_HTTP_PASSWORD

State is stored under the project’s Terraform state area in GitLab:

Project → Operate → Terraform states
8. Terraform State Name

By default, the pipeline uses:

TF_STATE_NAME=$CI_PROJECT_NAME

You can override it in GitLab CI/CD variables:

TF_STATE_NAME=dev
TF_STATE_NAME=network-dev
TF_STATE_NAME=gitlab-runner-platform

Recommended enterprise pattern:

Use Case	Recommended State Name
Single environment repo	project name default
Multi-environment repo	dev, stage, prod
Platform component	component/environment name
9. Required CI/CD Variables

Set these in the consuming Terraform project:

Project → Settings → CI/CD → Variables
AWS Variables

Because the current GitLab instance is self-managed over local HTTP, OIDC to AWS is not used yet.

Required variables:

Variable	Example	Notes
AWS_ACCESS_KEY_ID	AKIA...	Masked, protected
AWS_SECRET_ACCESS_KEY	xxxx	Masked, protected
AWS_DEFAULT_REGION	us-east-1	Protected
GitLab Terraform State Variables

Required for GitLab HTTP backend access:

Variable	Example	Notes
TF_HTTP_USERNAME	harip or token username	Protected
TF_HTTP_PASSWORD	GitLab PAT / project access token	Masked, protected

The token should have API access required for Terraform state operations.

Optional Image Version Variable
Variable	Example	Notes
TERRAFORM_IMAGE_TAG	1.0.0	Optional override

If not set, the pipeline uses:

1.0.0
Optional Destroy Variables

Only needed when a repo explicitly enables destroy:

Variable	Example
ALLOW_DESTROY	true
CHANGE_TICKET_ID	CHG123456

Destroy will not run without both variables.

Optional AWX Inventory Sync Variables

Only needed when a Terraform repo extends AWX inventory sync:

Variable	Example
AWX_HOST	http://awx.midhtech.local
AWX_USERNAME	admin
AWX_PASSWORD	masked password
AWX_INVENTORY	inventory ID or name
ANSIBLE_AWX_IMAGE_TAG	1.0.0

10. Minimal Usage in Terraform App Repo

In a Terraform repository, create .gitlab-ci.yml:

include:
  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/aws.yml

That is enough to get:

fmt → validate → plan

on all branches, and:

manual apply

on main.

11. Required Terraform Repo Files

A consuming Terraform repo should contain:

terraform-app-repo/
├── .gitlab-ci.yml
├── backend.tf
├── versions.tf
├── provider.tf
├── main.tf
├── variables.tf
└── outputs.tf
backend.tf
terraform {
  backend "http" {}
}
versions.tf
terraform {
  required_version = "~> 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
provider.tf
provider "aws" {
  region = var.region
}
12. Example Terraform Repo Usage
.gitlab-ci.yml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/aws.yml
GitLab Variables
AWS_ACCESS_KEY_ID=<masked>
AWS_SECRET_ACCESS_KEY=<masked>
AWS_DEFAULT_REGION=us-east-1

TF_HTTP_USERNAME=harip
TF_HTTP_PASSWORD=<masked token>
TF_STATE_NAME=vpc-dev

Pipeline behavior:

develop branch:
  fmt
  validate
  plan

main branch:
  fmt
  validate
  plan
  apply manual
13. Optional Destroy Usage

Destroy is not enabled by default.

If a repository needs destroy capability, explicitly add this in that repo:

include:
  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/aws.yml

destroy:
  stage: destroy
  extends: .terraform_destroy

Also add the stage locally if needed:

stages:
  - fmt
  - validate
  - plan
  - apply
  - configure
  - destroy

Required CI/CD variables:

ALLOW_DESTROY=true
CHANGE_TICKET_ID=CHG123456

Destroy behavior:

main branch only
manual only
requires explicit destroy flag
requires change ticket
14. Optional AWX Inventory Sync Usage

Some Terraform repos may create EC2 instances that need to be registered into AWX inventory.

This is optional and must be explicitly enabled.

Example:

include:
  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/aws.yml

sync-awx-inventory:
  stage: configure
  extends: .terraform_awx_sync
  needs:
    - job: apply

Required Terraform output:

output "instance_ips" {
  value = aws_instance.this[*].private_ip
}

Required CI/CD variables:

AWX_HOST=http://awx.midhtech.local
AWX_USERNAME=<awx-user>
AWX_PASSWORD=<masked>
AWX_INVENTORY=<inventory-name-or-id>

This job reads Terraform output:

terraform output -json

Then creates AWX hosts from:

instance_ips
15. What This Pipeline Does Not Do

This pipeline does not:

Store AWS credentials in images
Store AWX credentials in images
Run AWX job templates by default
Manage application deployments
Replace Terraform modules
Replace Ansible/AWX playbooks

AWX job launch logic should live in:

ansible/awx.yml

not in:

terraform/aws.yml
16. Security Model
Secrets

Secrets must be stored only in:

GitLab CI/CD protected variables

Never store secrets in:

Terraform files
Docker images
.gitlab-ci.yml
README files
Git history
AWS Authentication

Current lab setup:

GitLab CI/CD variables → AWS credentials

Future enterprise target:

GitLab HTTPS OIDC → AWS IAM Role → temporary credentials

OIDC requires GitLab to be reachable by AWS through trusted HTTPS.

Apply Control

Apply is:

main branch only
manual approval only

This prevents accidental infrastructure changes from feature branches.

Destroy Control

Destroy is:

not available by default
manual only
main branch only
requires ALLOW_DESTROY=true
requires CHANGE_TICKET_ID
17. Governance Rules

Consuming repositories should:

Use this shared pipeline instead of custom Terraform logic
Use approved Terraform modules
Use approved platform images
Store state in GitLab Terraform state
Use protected variables for secrets
Keep apply manual
Keep destroy disabled unless approved
18. Enterprise Operating Model
Platform team owns:
  - platform-pipelines
  - platform-images
  - runner configuration
  - image version rollout

Application / infrastructure teams own:
  - Terraform code
  - module inputs
  - environment variables
  - merge requests
19. Image Upgrade Model

The shared pipeline pins the Terraform image version:

terraform-aws:1.0.0

To upgrade:

Build new image in platform-images
Test image
Update TERRAFORM_IMAGE_TAG default in shared pipeline
Merge to main
Consuming repos inherit the new version

For temporary override, a repo may set:

TERRAFORM_IMAGE_TAG=1.0.1
20. Audit Statement

This shared Terraform pipeline standardizes AWS infrastructure delivery by enforcing consistent formatting, validation, planning, manual approval, state management, and runtime controls across Terraform repositories. Secrets are externalized through GitLab CI/CD variables, and high-risk operations such as apply and destroy are restricted through branch, manual approval, and change-control gates.