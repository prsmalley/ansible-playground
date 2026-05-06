# ansible-playground

A small Ansible practice project. Configures an Ubuntu host with packages, a
user, a directory, a templated config file, and nginx.

## Repo layout

```
.
├── ansible.cfg                 # Ansible defaults (inventory path, output style)
├── inventory.ini.example       # Copy to inventory.ini and edit for your host
├── site.yml                    # The playbook
├── templates/
│   └── config.yml.j2           # Template rendered to /opt/myapp/config.yml
├── .ansible-lint               # Lint settings
└── .github/workflows/lint.yml  # Runs ansible-lint on every PR
```

## Prerequisites

- Ansible installed locally.
- A target Ubuntu host you can reach over SSH.
- A user on the target with sudo (the playbook uses `become`).

## Run it

```bash
git clone git@github.com:prsmalley/ansible-playground.git
cd ansible-playground

cp inventory.ini.example inventory.ini
# Edit inventory.ini to point at your host

ansible devboxes -m ping            # smoke test
ansible-playbook site.yml --check   # dry run
ansible-playbook site.yml           # real run
```

The dry run may report errors. Some tasks depend on earlier ones (e.g. setting
ownership on a directory needs the user from the previous task), and `--check`
doesn't actually create anything. The real run handles this fine.

## What the playbook does

1. Updates the apt cache.
2. Installs `htop`, `jq`, `tree`.
3. Creates a user called `appuser`.
4. Creates `/opt/myapp/` owned by `appuser`.
5. Drops a config file at `/opt/myapp/config.yml` from a template that
   includes the host's hostname.
6. Installs nginx and makes sure it's running.

If the config file changes, a handler restarts nginx.

Re-running the playbook on an unchanged host does nothing proving idempotency.

## Linting

`ansible-lint` runs on every pull request via GitHub Actions. To run it
locally:

```bash
pip install ansible-lint
ansible-lint
```

## License

MIT — see [LICENSE](LICENSE).
