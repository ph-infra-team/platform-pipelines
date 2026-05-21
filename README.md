# Platform Pipelines (Shared CI/CD Library)

---

# 1. Overview

`platform-pipelines` is the centralized enterprise CI/CD template repository for the platform engineering organization.

This repository provides reusable, governed, and security-enforced GitLab CI/CD pipeline templates used across all infrastructure and automation repositories.

The primary goal of this repository is to standardize:

- Infrastructure delivery
- Terraform execution
- Security scanning
- Compliance enforcement
- Governance workflows
- AWX automation integration
- Runtime consistency
- Pipeline architecture
- Infrastructure lifecycle management

This repository is part of the broader enterprise platform ecosystem:

| Repository | Purpose |
|---|---|
| `platform-pipelines` | Shared CI/CD templates |
| `platform-images` | Standardized runtime container images |
| `platform-policies` | OPA/Rego governance policies |
| `tf-modules` | Approved Terraform modules |
| `live repositories` | Environment-specific infrastructure |

---

# 2. Enterprise Architecture

```text
Developer Push
      ↓
GitLab Pipeline
      ↓
platform-pipelines
      ↓
Security & Governance Enforcement
      ↓
Terraform Validation & Planning
      ↓
OPA / Rego Policy Validation
      ↓
Manual Approval
      ↓
Terraform Apply
      ↓
AWS Infrastructure Provisioned
      ↓
AWX Inventory Sync (Optional)
      ↓
Ansible Configuration Automation
      ↓
Application Deployment / System Configuration
```

---

# 3. Repository Structure

```text
platform-pipelines/
├── README.md
│
├── security/
│   ├── README.md
│   ├── common-security.yml
│   ├── pipeline-policy.yml
│   └── gitleaks.toml
│
├── terraform/
│   ├── README.md
│   │
│   ├── live/
│   │   ├── aws.yml
│   │   └── terragrunt.yml
│   │
│   └── modules/
│       ├── aws.yml
│       └── terragrunt.yml
│
└── ansible/
    ├── README.md
    └── awx.yml
```

---

# 4. Purpose of This Repository

This repository exists to solve enterprise-scale CI/CD challenges.

## 4.1 Standardization

All infrastructure repositories execute through a centralized and approved pipeline framework.

No custom Terraform execution logic should exist in application repositories.

---

## 4.2 Governance

The repository enforces:

- Branch restrictions
- Manual approvals
- Security validation
- Policy-as-code enforcement
- Controlled destroy workflows
- Runtime standardization
- Auditability

---

## 4.3 Security

Shift-left DevSecOps controls are enforced before infrastructure creation.

This includes:

- Secret scanning
- Terraform linting
- Compliance validation
- OPA policy validation
- Terraform plan governance

---

## 4.4 Operational Consistency

All repositories use:

- Same Terraform version
- Same AWS CLI version
- Same security tooling
- Same runtime images
- Same governance model

---

# 5. Pipeline Categories

This repository provides multiple pipeline categories.

| Pipeline | Purpose |
|---|---|
| `terraform/live/aws.yml` | Deploy live AWS infrastructure |
| `terraform/modules/aws.yml` | Validate & publish Terraform modules |
| `security/common-security.yml` | Security & compliance scanning |
| `security/pipeline-policy.yml` | OPA governance validation |
| `ansible/awx.yml` | AWX inventory and automation |

---

# 6. Design Principles

---

## 6.1 Centralized Pipeline Logic

Application repositories consume templates.

Pipeline logic is maintained only in this repository.

This prevents:

- Pipeline drift
- Version inconsistency
- Security bypasses
- Tool fragmentation

---

## 6.2 Immutable Runtime Environments

Pipelines run using centrally managed container images.

This guarantees:

- Consistent tool versions
- Reproducible builds
- Controlled upgrades
- Reduced operational variance

---

## 6.3 Governance First

Governance is enforced before infrastructure deployment.

Examples:

- Apply is manual
- Destroy requires explicit approval
- Security checks fail pipelines
- Policies validate infrastructure standards

---

## 6.4 Infrastructure as Code Compliance

Terraform is treated as governed code.

Infrastructure must pass:

- Formatting
- Validation
- Security scanning
- Compliance scanning
- OPA governance checks

before deployment is allowed.

---

## 6.5 Separation of Responsibilities

| Component | Responsibility |
|---|---|
| Live Repositories | Infrastructure definitions |
| Platform Pipelines | CI/CD execution logic |
| Platform Images | Runtime tooling |
| Platform Policies | Governance & compliance |
| AWX | Configuration automation |
| GitLab Variables | Secret management |

---

# 7. How Consuming Repositories Use This

Application repositories do NOT implement their own CI/CD logic.

They only include approved templates.

---

# 8. Terraform Live Repository Example

## Example Repository

```text
cloud_team/live/aws/dev/networking/vpc
```

---

## `.gitlab-ci.yml`

```yaml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: security/common-security.yml

  - project: infra_team/platform-pipelines
    ref: main
    file: security/pipeline-policy.yml

  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/live/aws.yml
```

---

# 9. Terraform Module Repository Example

## Example Repository

```text
cloud_team/tf-modules/aws/networking/tf-aws-vpc
```

---

## `.gitlab-ci.yml`

```yaml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: security/common-security.yml

  - project: infra_team/platform-pipelines
    ref: main
    file: security/pipeline-policy.yml

  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/modules/aws.yml
```

---

# 10. Security Pipeline

## File

```text
security/common-security.yml
```

---

## Purpose

Provides centralized DevSecOps scanning for all repositories.

---

## Security Tools

| Tool | Purpose |
|---|---|
| Gitleaks | Secret detection |
| TFLint | Terraform linting |
| Checkov | Terraform compliance scanning |
| Conftest | OPA policy enforcement |

---

## Security Stages

```text
security
governance
compliance
lint
```

---

## Security Philosophy

Infrastructure should fail fast before deployment.

No infrastructure should be created if:

- Secrets exist
- Terraform violates standards
- Compliance checks fail
- Policies are violated

---

# 11. Pipeline Governance

## File

```text
security/pipeline-policy.yml
```

---

## Purpose

Enforces enterprise pipeline governance using OPA/Rego.

---

## Example Governance Policies

- Required stages exist
- Security jobs cannot be removed
- Approved shared templates must be used
- Manual apply required
- Destroy restrictions enforced
- Prevent custom bypass pipelines

---

## Enterprise Benefit

Prevents engineers from bypassing platform standards.

---

# 12. Terraform Live Pipeline

## File

```text
terraform/live/aws.yml
```

---

# 13. Live Infrastructure Workflow

## Stages

```text
security
governance
compliance
lint
fmt
validate
plan
policy
apply
destroy
configure
```

---

## Pipeline Flow

```text
fmt
  ↓
validate
  ↓
plan
  ↓
OPA policy validation
  ↓
manual approval
  ↓
apply
  ↓
optional AWX sync
```

---

# 14. Terraform Backend

Terraform state is stored using GitLab HTTP backend.

---

## Backend Configuration

```bash
TF_HTTP_ADDRESS
TF_HTTP_LOCK_ADDRESS
TF_HTTP_UNLOCK_ADDRESS
```

---

## Benefits

- Centralized state storage
- GitLab-native locking
- No separate S3 backend dependency
- Simplified management
- Built-in access control

---

# 15. AWS Authentication

AWS authentication currently uses GitLab CI/CD variables.

---

## Required Variables

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION
```

---

## Future Architecture

Planned future enhancement:

```text
GitLab OIDC
    ↓
AWS IAM Role
    ↓
Temporary Credentials
```

---

# 16. Terraform Governance

## Apply Controls

Apply is:

- Manual
- Main branch only
- Policy-gated

---

## Destroy Controls

Destroy requires:

```text
ALLOW_DESTROY=true
CHANGE_TICKET_ID=CHG12345
```

This prevents accidental infrastructure deletion.

---

# 17. Terraform Module Pipeline

## File

```text
terraform/modules/aws.yml
```

---

# 18. Module Pipeline Purpose

This pipeline validates and certifies reusable Terraform modules.

Modules are treated as enterprise software artifacts.

---

# 19. Module Pipeline Stages

```text
security
governance
compliance
lint
fmt
validate
plan
policy
release
```

---

# 20. Module Validation Strategy

Modules must contain runnable examples.

Example:

```text
examples/basic/main.tf
```

The pipeline:

- Initializes example
- Validates module
- Executes Terraform plan
- Runs policy checks
- Publishes certified module

---

# 21. Terraform Module Publishing

Modules publish to GitLab Terraform Registry.

---

## Release Trigger

```text
v1.0.0
v1.1.0
v2.0.0
```

---

## Published Artifact

```text
module-name-system-version.tgz
```

---

## Enterprise Benefits

- Versioned modules
- Centralized module registry
- Governance-controlled releases
- Standardized module lifecycle

---

# 22. AWX Integration Pipeline

## File

```text
ansible/awx.yml
```

---

# 23. Purpose

Integrates Terraform infrastructure provisioning with Ansible Automation Platform (AWX).

---

# 24. Workflow

```text
Terraform creates infrastructure
        ↓
Terraform outputs generated
        ↓
Pipeline extracts server IPs
        ↓
AWX inventory updated
        ↓
AWX jobs execute
        ↓
Servers configured automatically
```

---

# 25. AWX Features

| Feature | Description |
|---|---|
| Inventory Sync | Create/update inventories |
| Host Management | Create/update hosts |
| Group Management | Create/update groups |
| Job Execution | Run job templates |
| Cleanup | Controlled removal operations |

---

# 26. AWX Cleanup Governance

Cleanup operations require:

```text
AWX_CLEANUP_ENABLED=true
CHANGE_TICKET_ID=CHG12345
```

This prevents accidental automation removal.

---

# 27. Runtime Images

All pipelines execute using centralized platform images.

---

## Registry

```text
registry.midhtech.local:5050/infra_team/platform-images
```

---

## Available Images

| Image | Purpose |
|---|---|
| terraform-aws | Terraform + AWS CLI |
| iac-security | Security scanning tools |
| ansible-awx | AWX + Ansible tooling |

---

# 28. Runtime Standardization

Benefits:

- Consistent tooling
- Centralized upgrades
- Predictable execution
- Reduced troubleshooting
- Easier governance

---

# 29. OPA / Rego Governance

Policy-as-Code is enforced using:

- OPA
- Conftest
- Rego policies

---

## Policy Repository

```text
platform-policies
```

---

## Governance Areas

| Policy Category | Example |
|---|---|
| Pipeline Governance | Required stages |
| Terraform Governance | Approved modules |
| Security Standards | Encryption required |
| Naming Standards | Enterprise naming |
| Tagging Standards | Mandatory tags |
| Compliance Controls | HIPAA / Security |

---

# 30. Required CI/CD Variables

## Terraform Variables

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION

TF_HTTP_USERNAME
TF_HTTP_PASSWORD

TF_STATE_NAME (optional)
```

---

## AWX Variables

```text
AWX_HOST
AWX_USERNAME
AWX_PASSWORD
AWX_INVENTORY
AWX_JOB_TEMPLATE
```

---

## Optional Governance Variables

```text
ALLOW_DESTROY
CHANGE_TICKET_ID
AWX_CLEANUP_ENABLED
AWX_CLEANUP_MODE
```

---

# 31. Branching Strategy

| Branch | Purpose |
|---|---|
| develop | Pipeline development |
| main | Production-ready templates |

---

# 32. Recommended Workflow

```text
feature branch
    ↓
develop
    ↓
pipeline validation
    ↓
merge request
    ↓
main
```

---

# 33. Lint Validation

The repository validates all shared templates automatically using GitLab CI lint API.

This ensures:

- All templates remain syntactically valid
- Includes resolve correctly
- Stages exist
- Pipeline inheritance works

The latest validation passed successfully.

---

# 34. Enterprise Governance Model

The platform follows a centralized platform engineering model.

---

## Platform Team Responsibilities

- Maintain pipeline templates
- Maintain runtime images
- Maintain governance policies
- Control versions
- Maintain runners
- Define enterprise standards

---

## Application Team Responsibilities

- Write Terraform code
- Consume approved templates
- Define infrastructure
- Request approvals
- Follow governance standards

---

# 35. Security Model

## Secret Management

Secrets are:

- Stored in GitLab variables
- Masked
- Protected
- Never committed to repositories

---

## Pipeline Restrictions

| Operation | Restriction |
|---|---|
| Apply | Manual |
| Destroy | Approval required |
| Cleanup | Controlled |
| Security bypass | Not allowed |

---

# 36. Future Roadmap

Planned enhancements include:

- GitLab OIDC integration
- Terragrunt orchestration
- Multi-account AWS deployments
- EKS deployment pipelines
- Helm deployment pipelines
- ArgoCD GitOps pipelines
- Kubernetes policy enforcement
- Drift detection
- Automated compliance reporting
- SecurityHub integration

---

# 37. Enterprise Best Practices

## DO

- Use approved templates
- Use centralized modules
- Store secrets in GitLab variables
- Use manual approvals
- Follow tagging standards
- Use policy-approved modules

---

## DO NOT

- Create custom CI/CD logic
- Hardcode credentials
- Bypass governance controls
- Disable security scanning
- Modify runtime images locally
- Use unapproved Terraform resources

---

# 38. Summary

`platform-pipelines` is the central enterprise CI/CD orchestration framework for infrastructure automation.

It provides:

- Standardized Terraform delivery
- Enterprise governance
- Security enforcement
- Policy-as-code validation
- Controlled infrastructure lifecycle management
- AWX integration
- Centralized runtime consistency
- Scalable platform engineering architecture

This repository is a foundational component of the organization's enterprise cloud platform strategy.