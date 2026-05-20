# ansible-playground

One active Ansible playbook (`bootstrap-k3s.yml`) plus K8s manifests
(`manifests/`) deployed via GitHub Actions on **ephemeral self-hosted
runners managed by ARC**. Two archived playbooks in `legacy/`.

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
├── manifests/                  # K8s manifests applied per deploy
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
├── setup/                      # One-time cluster bootstrap (not redeployed)
│   └── rbac.yaml               # ServiceAccount + Role + RoleBinding for deploy-runner
├── legacy/                     # Archived from earlier architecture
│   ├── site.yml
│   ├── deploy-flaskapp.yml
│   ├── inventory-runner.ini
│   └── templates/
│       └── config.yml.j2
├── .github/workflows/
│   ├── lint.yml                # ansible-lint on every PR
│   └── deploy.yml              # ARC ephemeral runner runs kubectl apply
└── ARCHITECTURE.md
```

## Prerequisites

- Ansible installed locally (control machine).
- Helm installed locally OR on the bootstrapped EC2.
- A target Ubuntu host reachable over SSH with sudo.
- A GitHub PAT with `read:packages` for pulling the container image from GHCR.
- A GitHub App for ARC authentication (see step 7 below).

## Initial cluster setup

One-time procedure for spinning up a new cluster from scratch. After this,
deploys are just `gh workflow run deploy.yml`.

### 1. Provision the EC2

In [terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra):

```bash
terraform apply
EC2_IP=$(terraform output -raw public_ip)
```

### 2. Bootstrap k3s on the EC2

In this repo:

```bash
cp inventory.ini.example inventory.ini
# Edit inventory.ini: set ansible_host=<EC2_IP> under [appservers]

ansible-playbook bootstrap-k3s.yml
```

Installs k3s, sets up `~/.kube/config` for the `ubuntu` user, configures TLS
SANs for remote `kubectl` access.

### 3. (Optional) Remote kubectl from your local machine

```bash
scp ubuntu@$EC2_IP:/home/ubuntu/.kube/config ~/.kube/k3s-aws-config
# Edit the server: line to use the EC2 public IP, not 127.0.0.1
export KUBECONFIG=~/.kube/k3s-aws-config
kubectl get nodes
```

### 4. Create the application namespace

```bash
kubectl create namespace flaskapp
```

### 5. Create the image pull secret

So k3s can pull the private GHCR image:

```bash
kubectl create secret docker-registry regcred \
  --namespace=flaskapp \
  --docker-server=ghcr.io \
  --docker-username=prsmalley \
  --docker-password=YOUR_GHCR_PAT \
  --docker-email=prsmalley@gmail.com
```

### 6. Apply the deploy-runner RBAC

```bash
kubectl apply -f setup/rbac.yaml
```

Creates a `deploy-runner` ServiceAccount in `arc-runners` with a Role
granting it create/update/delete on Deployments, Services, Ingresses, Pods,
Secrets, and ConfigMaps in the `flaskapp` namespace.

### 7. Create a GitHub App for ARC

GitHub → Settings → Developer settings → GitHub Apps → New GitHub App.

- Permissions: Actions (read), Administration (read & write), Metadata (read)
- Webhook: disable
- Install on this repo only

Generate a private key (`.pem`) and save it locally. Note the App ID and the
Installation ID after installing.

### 8. Create the ARC namespaces

```bash
kubectl create namespace arc-systems
kubectl create namespace arc-runners
```

### 9. Create the ARC GitHub App secret

```bash
kubectl create secret generic pre-defined-secret \
  --namespace=arc-runners \
  --from-literal=github_app_id=<APP_ID> \
  --from-literal=github_app_installation_id=<INSTALLATION_ID> \
  --from-file=github_app_private_key=/path/to/app-private-key.pem
```

### 10. Install ARC via Helm

The controller (one per cluster):

```bash
helm install arc \
  --namespace arc-systems \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
```

The runner scale set (one per repo):

```bash
helm install arc-runner-set \
  --namespace arc-runners \
  --set githubConfigUrl=https://github.com/prsmalley/ansible-playground \
  --set githubConfigSecret=pre-defined-secret \
  --set template.spec.serviceAccountName=deploy-runner \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

`serviceAccountName=deploy-runner` is what wires the runner pods to the RBAC
from step 6.

### 11. Verify

```bash
kubectl get pods -n arc-systems
# Expected: controller + listener, both Running

kubectl get pods -n arc-runners
# Expected: empty — runner pods only spawn when a job is queued

kubectl auth can-i create deployments -n flaskapp \
  --as=system:serviceaccount:arc-runners:deploy-runner
# Expected: yes
```

Cluster is ready. Deploys from here on are workflow-triggered.

## Deploying (`manifests/` + `kubectl apply`)

The active deploy chain:

1. `release.yml` in flaskapp-docker-practice publishes a multi-arch OCI image
   to GHCR.
2. Operator triggers `deploy.yml` via `workflow_dispatch`.
3. ARC's listener detects the queued job and spawns an **ephemeral
   self-hosted runner pod** inside the k3s cluster.
4. The runner pod authenticates to the K8s API via its mounted ServiceAccount
   token, runs `kubectl apply -f manifests/`.
5. k3s reconciles to the declared state; flaskapp pods come up.
6. The runner pod terminates.

Trigger from the CLI:

```bash
gh workflow run deploy.yml \
  -f image_tag=sha-abc1234 \
  -f dry_run=false
```

Or from the Actions tab in the GitHub UI.

The `image_tag` input gets `sed`-substituted into `manifests/deployment.yaml`
before apply, so you can deploy a specific SHA without editing files.

## Legacy (`legacy/`)

Three archived files from earlier architecture iterations. Kept for portfolio
depth and as Ansible examples; not part of the active deploy.

- `site.yml` — relic from learning Ansible. Configured my Multipass VM with
  packages, a user, a templated config, and nginx. Demonstrates templates,
  handlers, and multi-module orchestration.
- `deploy-flaskapp.yml` — Ansible-based deploy that pulled the flaskapp image
  and ran it via Docker on a single host. Replaced by the K8s deploy.
  Demonstrates the `community.docker` collection.
- `inventory-runner.ini` — inventory used by the old `deploy.yml` workflow
  when the runner was the deploy target (Multipass VM era).

## Linting

`ansible-lint` runs on every PR via GitHub Actions. To run it locally:

```bash
pip install ansible-lint
ansible-galaxy collection install -r requirements.yml
ansible-lint
```

## License

MIT — see [LICENSE](LICENSE).
