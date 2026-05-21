# Ansible Shared Pipeline Library

This directory contains reusable GitLab CI/CD templates for enterprise Ansible and AWX workflows.

## Files

```text
ansible/
├── awx.yml       # AWX API integration jobs
├── awx-app.yml   # app repo pipeline for central AWX resource config and job launch
└── ee.yml        # AWX/AAP Execution Environment build and registration jobs
```

## Design Principle

The platform separates three responsibilities:

```text
GitLab pipeline images
  -> run CI jobs and call APIs

AWX inventory sync
  -> creates or updates AWX organizations, inventories, groups, and hosts

AWX execution environments
  -> run Ansible playbooks inside AWX/AAP
```

Do not put playbook runtime dependencies into the GitLab AWX CLI image unless the pipeline itself needs them. Playbook runtime dependencies belong in an Execution Environment.

## awx.yml

`awx.yml` is for interacting with AWX from GitLab CI.

Typical uses:

- validate AWX connectivity
- create/update AWX inventory
- create/update AWX groups
- create/update AWX hosts
- launch AWX job templates
- cleanup AWX objects when explicitly approved

Runtime image:

```text
registry.midhtech.local:5050/infra_team/platform-images/ansible-awx:latest
```

That image is a pipeline control image. It contains the AWX CLI and supporting tools used to call AWX APIs.

## awx-app.yml

`awx-app.yml` is for application automation repositories. It lets a team keep
its playbooks and AWX resource declarations in the same repo while the actual
AWX creation logic stays in a trusted central project.

Enterprise flow:

```text
app repo pipeline
  -> validates awx/team_vars.yml
  -> launches central-awx-configure-resources in AWX
  -> central AWX project runs approved config-as-code collection
  -> AWX Project and Job Template are created or updated for the app repo
  -> app AWX Project is synced
  -> app Job Template can be launched from GitLab with output streamed back
```

The app repo includes this template:

```yaml
include:
  - project: infra_team/platform-pipelines
    ref: main
    file: ansible/awx-app.yml
```

The app repo owns:

```text
awx/team_vars.yml
playbooks/
group_vars/
requirements.yml
```

The platform owns:

```text
central AWX organization
central AWX Project: central-awx-configure
central configure template: central-awx-configure-resources
central cleanup template: central-awx-cleanup-resources
shared AWX controller credential
shared AWS base credential
AWX custom credential types
```

Required inherited GitLab variables:

```text
AWX_HOST
AWX_USERNAME
AWX_PASSWORD
AWX_SCM_CREDENTIAL_NAME
AWX_BASE_AWS_CREDENTIAL_NAME
AWX_SSM_BUCKET_NAME
```

For AWS SSM automation, each app or team provides:

```text
AWS_ASSUME_ROLE_ARN
AWS_DEFAULT_REGION
```

`AWS_ASSUME_ROLE_ARN` is intentionally separate from the central AWS base
credential. The base credential only calls `sts:AssumeRole`; team permissions
live on the target role.

Default central names:

```text
AWX_CONFIGURE_JOB_TEMPLATE=central-awx-configure-resources
AWX_CLEANUP_JOB_TEMPLATE=central-awx-cleanup-resources
```

Use `awx-app.yml` when developers should not need AWX UI access. Their merge to
`main` updates the AWX resources through the central template, and manual jobs in
GitLab can launch or clean up the AWX resources.

## ee.yml

`ee.yml` is for building AWX/AAP Execution Environment images.

A consuming EE repo is expected to contain:

```text
execution-environment.yml
requirements.yml
requirements.txt
bindep.txt
README.md
```

The shared pipeline performs:

```text
validate
  -> confirm required files exist
  -> run ansible-builder create to validate the EE definition

build
  -> run ansible-builder build
  -> create a container image from the EE definition
  -> tag image with commit SHA, latest on main, and semantic Git tag when present
  -> push image to GitLab Container Registry

smoke
  -> run the built EE image
  -> confirm Ansible and required collections/modules are present

register
  -> optional manual job
  -> create or update the AWX Execution Environment object
  -> point AWX to the GitLab registry image
```

## Why Generic Bootstrap Images Are Used

The shared EE pipeline currently uses:

```yaml
image: python:3.11-slim
```

for validation because `ansible-builder` is a Python tool and the validate job does not need Docker.

It also uses:

```yaml
image: docker:29
```

for build because that job must run Docker client commands and talk to Docker-in-Docker to build and push the EE image.

These are bootstrap images only. They are not the final AWX Execution Environment image.

Enterprise hardening path:

```text
Phase 1: use pinned public bootstrap images
Phase 2: mirror bootstrap images into GitLab Container Registry
Phase 3: replace bootstrap images with approved internal platform images
```

For example, later replace:

```yaml
image: docker:29
```

with:

```yaml
image: registry.midhtech.local:5050/infra_team/platform-images/docker-builder:29
```

and replace:

```yaml
image: python:3.11-slim
```

with:

```yaml
image: registry.midhtech.local:5050/infra_team/platform-images/python-ci:3.11
```

## Consuming ee.yml

Example `.gitlab-ci.yml` in an EE repo:

```yaml
include:
  - project: infra_team/platform-pipelines
    file: ansible/ee.yml

variables:
  AWX_EE_NAME: "ee-aws-ssm"
  EE_REGISTER_IN_AWX: "false"
```

To register/update the EE in AWX from CI, set:

```text
EE_REGISTER_IN_AWX=true
AWX_USERNAME=<awx service account>
AWX_PASSWORD=<masked password>
```

Optional variables:

```text
AWX_HOST=http://awx.midhtech.local
AWX_EE_PULL=always
AWX_EE_REGISTRY_CREDENTIAL_NAME=gitlab-container-registry
EE_SMOKE_EXECUTABLES=session-manager-plugin
EE_IMAGE_TAG=$CI_COMMIT_SHORT_SHA
EE_LATEST_TAG=latest
```

## Image Tags

The shared pipeline pushes:

```text
<registry>/<project>:<commit-sha>
<registry>/<project>:latest        # main branch only
<registry>/<project>:<git-tag>     # tag pipeline only
```

For production AWX job templates, prefer immutable version tags over `latest`.

## Registry Authentication

If the GitLab Container Registry is private, AWX must have credentials to pull the EE image.

Configure an AWX container registry credential and attach it to the Execution Environment.

Example AWX credential:

```text
Name: gitlab-container-registry
Credential Type: Container Registry
Authentication URL: registry.midhtech.local:5050
Username: <gitlab deploy token or service account>
Password: <token with read_registry>
Verify SSL: false only if the registry uses an untrusted internal certificate
```

Then set this in the EE repo:

```text
AWX_EE_REGISTRY_CREDENTIAL_NAME=gitlab-container-registry
```

Without this credential, AWX may create the job successfully but fail before the
playbook starts with `ImagePullBackOff`.

If the registry uses plain HTTP or an internal CA, the AWX execution nodes or
Kubernetes container runtime must also trust that registry. The GitLab runner
Docker-in-Docker `--insecure-registry` setting only helps CI build/push; it does
not configure AWX runtime image pulls.

For AWX Operator on k3s with an HTTP GitLab registry, configure k3s on the AWX
node:

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "registry.midhtech.local:5050":
    endpoint:
      - "http://registry.midhtech.local:5050"
```

Then restart k3s:

```bash
systemctl restart k3s
```

The AWX node must also resolve the registry hostname. In the lab environment,
the GitLab VM serves both GitLab and the registry:

```text
192.168.1.101 gitlab.midhtech.local registry.midhtech.local
```

If AWX reports `ImagePullBackOff`, inspect the AWX namespace events. A message
like `lookup registry.midhtech.local: Try again` is a DNS problem, not an AWX
credential problem.
