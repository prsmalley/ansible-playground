# Bootstrap a new cluster

One-time procedure for spinning up a new k3s cluster + ARC + RBAC from scratch. After this, deploys are workflow-triggered via `gh workflow run deploy.yml` (see [README.md](README.md)).

Total time: ~30 minutes, assuming AWS, GitHub, and SSH credentials are already in place.

## Prerequisites

- Ansible installed locally (control machine).
- `kubectl` installed locally.
- Helm 3.8+ installed locally (OCI registry support is required for the ARC chart).
- A target Ubuntu host reachable over SSH with sudo (provisioned by [terraform-flaskapp-infra](https://github.com/prsmalley/terraform-flaskapp-infra)).
- A GitHub Personal Access Token (classic) with `read:packages` scope for pulling the container image from GHCR. Create one at https://github.com/settings/tokens.
- Permission to create a GitHub App on the account that owns this repo.

## Steps

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
# Edit inventory.ini: under [appservers], add a line like:
#   ec2-flaskapp ansible_host=<EC2_IP>

ansible-playbook bootstrap-k3s.yml
```

Installs k3s, sets `~/.kube/config` for the `ubuntu` user, and configures TLS SANs for remote `kubectl` access.

### 3. (Optional) Remote kubectl from your local machine

```bash
scp ubuntu@$EC2_IP:/home/ubuntu/.kube/config ~/.kube/k3s-aws-config
# Edit the server: line in that file to use $EC2_IP instead of 127.0.0.1
export KUBECONFIG=~/.kube/k3s-aws-config
kubectl get nodes
```

All remaining `kubectl` and `helm` commands assume this is set up. If you'd rather run them on the EC2 directly via SSH, skip this step.

### 4. Create all three namespaces

Create them up front. Later steps reference resources across these namespaces — in particular, `setup/rbac.yaml` creates a ServiceAccount in `arc-runners`, so that namespace must exist before step 6.

```bash
kubectl create namespace flaskapp
kubectl create namespace arc-systems
kubectl create namespace arc-runners
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

Creates a `deploy-runner` ServiceAccount in `arc-runners` and a Role in `flaskapp` (with a matching RoleBinding) granting create/update/delete on Deployments, Services, Ingresses, Pods, Secrets, and ConfigMaps in the `flaskapp` namespace.

### 7. Create a GitHub App for ARC

GitHub → Settings → Developer settings → GitHub Apps → New GitHub App.

- Permissions: Actions (read), Administration (read & write), Metadata (read)
- Webhook: disable
- Install on this repo only

Generate a private key (`.pem`) and save it locally. Note the App ID and the Installation ID after installing.

### 8. Create the ARC GitHub App secret

```bash
kubectl create secret generic pre-defined-secret \
  --namespace=arc-runners \
  --from-literal=github_app_id=<APP_ID> \
  --from-literal=github_app_installation_id=<INSTALLATION_ID> \
  --from-file=github_app_private_key=/path/to/app-private-key.pem
```

### 9. Install ARC via Helm

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

`serviceAccountName=deploy-runner` wires the runner pods to the RBAC from step 6.

### 10. Verify

```bash
kubectl get pods -n arc-systems
# Expected: controller + listener, both Running

kubectl get pods -n arc-runners
# Expected: empty — runner pods only spawn when a job is queued

kubectl auth can-i create deployments -n flaskapp \
  --as=system:serviceaccount:arc-runners:deploy-runner
# Expected: yes
```

Cluster is ready. Deploys from here on are workflow-triggered — see [README.md](README.md).
