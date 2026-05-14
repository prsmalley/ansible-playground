# Architecture

A three-repo CI/CD/CD setup that goes from a code push to a container running
on a Kubernetes cluster on AWS, with ephemeral GitHub Actions runners
provisioned by ARC.

## Repos

- **[flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)**
  — the app. Flask code, Dockerfile, CI, release pipeline.
- **[terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra)**
  — the infrastructure. Terraform provisioning AWS EC2, security groups,
  key pairs.
- **[ansible-playground](https://github.com/prsmalley/ansible-playground)**
  (this repo) — the bootstrap + deploy.  k3s install, ARC
  setup, container deploy.

## Why three repos

1. **Separation of concerns.** App, infrastructure, and deploy each get
   their own repo with one responsibility.
2. **Independent change cadence.** Code, infra, and deploy logic evolve
   at different speeds.
3. **Different security postures.** Each repo gets only the credentials
   it actually needs.

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
       EC2 instance provisioned (bare Ubuntu, no user-data)
       Security group: SSH (22), HTTP (80), Kubernetes API (6443)

ansible-playground — bootstrap (one-time)
  operator → ansible-playbook bootstrap-k3s.yml
                ↓
       k3s installed on EC2
       Kubeconfig set up for the ubuntu user
                ↓
  operator → helm install ARC + scale set (manual, one-time)
                ↓
       arc-systems namespace: controller + listener
       arc-runners namespace: ready to spawn ephemeral runner pods

ansible-playground — deploy (per-release)
  operator → gh workflow run deploy.yml
                ↓
       ARC listener detects queued job
                ↓
       Fresh runner pod spawned in arc-runners
                ↓
       Workflow runs on pod, pod terminates
```

## Step by step

1. Developer opens a PR in flaskapp-docker-practice. CI runs six jobs:
   ruff, hadolint, gitleaks, Semgrep, pytest, Trivy. All must pass.
2. PR merges to main. CI runs again. On success, `workflow_run` fires
   release.yml.
3. Release builds multi-arch (amd64 + arm64), tags with SHA + branch +
   semver + latest, pushes to GHCR. Post-publish Trivy scan against the
   published artifact.
4. Operator runs `terraform apply` in terraform-flaskapp-infra. AWS
   provisions an EC2 instance, security group, and key pair.
5. Operator runs `ansible-playbook bootstrap-k3s.yml` from this repo
   against the new EC2. k3s gets installed and kubeconfig set up.
6. Operator installs ARC via Helm on the cluster (manual one-time step)
   and configures the runner scale set with a GitHub App for auth.
7. Operator triggers `deploy.yml` via `workflow_dispatch`. ARC's listener
   detects the queued job and spawns an ephemeral runner pod in
   `arc-runners`. The workflow runs on that pod and the pod terminates
   when done.

## Design decisions

**GHCR over Docker Hub.** Co-located with the source repo, free for public
packages, `GITHUB_TOKEN` works for push without setting up a PAT.

**SHA tags for production deploys.** Immutable. `latest` is for demos.

**Terraform for infrastructure, Ansible for configuration.** Declarative
on both sides. Terraform creates the instance; Ansible installs k3s and
ongoing config. Each tool does what it's best at; both are re-runnable
and idempotent.

**k3s instead of EKS.** Single-node lightweight Kubernetes that runs on a
single EC2 instance. Production would use EKS for managed control plane,
multi-AZ, IAM integration. k3s gives the same `kubectl` surface for a
fraction of the cost.

**ARC (GitHub Actions Runner Controller) for ephemeral runners.**
Self-hosted runners that run as Kubernetes pods, spawned on demand and
destroyed after each job. No long-lived runners, no idle resources.
Standard production pattern; replaces the older "long-lived VM with a
runner installed" approach.

**GitHub App for ARC auth.** Fine-grained per-repo permissions and
auto-rotating short-lived tokens. Better than a PAT for both security
and audit trail.

**Cross-repo image pulls via PAT.** The deploy workflow in
ansible-playground pulls images from flaskapp-docker-practice's GHCR
namespace; `GITHUB_TOKEN` is per-repo, so we use a classic PAT with
`read:packages` scope.

## Known limitations and next steps

**The ephemeral runner architecture is proven, but the current `deploy.yml`
playbook can't fully execute inside a runner pod.** The runner image is
minimal — no Ansible binary, no SSH keys, no Docker. Two paths to fix:

1. **Custom runner image with Ansible pre-installed**, plus SSH keys
   mounted from a Kubernetes Secret. The runner SSHes out to a target
   host the same way an operator would. Maintains the "deploy to a
   server" pattern.
2. **Refactor deploy to use `kubectl apply` against the same k3s
   cluster.** The flaskapp runs as a k8s Deployment inside the cluster;
   the runner uses `kubectl` to apply manifests. No SSH, no separate
   deploy target — the cluster is both the runner host and the
   workload host.

Path 2 is the planned next step as it matches real production patterns.

## What a production version would add

### Infrastructure

- **EKS instead of k3s.** Managed control plane, multi-AZ, IAM
  integration via IRSA.
- **Remote Terraform state** in S3 with DynamoDB locking. Local
  `terraform.tfstate` doesn't survive lost laptops and breaks team
  workflows.
- **Custom VPC** with private subnets for the cluster and an ALB in a
  public subnet.
- **IAM Role attached to the instance.** No static AWS credentials on
  the host; the instance assumes the role via the metadata service.
- **ALB + ACM cert + Route 53.** HTTPS on a real domain.
- **Bastion + AWS Session Manager** instead of inbound SSH.

### Deploy

- **Refactor to kubectl-apply pattern** (see "next steps" above).
- **Blue/green or canary** for safer rollouts.
- **Smoke tests beyond `/health`.** Real endpoints with real payloads.
- **SHA-only deploys in production.** Block `latest` as a production
  input.

### Security

- **Image signing (cosign).** Sign on successful CI; verify at deploy
  time. Closes the gap between "scanned" and "delivered."
- **Pod Security Standards** enforcement on the cluster (restricted
  profile).
- **NetworkPolicies** to limit pod-to-pod traffic.
- **Sealed Secrets or External Secrets Operator** for sensitive config
  in the cluster.

### Operations

- **Observability.** Logs to a log aggregator, Prometheus metrics,
  alerting on health-check and deploy failures. Grafana + Loki +
  Prometheus is the canonical OSS stack.
- **Real secrets management.** HashiCorp Vault or AWS Secrets Manager.
- **GitOps deploys via ArgoCD or Flux.** Cluster state defined in git,
  cluster reconciles itself instead of being pushed to.

### Branch protection

- **Required approvals on PRs** (off for solo work).
- **Force-push restrictions** on `feature/*` branches.
- **Signed commits required.**
