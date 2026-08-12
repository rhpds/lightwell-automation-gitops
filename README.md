# Lightwell Automation GitOps

GitOps repository for the **Red Hat Lightwell** trusted software supply chain demo.

An ArgoCD Application (bootstrapped by the [AgnosticV cluster CI](https://github.com/rhpds/agnosticv)) points at this repo to deploy and manage demo infrastructure on OpenShift.

## What is Lightwell?

Red Hat + IBM's Lightwell is a trusted software supply chain initiative. **Lightwell Network** provides remediated, digitally signed open-source dependencies. **Lightwell Clearinghouse** coordinates vulnerability disclosure and patching across the ecosystem.

This demo shows how enterprises integrate Lightwell with their build pipelines to detect, triage, and remediate CVEs in application dependencies.

## Repository Structure

```
bootstrap-infra/          Helm chart — cluster-scoped shared infrastructure
  templates/
    aap/                  Ansible Automation Platform (Controller + EDA)
    argocd/               ArgoCD admin password configuration
    artifactory/          JFrog Artifactory OSS (proxy to Lightwell Network repos)
    gitlab/               GitLab CE (source code management, CI/CD)
    jenkins/              Jenkins (build automation)
    quay/                 Red Hat Quay (container image registry)
    rhads/                RHTAS (Trusted Artifact Signer) + TPA (Trusted Profile Analyzer)
    rhoai/                OpenShift AI (operator + DataScienceCluster)
    tekton/               OpenShift Pipelines (build/sign/scan/push demo pipeline)
    sonarqube/            SonarQube (static code analysis)
  values.yaml             Configurable values (domain, passwords, image tags, channels)
  Chart.yaml

bootstrap-tenant/         Helm chart — per-user tenant resources (placeholder)
  templates/
  values.yaml
  Chart.yaml
```

## Components

| Component | Purpose | Namespace |
|-----------|---------|-----------|
| **GitLab CE** | Source code management and CI/CD pipelines | `gitlab` |
| **RHTAS** (Trusted Artifact Signer) | Keyless code signing via Sigstore/cosign | `lightwell-tas` |
| **RHTPA** (Trusted Profile Analyzer) | SBOM ingestion, vulnerability analysis, VEX | `lightwell-tpa` |
| **Artifactory OSS** | Maven/npm proxy to Lightwell Network repositories | `lightwell-artifactory` |
| **Jenkins** | Build pipeline automation | `lightwell-jenkins` |
| **SonarQube** | Static code analysis and quality gates | `lightwell-sonarqube` |
| **Red Hat Quay** | Container image registry with Clair vulnerability scanning | `lightwell-quay` |
| **AAP** (Ansible Automation Platform) | Automation orchestration and event-driven automation | `aap` |
| **OpenShift AI** | Data science dashboard, workbenches, pipelines, model serving | `redhat-ods-operator` / `redhat-ods-applications` |
| **OpenShift Pipelines** | Tekton-based build/sign/scan/push demo pipeline | `lightwell-tssc` |

## How It's Deployed

1. The AgnosticV **cluster CI** (`agd_v2/lightwell-infra-cluster`) provisions an OpenShift cluster with Keycloak auth and the OpenShift GitOps operator.
2. The `ocp4_workload_gitops_bootstrap` Ansible workload creates an ArgoCD `Application` CR pointing at `bootstrap-infra/` in this repo.
3. ArgoCD syncs the Helm chart, deploying all components in dependency order via sync-wave annotations.
4. The AgnosticV **tenant CI** (`agd_v2/lightwell-infra-tenant`) provisions per-user namespaces and Showroom on top of the shared cluster.

### Values Passed from Ansible

The cluster CI injects runtime values into the Helm chart:

```yaml
ocp4_workload_gitops_bootstrap_helm_values:
  deployer:
    domain: "{{ openshift_cluster_ingress_domain }}"
  admin:
    password: "{{ common_admin_password }}"
  gitlab:
    rootPassword: "{{ common_admin_password }}"
  jenkins:
    password: "{{ common_admin_password }}"
```

## Key Technologies

- **OpenShift 4.22** — target platform
- **OpenShift GitOps (ArgoCD)** — app-of-apps deployment
- **RHTAS** (Red Hat Trusted Artifact Signer) — Sigstore-based artifact signing (cosign, Rekor, Fulcio)
- **RHTPA** (Red Hat Trusted Profile Analyzer) — SBOM and vulnerability analysis
- **Ansible Automation Platform** — orchestration (via AgnosticV/AgnosticD)

## Development

To test chart rendering locally:

```bash
helm template lightwell-infra bootstrap-infra/ \
  --set deployer.domain=apps.example.com \
  --set admin.password=testpass
```

To validate the chart:

```bash
helm lint bootstrap-infra/
```

## Related Repositories

- [`rhpds/agnosticv`](https://github.com/rhpds/agnosticv) — Catalog items (`agd_v2/lightwell-infra-cluster`, `agd_v2/lightwell-infra-tenant`)
- [`rhpds/agnosticd-v2`](https://github.com/rhpds/agnosticd-v2) — Ansible deployer with workload roles
