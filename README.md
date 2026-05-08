# ansible-playground

Two Ansible playbooks: one configures an Ubuntu host, the other deploys a
Flask container pulled from GHCR. Includes a GitHub Actions deploy workflow
that runs on a self-hosted runner inside the target VM.

Part of a multi-repo design for proof of concept for CI/CD setup going from code push to running container. Workflow explained in full in [ARCHITECTURE.md](ARCHITECTURE.md).

## Repo layout

```
.
├── ansible.cfg                 # Ansible defaults
├── inventory.ini.example       # Copy to inventory.ini and edit
├── inventory-runner.ini        # Used by the deploy workflow (localhost target)
├── requirements.yml            # Ansible collections (community.docker)
├── site.yml                    # Host config playbook
├── deploy-flaskapp.yml         # Flask container deploy playbook
├── templates/
│   └── config.yml.j2           # Used by site.yml
├── .github/workflows/
│   ├── lint.yml                # ansible-lint on every PR
│   └── deploy.yml              # Manual-trigger deploy via self-hosted runner
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

`site.yml` installs packages, creates a user, drops a templated config file,
and ensures nginx is running. Re-running on an unchanged host does nothing
— that's the idempotency.

The `--check` dry run may show errors. Some tasks depend on earlier ones
(e.g. setting ownership on a directory needs the user from the previous
task), and `--check` doesn't actually create anything. The real run handles
this fine.

## Deploying flaskapp-docker-practice (`deploy-flaskapp.yml`)

Pulls a container image from
[flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)'s
GHCR and runs it on a target host.

### Run locally

```bash
ansible-playbook deploy-flaskapp.yml \
  -e "ghcr_token=YOUR_PAT" \
  -e "image_tag=latest"
```

The PAT needs `read:packages` scope.

### Run via GitHub Actions

`.github/workflows/deploy.yml` triggers on `workflow_dispatch` (manual execution) with inputs for image tag (defaults to latest) and dry run toggle. The job
runs on a self-hosted runner inside the target VM, so it reaches the host
as `localhost` without any inbound network access.

Trigger from the CLI:

```bash
gh workflow run deploy.yml -f image_tag=sha-abc1234 -f dry_run=false
```

Or from the Actions tab in the GitHub UI.

The token is stored as the `GHCR_DEPLOY_TOKEN` repository secret. See
[ARCHITECTURE.md](ARCHITECTURE.md) for the full multi-repo design.

## Linting

`ansible-lint` runs on every PR via GitHub Actions. To run it locally:

```bash
pip install ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible-lint
```

## License

MIT — see [LICENSE](LICENSE).
