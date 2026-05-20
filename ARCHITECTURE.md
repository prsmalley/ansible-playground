# Architecture

A three-repo CI/CD/CD setup that goes from a code push to a container
running on a Kubernetes cluster on AWS, deployed by ephemeral self-hosted
GitHub Actions runners managed by ARC.

## Repos

- **[flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)**
  — the app. Flask code, Dockerfile, CI, release pipeline.
- **[terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra)**
  — the infrastructure. Terraform provisioning AWS EC2.
- **[ansible-playground](https://github.com/prsmalley/ansible-playground)**
  (this repo) — bootstrap + deploy. k3s install playbook, K8s manifests,
  ARC + workflow setup.

## Why three repos

1. **Separation of concerns.** App, infrastructure, and deploy each get
   their own repo with one responsibility.
2. **Independent change cadence.** Code, infra, and deploy logic evolve at
   different speeds.
3. **Different security postures.** Each repo gets only the credentials it
   actually needs.

## End-to-end flow

```
flaskapp-docker-practice
  push → CI (6 jobs) → merge → workflow_run gate → release.yml
                                                       ↓
                                          multi-arch build + push to GHCR
                                                       ↓
                                          post-publish Trivy scan

terraform-flaskapp-infra
  operator → terraform apply
                ↓
       Bare Ubuntu EC2 (no user_data)
       Security group: SSH, HTTP, k8s API

ansible-playground — initial cluster setup (one-time)
  operator → ansible-playbook bootstrap-k3s.yml
                ↓
       k3s installed; kubeconfig set up
                ↓
  operator → kubectl create namespace + secrets + apply setup/rbac.yaml
                ↓
  operator → helm install ARC (controller + scale set, with deploy-runner SA)
                ↓
       arc-systems: controller + listener (always running)
       arc-runners: ready to spawn ephemeral runner pods

ansible-playground — per-release deploy
  operator → gh workflow run deploy.yml
                ↓
       ARC listener detects queued job
                ↓
       Fresh runner pod spawned in arc-runners (using deploy-runner SA)
                ↓
       Pod runs: actions/checkout → kubectl apply -f manifests/
                ↓
       k3s reconciles: flaskapp pods come up in flaskapp namespace
                ↓
       Pod terminates after kubectl rollout status returns
```

## Step by step

1. Developer opens a PR in flaskapp-docker-practice. CI runs six jobs:
   ruff, hadolint, gitleaks, Semgrep, pytest, Trivy. All must pass.
2. PR merges to main. CI runs again. On success, `workflow_run` fires
   release.yml.
3. Release builds multi-arch (amd64 + arm64), tags with SHA + branch +
   semver + latest, pushes to GHCR. Post-publish Trivy scan against the
   published artifact.
4. Operator triggers `deploy.yml` via `workflow_dispatch`, providing the
   image tag.
5. ARC's listener detects the queued job. The controller spawns a fresh
   runner pod in `arc-runners`, using the `deploy-runner` ServiceAccount.
6. The runner pod git-clones ansible-playground, installs kubectl,
   `sed`-substitutes the image tag into the deployment manifest, then runs
   `kubectl apply -f manifests/`.
7. k3s reconciles to the declared state; flaskapp pods come up in the
   `flaskapp` namespace, become Ready when their readiness probes pass.
   Traefik (built into k3s) picks up the Ingress and exposes the app on
   port 80.
8. `kubectl rollout status` blocks until the new pods are Ready; the runner
   reports success and the pod terminates.

## Design decisions

**GHCR over Docker Hub.** Co-located with the source repo, free for public
packages, `GITHUB_TOKEN` works for push without setting up a PAT.

**SHA tags for production deploys.** Immutable. `latest` is for demos.

**Terraform for infrastructure, Ansible for configuration.** Declarative on
both sides. Terraform creates the instance; Ansible bootstraps k3s on top.
Each tool does what it's best at.

**k3s instead of EKS.** Single-node lightweight Kubernetes that runs on a
single EC2 instance. Production would use EKS for managed control plane,
multi-AZ, IAM integration. k3s gives the same `kubectl` surface for a
fraction of the cost.

**Ephemeral self-hosted runners via ARC.** Runners are Kubernetes pods,
spawned on demand and destroyed after each job. No long-lived runners, no
idle resources. Authentication via a GitHub App with fine-grained per-repo
permissions and auto-rotating tokens.

**K8s-native deploy via `kubectl apply`.** The deploy is a set of K8s
manifests (Deployment + Service + Ingress) applied by the runner pod inside
the same cluster it lives in. No SSH out to a separate target; no separate
deploy host. Eliminates the runner-vs-target distinction the previous
Ansible-over-SSH deploy had.

**Bootstrap state vs. deploy state — separated directories.** `setup/`
holds one-time cluster bootstrap (RBAC). `manifests/` holds per-deploy
state (Deployment, Service, Ingress). The workflow only re-applies
`manifests/`; never touches `setup/`. Prevents a `kubectl delete -f` of
the deploy state from cascading to the bootstrap state.

**GitHub App for ARC auth.** Fine-grained per-repo permissions and
auto-rotating short-lived tokens. Better than a PAT for both security and
audit trail.

**Cross-repo image pulls via PAT.** The runner pulls images from
flaskapp-docker-practice's GHCR namespace using a classic PAT (stored as
the `regcred` Secret in the `flaskapp` namespace). `GITHUB_TOKEN` is
per-repo, so cross-repo access needs a PAT (or, in production, a GitHub
App).

## What a production version would add

### Infrastructure

- **EKS instead of k3s.** Managed control plane, multi-AZ, IAM integration
  via IRSA.
- **Remote Terraform state** in S3 with DynamoDB locking.
- **Custom VPC** with private subnets for the cluster and an ALB in a
  public subnet.
- **IAM Role attached to the instance.** No static AWS credentials on the
  host.
- **ALB + ACM cert + Route 53.** HTTPS on a real domain.
- **Bastion + AWS Session Manager** instead of inbound SSH.
- **Packer-baked AMI** with k3s pre-installed, eliminating the manual
  Ansible bootstrap step.

### Deploy

- **Blue/green or canary** for safer rollouts.
- **Smoke tests beyond `/health`.** Real endpoints with real payloads.
- **SHA-only deploys in production.** Block `latest` as a production input.
- **GitOps via ArgoCD or Flux.** Cluster watches a manifests repo and
  reconciles itself; eliminates the push-based workflow entirely.
- **Kustomize or Helm chart** for the flaskapp manifests instead of `sed`
  substitution.

### Security

- **Image signing (cosign).** Sign on successful CI; verify at deploy time.
- **GitHub App for GHCR pulls** instead of a PAT. Could consolidate the
  ARC App's scope to cover packages access.
- **Pod Security Standards** enforcement on the cluster (restricted profile).
- **NetworkPolicies** to limit pod-to-pod traffic.
- **Sealed Secrets or External Secrets Operator** for sensitive config in
  the cluster.

### Operations

- **Observability.** Logs to a log aggregator, Prometheus metrics, alerts
  on health-check and deploy failures. Grafana + Loki + Prometheus is the
  canonical OSS stack.
- **HorizontalPodAutoscaler** on flaskapp for load-based scaling.
- **Real secrets management.** HashiCorp Vault or AWS Secrets Manager.

### Branch protection

- **Required approvals on PRs** (off for solo work).
- **Force-push restrictions** on `feature/*` branches.
- **Signed commits required.**
