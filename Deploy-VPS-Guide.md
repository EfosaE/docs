# Deploying Credora Backend to the VPS

Automated CI/CD for the Credora Go backend using **GitHub Actions**, **GHCR**, **Docker Compose** and **SSH**.

Every push to `main` runs the tests, builds one Docker image tagged with the commit SHA, publishes it to GHCR, and deploys that exact image to the VPS. The VPS never builds source code; it only pulls a prebuilt image.

---

## Overview

```
push to main
    │
    ▼
Unit tests (make test)
    │
    ▼
Integration tests (make test-integration, Testcontainers)
    │
    ▼
Build image ──► push ghcr.io/efosae/credora-backend:<commit-sha>
    │
    ▼
SSH to VPS ──► docker pull ──► update API + 3 workers
```

- Pull requests run both test suites but **never** publish or deploy.
- The API and all three Asynq workers run the **same image tag**.
- The API and workers share the **existing Redis container** over the Docker network `backend_app-network`.
- Production `.env`, `config.yaml`, Firebase credentials and persistent volumes stay on the VPS and are not touched.

---

## Repository files

| File | Purpose |
|---|---|
| `.github/workflows/deploy.yaml` | Combined test, build, publish and deploy workflow |
| `docker-compose.yml` | API (`app`), Redis, Prometheus, Grafana |
| `docker-compose.worker.yaml` | Asynq worker service (`worker`) |
| `Dockerfile` | Image build |

> Delete or disable the old `.github/workflows/test.yaml`. The tests now live in `deploy.yaml`, and keeping both runs them twice.

---

## VPS layout

Deployment directory: `/opt/credora/backend`

```
/opt/credora/backend/
├── docker-compose.yml
├── docker-compose.worker.yaml
├── .env                                   # production secrets (never overwritten)
├── config.yaml                            # production config (never overwritten)
└── credora-backend-firebase-adminsdk.json # Firebase credentials (never overwritten)
```

The pipeline only runs `docker pull` and `docker compose up -d`. It does not copy files over these.

---

## 1. Compose configuration

Both Compose files reference the GHCR image through the same expression.

**`docker-compose.yml`** (API service):

```yaml
image: ${CREDORA_IMAGE:-ghcr.io/efosae/credora-backend}:${IMAGE_TAG:-latest}
```

**`docker-compose.worker.yaml`**:

```yaml
services:
  worker:
    image: ${CREDORA_IMAGE:-ghcr.io/efosae/credora-backend}:${IMAGE_TAG:-latest}
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - app-network
    command: ./worker
    volumes:
      - ./config.yaml:/app/config.yaml
      - ./credora-backend-firebase-adminsdk.json:/app/firebase.json

networks:
  app-network:
    external: true
    name: backend_app-network
```

Notes:

- The worker file must **not** define its own Redis service. It uses the existing Redis from the API Compose project.
- The worker service must **not** set `container_name`, otherwise `--scale worker=3` fails.
- `IMAGE_TAG` is supplied at deploy time, so the production `.env` never needs to be edited per release.
- Keep any other settings your app needs (ports, healthchecks, volumes).

---

## 2. SSH access for GitHub Actions

Create a **dedicated** deployment key. Do not reuse your personal key.

### Generate the key (Windows PowerShell)

```powershell
ssh-keygen -t ed25519 -C "credora-github-actions" -f "$HOME\.ssh\credora_deploy"
```

Leave the passphrase empty, since the workflow runs non-interactively.

- `credora_deploy` is the private key, which goes into GitHub Secrets.
- `credora_deploy.pub` is the public key, which goes on the VPS.
- Never commit either file.

### Install the public key on the VPS

```powershell
Get-Content "$HOME\.ssh\credora_deploy.pub"
```

On the VPS, as the Linux user that GitHub Actions will use:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys     # paste the public key on a new line
chmod 600 ~/.ssh/authorized_keys
```

Test from your laptop:

```powershell
ssh -i "$HOME\.ssh\credora_deploy" YOUR_VPS_USER@YOUR_VPS_HOST
```

### Docker permission

The deployment user must be able to run Docker:

```bash
docker ps
```

Docker group membership is effectively root-equivalent. Use a trusted account and protect its key.

### Verify the host key

On the VPS:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On your laptop:

```powershell
ssh-keyscan -t ed25519 YOUR_VPS_HOST
```

Confirm the fingerprints match, then save the `ssh-keyscan` output as the `VPS_KNOWN_HOSTS` secret. Do **not** disable host-key checking.

---

## 3. GitHub repository secrets

Go to **Settings → Secrets and variables → Actions → New repository secret** and create:

| Secret | Value |
|---|---|
| `VPS_HOST` | VPS IP address or hostname |
| `VPS_USER` | Linux deployment username |
| `VPS_SSH_KEY` | Full contents of the private key `credora_deploy` |
| `VPS_KNOWN_HOSTS` | Verified `ssh-keyscan` line |

Also create a **`production`** environment under **Settings → Environments**. The deploy job targets it, so you can add required reviewers or restrict it to the `main` branch.

---

## 4. GHCR package visibility

A public repository does **not** make its GHCR package public.

After the first successful publish, open the `credora-backend` package on GitHub and choose one:

- **Public package** is the simplest, because the VPS pulls without authentication.
- **Private package**: create a Personal Access Token with only `read:packages` and log in on the VPS:

  ```bash
  echo "<TOKEN>" | docker login ghcr.io -u <github-username> --password-stdin
  ```

Never put a registry token in a Compose file or commit it to Git.

---

## 5. First deployment

```bash
git add .github/workflows/deploy.yaml docker-compose.yml docker-compose.worker.yaml
git commit -m "Add automated Credora deployment"
git push origin main
```

Open the **Actions** tab. You should see four jobs run in order: **Unit Tests**, **Integration Tests**, **Build and Publish Image**, **Deploy to VPS**.

### Verify on the VPS

```bash
docker ps

docker compose -f /opt/credora/backend/docker-compose.yml logs --tail=100 app
docker compose -f /opt/credora/backend/docker-compose.worker.yaml logs --tail=100 worker
```

Expected: the API container, three worker replicas, and the existing Redis container, all running the new image tag.

Check which tag is live:

```bash
docker ps --format '{{.Names}}\t{{.Image}}'
```

---

## 6. What the deploy step does

On the VPS, in `/opt/credora/backend`:

```bash
docker pull ghcr.io/efosae/credora-backend:$IMAGE_TAG
export IMAGE_TAG

docker compose -f docker-compose.yml up -d --no-deps app
docker compose -f docker-compose.worker.yaml up -d --no-deps --scale worker=3 worker
```

- `--no-deps` means Redis, Prometheus, Grafana and their volumes are not recreated.
- Only `app` and `worker` containers are replaced.

---

## 7. Manual rollback

Each release is tagged with its commit SHA, so rolling back means redeploying a previous tag. Find the previous SHA in the GitHub Actions history or the GHCR package page, then on the VPS:

```bash
cd /opt/credora/backend
export IMAGE_TAG=<previous-commit-sha>

docker pull ghcr.io/efosae/credora-backend:$IMAGE_TAG
docker compose -f docker-compose.yml up -d --no-deps app
docker compose -f docker-compose.worker.yaml up -d --no-deps --scale worker=3 worker
```

Alternatively, revert the bad commit on `main` and let the pipeline redeploy.

---

## 8. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `Host key verification failed` | `VPS_KNOWN_HOSTS` is missing or wrong. Re-run `ssh-keyscan -t ed25519 HOST` and update the secret. |
| `Permission denied (publickey)` | Public key not in `authorized_keys`, wrong `VPS_USER`, or the private key was pasted incompletely (include the BEGIN/END lines). |
| `permission denied` on `docker` | Deployment user is not in the `docker` group. |
| `pull access denied` / `unauthorized` | GHCR package is private and the VPS isn't logged in. Make it public or run `docker login ghcr.io`. |
| `go mod tidy -diff` fails | `go.mod` / `go.sum` are out of date. Run `go mod tidy` locally and commit. If your Go version lacks `-diff`, run `go mod tidy` and then `git diff --exit-code go.mod go.sum`. |
| Integration tests fail in CI | They must run without production credentials. Check Testcontainers can reach Docker on the runner. |
| Workers can't reach Redis | Worker file isn't on `backend_app-network`, or `REDIS_ADDR` in `.env` doesn't match the Redis service name. |
| `--scale worker=3` errors | `container_name` is set on the worker service, or a host port is published on it. |
| Orphan container warnings | Both Compose files share the project name `backend` (from the directory name). Do **not** add `--remove-orphans`, because it would remove services defined in the other file. |

---

## 9. Security notes

- The repository is public, so deployments must only run from trusted pushes to `main`. Pull requests never deploy.
- Protect `main` (require PRs and passing checks) and use the `production` environment's protection rules.
- Keep secrets only in GitHub Secrets and the VPS `.env`. Never commit them.
- The deployment SSH key is root-equivalent via Docker. Rotate it if exposed.

---

## 10. Known limitations and next steps

This pipeline is a working first pass. Before relying on it for unattended production deploys, add:

1. **Health-gated deploys:** after `up -d`, poll the API health endpoint and confirm workers started, failing the job if they don't.
2. **Automatic rollback:** record the previously running image tag before deploying, and redeploy it if the health check fails.
3. **Zero-downtime updates:** `up -d` briefly restarts the API container. A reverse proxy with health-aware routing or a blue/green setup would remove that gap.
