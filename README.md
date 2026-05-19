# ansible-playground

One active Ansible playbook (`bootstrap-k3s.yml`) plus K8s manifests
(`manifests/`) deployed via GitHub Actions. Two archived playbooks
in `legacy/`.

Part of a three-repo CI/CD/CD design — see [ARCHITECTURE.md](ARCHITECTURE.md):

- [flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)
  builds and publishes the container image.
- [terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra)
  provisions AWS EC2 infrastructure to deploy to.
- This repo (ansible-playground) bootstraps the cluster and deploys the app.

## Repo layout

```
.
├── ansible.cfg                 # Ansible defaults
├── inventory.ini.example       # Copy to inventory.ini for bootstrap
├── requirements.yml            # Ansible collections (community.docker — used by legacy/)
├── bootstrap-k3s.yml           # k3s install playbook (one-time per cluster)
├── manifests/                  # K8s manifests (Deployment, Service, Ingress)
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── legacy/                     # Archived from earlier architecture
│   ├── site.yml                # Multipass host config (Day 3 learning artifact)
│   ├── deploy-flaskapp.yml     # Ansible-based deploy (replaced by K8s)
│   ├── inventory-runner.ini    # Used by old workflow (when runner = target)
│   └── templates/
│       └── config.yml.j2
├── .github/workflows/
│   ├── lint.yml                # ansible-lint on every PR
│   └── deploy.yml              # ARC ephemeral runner runs kubectl apply
└── ARCHITECTURE.md
```

## Prerequisites

- Ansible installed locally.
- A target Ubuntu host reachable over SSH.
- A user on the target with sudo (the playbooks use `become`).

## Legacy (`legacy/`)

Three archived files from earlier architecture iterations. Kept for
portfolio depth and as Ansible examples; not part of the active deploy.

- `site.yml` — relic from learning Ansible. Configured my Multipass VM
  with packages, a user, a templated config, and nginx. Demonstrates
  templates, handlers, and multi-module orchestration.
- `deploy-flaskapp.yml` — Ansible-based deploy that pulled the flaskapp
  image and ran it via Docker on a single host. Replaced by the K8s
  deploy (`manifests/` + `kubectl apply`). Demonstrates the
  `community.docker` collection.
- `inventory-runner.ini` — inventory used by the old `deploy.yml`
  workflow when the runner was the deploy target (Multipass VM era).

## Bootstrapping k3s (`bootstrap-k3s.yml`)

One-time setup against a fresh EC2 instance provisioned by
[terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra). Be sure to add EC2 IP to inventory.ini.

```bash
# After terraform apply, with the EC2 IP in inventory.ini under [appservers]
ansible-playbook bootstrap-k3s.yml
```

Installs k3s, sets up `~/.kube/config` for the `ubuntu` user, and configures
TLS SANs so remote `kubectl` from your laptop works.

After bootstrap: install ARC via Helm directly on the cluster (manual
one-time step). See [ARCHITECTURE.md](ARCHITECTURE.md) for the full
sequence.


### Run via GitHub Actions

`.github/workflows/deploy.yml` triggers on `workflow_dispatch` with inputs
for image tag and dry-run toggle. Jobs run on `arc-runner-set` — ephemeral
GitHub Actions runner pods spawned by ARC inside the k3s cluster.

```bash
gh workflow run deploy.yml -f image_tag=sha-abc1234 -f dry_run=false
```

The `GHCR_DEPLOY_TOKEN` secret holds the PAT. See
[ARCHITECTURE.md](ARCHITECTURE.md) for the full design and known
limitations.

## Linting

`ansible-lint` runs on every PR via GitHub Actions. To run it locally:

```bash
pip install ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible-lint
```

## License

MIT — see [LICENSE](LICENSE).
