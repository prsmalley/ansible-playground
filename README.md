# ansible-playground

Three Ansible playbooks plus the GitHub Actions deploy workflow for a
container deploy pipeline:

- `site.yml` — host config (packages, user, templated config, nginx)
- `bootstrap-k3s.yml` — installs k3s on a fresh EC2 host
- `deploy-flaskapp.yml` — pulls and runs the flaskapp container

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
├── inventory.ini.example       # Copy to inventory.ini and edit
├── inventory-runner.ini        # Used by the legacy Multipass deploy workflow
├── requirements.yml            # Ansible collections (community.docker)
├── site.yml                    # Host config playbook
├── bootstrap-k3s.yml           # k3s install playbook (one-time per cluster)
├── deploy-flaskapp.yml         # Container deploy playbook
├── templates/
│   └── config.yml.j2           # Used by site.yml
├── .github/workflows/
│   ├── lint.yml                # ansible-lint on every PR
│   └── deploy.yml              # Triggers ARC ephemeral runners
└── ARCHITECTURE.md             # Multi-repo design and future work
```

## Prerequisites

- Ansible installed locally.
- A target Ubuntu host reachable over SSH.
- A user on the target with sudo (the playbooks use `become`).

## Host config (`site.yml`)

```bash
git clone git@github.com:prsmalley/ansible-playground.git
cd ansible-playground

cp inventory.ini.example inventory.ini
# Edit inventory.ini to point at your host

ansible devboxes -m ping            # smoke test
ansible-playbook site.yml --check   # dry run
ansible-playbook site.yml           # real run
```

`site.yml` installs packages, creates a user, drops a templated config
file, and ensures nginx is running. Re-running on an unchanged host
does nothing (idempotent).

## Bootstrapping k3s (`bootstrap-k3s.yml`)

One-time setup against a fresh EC2 instance provisioned by
[terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra).

```bash
# After terraform apply, with the EC2 IP in inventory.ini under [appservers]
ansible-playbook bootstrap-k3s.yml
```

Installs k3s, sets up `~/.kube/config` for the `ubuntu` user, and configures
TLS SANs so remote `kubectl` from your laptop works.

After bootstrap: install ARC via Helm directly on the cluster (manual
one-time step). See [ARCHITECTURE.md](ARCHITECTURE.md) for the full
sequence.

## Deploying flaskapp-docker-practice (`deploy-flaskapp.yml`)

Pulls a container image from
[flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)'s
GHCR and runs it.

### Run locally

```bash
ansible-playbook deploy-flaskapp.yml \
  -e "ghcr_token=YOUR_PAT" \
  -e "image_tag=latest" \
  -e "app_port=80"
```

The PAT needs `read:packages` scope.

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
