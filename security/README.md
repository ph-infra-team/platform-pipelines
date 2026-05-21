# security Pipelines

## 1. Overview

Developer opens MR
        ↓
GitLab starts pipeline
        ↓
pipeline-policy-check
        ↓
Checks .gitlab-ci.yml against platform-policies/pipeline
        ↓
If required include/stage/job missing → fail immediately
        ↓
secret-detection / checkov / tflint / fmt / validate
        ↓
plan job creates:
    tfplan
    tfplan.json
        ↓
terraform-policy-check
        ↓
Checks tfplan.json against platform-policies/terraform
        ↓
If AWS resource violates policy → fail
        ↓
Pipeline fails
        ↓
Merge request blocked

The `security` pipeline module provides **pre-execution security and compliance validation** for Infrastructure as Code (IaC).

It enforces **shift-left security** by scanning Terraform code **before any infrastructure is created**.

This module integrates with the `iac-security` image and is designed to be reused across all Terraform repositories.

---

## 2. Purpose

This pipeline ensures:

- No secrets are committed to repositories
- Infrastructure follows security best practices
- Terraform code is linted and validated
- Compliance policies are enforced before deployment

---

## 3. Pipeline Stages

```text
security → compliance → lint

4. Checks Performed
4.1 Secret Detection (Gitleaks)

Detects:

Hardcoded credentials
API keys
Tokens
Passwords

Command:

gitleaks detect --source . --no-git --redact

4.2 Terraform Linting (TFLint)

Validates:

Terraform syntax issues
Provider-specific best practices
Misconfigurations

Command:

tflint --recursive

4.3 IaC Security Scan (Checkov)

Checks for:

Open security groups
Public S3 buckets
Missing encryption
IAM misconfigurations
Compliance violations

Command:

checkov -d . --framework terraform

4.4 Policy as Code (Conftest / OPA)

Optional validation using custom policies.

Example:

conftest test . --policy policy/

Runs only if:

policy/ directory exists

5. Image Used
registry.midhtech.local:5050/infra_team/platform-images/iac-security:<tag>

Default:

1.0.0

Override using:

IAC_SECURITY_IMAGE_TAG

6. Usage in Terraform Repos

Add this to your .gitlab-ci.yml:

include:
  - project: infra_team/platform-pipelines
    ref: main
    file: security/terraform-security.yml

  - project: infra_team/platform-pipelines
    ref: main
    file: terraform/aws.yml

7. Execution Flow
1. Secret detection
2. Terraform linting
3. Security compliance scanning
4. Policy validation (optional)
5. Terraform execution (next stage)

8. Failure Behavior
Stage	Behavior
Gitleaks	Fails pipeline
TFLint	Fails pipeline
Checkov	Fails pipeline
Conftest	Optional (can be allowed to fail)

9. Required Variables

No mandatory variables required.

Optional:

IAC_SECURITY_IMAGE_TAG

10. Optional Policy Directory

To enable policy enforcement:

policy/
├── deny-public-sg.rego
├── enforce-tags.rego

If not present:

Conftest step is skipped

11. Enterprise Security Model
Layer	Responsibility
Code	Developer writes Terraform
Security Pipeline	Validates code
Terraform Pipeline	Executes code
AWS	Enforces runtime controls

12. Best Practices
Always include security pipeline before Terraform
Treat security failures as blockers
Use custom policies for organization standards
Keep tools updated via iac-security image

13. What This Pipeline Does NOT Do
Does not modify infrastructure
Does not store secrets
Does not replace cloud-native security controls

14. Benefits
Prevents insecure deployments
Enforces compliance early
Reduces production risk
Improves audit readiness

15. Example Pipeline Flow
security
  ├── gitleaks
  ├── tflint
  ├── checkov
  └── conftest (optional)

terraform
  ├── fmt
  ├── validate
  ├── plan
  └── apply (manual)
16. Future Enhancements
Custom compliance packs
Central policy repository
Integration with audit tools
Severity-based enforcement
SARIF reporting

17. Summary

This module ensures that all Terraform code is secure, compliant, and validated before execution, enabling enterprise-grade DevSecOps practices.