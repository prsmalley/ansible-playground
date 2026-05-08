# Architecture

A two-repo CI/CD setup that goes from a code push to a running container.

## Repos

- **[flaskapp-docker-practice](https://github.com/prsmalley/flaskapp-docker-practice)**
  — the app. Flask code, Dockerfile, CI, release pipeline.
- **[ansible-playground](https://github.com/prsmalley/ansible-playground)**
  (this repo) — the deploy. Host config playbook, deploy playbook, deploy
  workflow.

## Why two repos

Three reasons:

1. **Separation of concerns.** App code in one place, deploy logic in
   another. Each repo has one job.
2. **Independent change cadence.** App changes don't touch deploy YAML, and
   vice versa.
3. **Different security postures.** The app repo doesn't need deploy
   credentials. The deploy repo does. Splitting them keeps least-privilege
   easy.

## End-to-end flow

```
flaskapp-docker-practice
  push → CI (6 jobs in parallel) → merge → CI on main
                                              ↓
                                    workflow_run on success
                                              ↓
                                    release.yml: build + push to GHCR
                                              ↓
                                    post-publish-scan (Trivy on artifact)

ansible-playground
  operator → workflow_dispatch → self-hosted runner (in VM)
                                              ↓
                                    ansible-playbook deploy-flaskapp.yml
                                              ↓
                                    docker pull + run + /health check
```

## Step by step

1. Developer opens a PR. CI runs six jobs: ruff, hadolint, gitleaks,
   Semgrep, pytest, Trivy. All must pass.
2. PR merges to main (squash merge, branch protection enforces CI).
3. CI runs again on main. `workflow_run` fires `release.yml` on success.
4. Release builds multi-arch (amd64 + arm64), tags with SHA + branch +
   semver + latest, pushes to GHCR.
5. Post-publish scan pulls the SHA tag and re-scans the actual artifact.
6. Operator triggers `deploy.yml` in ansible-playground with an image tag.
7. Self-hosted runner inside the VM picks up the job over its outbound
   GitHub connection.
8. Playbook logs into GHCR, pulls the image, replaces the container,
   waits for `/health` to return 200.


## What a production version would add

### Deploy

- **Blue/green.** Run old + new in parallel, flip traffic via reverse
  proxy after health check. Sub-second rollback by flipping back.
- **Canary releases.** Route 1% of traffic first, monitor, ramp gradually.
- **Smoke tests beyond `/health`.** Real endpoints with real payloads.
- **SHA-only deploys in production.** Block `latest` as a production input.

### Rollback

- **`block:` / `rescue:` in the playbook.** Capture the running image SHA
  before deploying, redeploy on failure.



### Security

- **Image signing (cosign).** Sign on successful CI; verify at deploy
  time. Closes the gap between "scanned" and "delivered."
- **GitHub App instead of PAT.** Fine-grained permissions, auto-rotating
  credentials, audit logs tied to an app identity.
- **Ephemeral runners.** One job per fresh container/VM, destroyed after.
  GitHub's Actions Runner Controller (ARC) is the standard.


### Operations

- **Idempotent container deploy.** Compare running image digest to
  desired digest, skip recreate when they match.
- **Real secrets management.** HashiCorp Vault or AWS Secrets Manager
  with short-lived tokens.
- **Observability.** Logs to a log aggregator, Prometheus metrics,
  alerting on health-check and deploy failures.

### Branch protection

- **Required approvals on PRs** (off for solo work).
- **Force-push restrictions** on `feature/*` branches.
- **Signed commits required.**