# GitHub to VPS: Deployment Playbook

A reusable, battle-tested guide for wiring any containerized app to a VPS with **GitHub Actions**, **GHCR**, **Docker Compose** and **SSH**.

It was written after the Credora backend setup, so every warning in it comes from something that actually went wrong. Work through it top to bottom for the next project and you should skip most of the pain.

---

## Contents

1. [The big picture](#1-the-big-picture)
2. [Placeholders used in this guide](#2-placeholders-used-in-this-guide)
3. [Phase 1: Prepare the VPS](#3-phase-1-prepare-the-vps)
4. [Phase 2: Make Compose registry-ready](#4-phase-2-make-compose-registry-ready)
5. [Phase 3: SSH keys and host key](#5-phase-3-ssh-keys-and-host-key)
6. [Phase 4: GitHub environment and secrets](#6-phase-4-github-environment-and-secrets)
7. [Phase 5: The workflow](#7-phase-5-the-workflow)
8. [Phase 6: GHCR visibility](#8-phase-6-ghcr-visibility)
9. [Phase 7: First deploy and verification](#9-phase-7-first-deploy-and-verification)
10. [Rollback](#10-rollback)
11. [Troubleshooting: real errors and real fixes](#11-troubleshooting-real-errors-and-real-fixes)
12. [Golden rules](#12-golden-rules)
13. [Printable checklist](#13-printable-checklist)

---

## 1. The big picture

```
 You push to main
        │
        ▼
 ┌──────────────┐   fail   ┌───────────────────────────┐
 │  Unit tests  │─────────►│ Pipeline stops, nothing   │
 └──────┬───────┘          │ is built or deployed      │
        ▼                  └───────────────────────────┘
 ┌──────────────────┐
 │ Integration tests│
 └──────┬───────────┘
        ▼
 ┌──────────────────────────────┐
 │ Build image, push to GHCR    │   ghcr.io/<gh-user>/<app>:<commit-sha>
 └──────┬───────────────────────┘
        ▼
 ┌──────────────────────────────┐
 │ SSH into VPS                 │
 │  docker pull <that exact tag>│
 │  docker compose up -d        │
 └──────────────────────────────┘
```

**Why it's built this way**

| Decision | Reason |
|---|---|
| Build on GitHub, not on the VPS | The VPS stays small and simple, and every deploy runs the exact image that passed the tests. |
| Tag images with the commit SHA | You can always tell what is running, and rollback is just "deploy an older tag". |
| Avoid `latest` as the deploy tag | `latest` moves, so it tells you nothing about what is deployed. |
| Secrets live in a GitHub *environment* | Only the deploy job can read them, and you can restrict it to `main`. This matters on a public repo. |
| Pull requests never deploy | PRs run tests only, so untrusted code never touches production. |
| Server config stays on the server | `.env`, `config.yaml` and credential files are never overwritten by a deploy. |

---

## 2. Placeholders used in this guide

Replace these everywhere as you go.

| Placeholder | Example from Credora | Meaning |
|---|---|---|
| `<app>` | `credora-backend` | Image and repo name |
| `<gh-user>` | `efosae` | GitHub account that owns the package (**lowercase** in image paths) |
| `<vps-ip>` | `169.58.67.36` | VPS address |
| `<deploy-user>` | `efosa` | Linux account GitHub will SSH in as |
| `<app-dir>` | `/opt/credora/backend` | Deployment directory on the VPS |
| `<env-name>` | `VPS-Deploy` | GitHub environment name |
| `<service>` | `app`, `worker` | Compose service names |

> Image paths must be lowercase. `ghcr.io/EfosaE/app` is rejected, `ghcr.io/efosae/app` works.

---

## 3. Phase 1: Prepare the VPS

### 3.1 Install Docker and Compose

```bash
docker --version
docker compose version
```

### 3.2 Create the deployment directory

```bash
sudo mkdir -p <app-dir>
sudo chown <deploy-user>:<deploy-user> <app-dir>
cd <app-dir>
```

### 3.3 Decide what stays on the server (and never in Git)

```
<app-dir>/
├── docker-compose.yml        # in the repo
├── .env                      # SERVER ONLY: production secrets
├── config.yaml               # SERVER ONLY (if used)
└── <credentials>.json        # SERVER ONLY: Firebase or other service credentials
```

The pipeline only pulls an image and runs `docker compose`, so these files are never touched. Confirm they are not tracked by Git:

```bash
git status --short     # .env etc. must NOT appear as modified/tracked
```

### 3.4 Let the deploy user run Docker

```bash
docker ps    # must work without sudo, as <deploy-user>
```

If it fails, add the user to the `docker` group and log in again.

> Docker group membership is effectively root. Use a dedicated deployment account where possible and guard its key carefully.

### 3.5 Lock down the firewall

Only publish ports you intend the world to reach.

```bash
sudo ufw status
```

Databases and caches (Redis `6379`, Postgres `5432`) should **not** be reachable from the internet. See [Phase 2](#4-phase-2-make-compose-registry-ready).

---

## 4. Phase 2: Make Compose registry-ready

### 4.1 Use a parameterized image

In every service that runs your app image:

```yaml
image: ${APP_IMAGE:-ghcr.io/<gh-user>/<app>}:${IMAGE_TAG:-latest}
```

The deploy supplies `IMAGE_TAG` at run time, so you never edit the server's `.env` per release.

### 4.2 Don't publish internal ports

```yaml
# Bad: exposes Redis on the VPS public IP
redis:
  ports:
    - "6379:6379"

# Good: reachable only over the Docker network
redis:
  networks:
    - app-network
```

Services on the same Docker network reach each other by service name (`redis:6379`). They don't need published ports.

> **Gotcha:** editing Compose does not change a container that is already running. If you remove `ports:` later, recreate that container:
> `docker compose up -d --no-deps redis`. Then verify with `docker ps --format '{{.Names}}\t{{.Ports}}'`. You want `6379/tcp`, **not** `0.0.0.0:6379->6379/tcp`.

### 4.3 Scaling services

If you scale with `--scale worker=3`:

- Do **not** set `container_name` on that service. Names must be unique, so scaling fails.
- Do not publish a fixed host port on it for the same reason.

### 4.4 Sharing one network or Redis across Compose files

```yaml
networks:
  app-network:
    external: true
    name: <existing-network-name>
```

Find the real name with `docker network ls`. Compose prefixes network names with the project name (the directory name by default), so `app-network` in `/opt/x/backend` is usually `backend_app-network`.

> Two Compose files in the same directory share one project name. Don't use `--remove-orphans`, because it will remove services defined in the *other* file.

### 4.5 Validate before you trust it

```bash
docker compose -f docker-compose.yml config -q        # silent = valid
grep -n "image:" docker-compose.yml
```

---

## 5. Phase 3: SSH keys and host key

You need **two** separate pieces of trust:

| Piece | Proves | Goes where |
|---|---|---|
| Deploy **key pair** | "GitHub is allowed to log in to my VPS" | Private key into a GitHub secret, public key on the VPS |
| **Host key** line | "this really is my VPS, not an impostor" | GitHub secret (`known_hosts` format) |

### 5.1 Generate a dedicated deploy key

On your laptop (PowerShell):

```powershell
ssh-keygen -t ed25519 -C "<app>-github-actions" -f "$HOME\.ssh\<app>_deploy"
```

Leave the passphrase **empty** (the workflow is non-interactive). This creates:

- `<app>_deploy`: **private**. Goes into GitHub. Never commit or paste it anywhere.
- `<app>_deploy.pub`: **public**. Goes on the VPS.

### 5.2 Install the public key on the VPS

```powershell
Get-Content "$HOME\.ssh\<app>_deploy.pub"
```

On the VPS, as `<deploy-user>`:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys     # paste the public key on a NEW line
chmod 600 ~/.ssh/authorized_keys
```

Test it passwordless from your laptop:

```powershell
ssh -i "$HOME\.ssh\<app>_deploy" <deploy-user>@<vps-ip>
```

### 5.3 Build and verify the `known_hosts` line

**Step 1: get the server's own fingerprint** (on the VPS, over a session you already trust):

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# 256 SHA256:xxxxxxxx... root@hostname (ED25519)
```

**Step 2: get the host key line.** Try the normal way first:

```powershell
ssh-keyscan -t ed25519 <vps-ip>
```

> **If you see `choose_kex: unsupported KEX method sntrup761x25519-sha512@openssh.com`:** your Windows OpenSSH is older than the server's. It's a client-side limitation, not a server problem. Skip `ssh-keyscan` and build the line yourself:
>
> On the VPS:
> ```bash
> cat /etc/ssh/ssh_host_ed25519_key.pub
> # ssh-ed25519 AAAAC3Nza... root@hostname
> ```
> Prefix the IP and drop the trailing comment, so the line becomes:
> ```
> <vps-ip> ssh-ed25519 AAAAC3Nza...
> ```
> Save it to a file on your laptop (for example `$HOME\vps_hostkey.txt`).

**Step 3: prove the line is right.**

```powershell
ssh-keygen -lf "$HOME\vps_hostkey.txt"
```

The `SHA256:...` it prints must equal the one from Step 1. If it doesn't, you copied something wrong. Redo Step 2.

**Step 4: copy secrets to the clipboard without hand-selecting text:**

```powershell
Get-Content "$HOME\.ssh\<app>_deploy" -Raw | Set-Clipboard    # private key
Get-Content "$HOME\vps_hostkey.txt" -Raw | Set-Clipboard      # host key line
```

---

## 6. Phase 4: GitHub environment and secrets

### 6.1 Create the environment

Repo **Settings → Environments → New environment**, named `<env-name>`.

Then configure:

- **Deployment branches and tags**: *Selected branches and tags*, add `main`. This is the key protection: no other branch can ever read the secrets.
- **Required reviewers** (optional): adds a manual Approve click before each deploy. Skip it if you want fully automatic deploys.

### 6.2 Add four environment secrets

**Environments → `<env-name>` → Environment secrets → Add environment secret**

| Secret name | Value |
|---|---|
| `VPS_HOST` | `<vps-ip>` only, with no `user@`, no port, no spaces |
| `VPS_USER` | `<deploy-user>` (the account whose `authorized_keys` holds the public key) |
| `VPS_SSH_KEY` | Entire private key including `-----BEGIN` / `-----END` lines |
| `VPS_KNOWN_HOSTS` | The single verified line `<vps-ip> ssh-ed25519 AAAA...` |

### 6.3 The most expensive typo in this whole setup

> **GitHub does not error when a workflow references a secret that doesn't exist.** It silently substitutes an empty string.
>
> In the Credora setup the secret was saved as `VPS_KNOWN_HOST` (no final **S**) while the workflow read `VPS_KNOWN_HOSTS`. The result was an empty `known_hosts` file and this error:
> ```
> No ED25519 host key is known for *** and you have requested strict checking.
> Host key verification failed.
> ```

Secret names are **case-sensitive** and must match the workflow character for character. You cannot rename a secret, so delete it and recreate it.

Other facts that save time:

- GitHub never displays an existing secret's value. A blank edit box proves nothing, and clicking **Update secret** on it overwrites the real value with nothing.
- Don't define the same secret at both repository and environment level.
- The environment name in the workflow must match exactly (`environment: <env-name>`). A mismatch silently creates a *new, empty* environment.

---

## 7. Phase 5: The workflow

Save as `.github/workflows/deploy.yaml`. This is a generic template. Adjust the test commands, Go version handling, ports and service names to fit your app.

```yaml
name: CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: deploy-production
  cancel-in-progress: false      # never kill a deploy halfway

env:
  IMAGE_NAME: ghcr.io/<gh-user>/<app>

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go mod tidy -diff          # fails if go.mod/go.sum aren't tidy
      - run: make test

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version-file: go.mod
          cache: true
      - run: go mod tidy -diff
      - env:
          TESTCONTAINERS_RYUK_DISABLED: true
        run: make test-integration

  build:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    if: github.event_name != 'pull_request'     # PRs never publish
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.image_tag }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        run: echo "image_tag=${GITHUB_SHA}" >> "$GITHUB_OUTPUT"
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.image_tag }}

  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name != 'pull_request'
    environment: <env-name>                      # must match GitHub exactly
    steps:
      - name: Configure SSH
        env:
          VPS_SSH_KEY: ${{ secrets.VPS_SSH_KEY }}
          VPS_KNOWN_HOSTS: ${{ secrets.VPS_KNOWN_HOSTS }}
        run: |
          mkdir -p ~/.ssh && chmod 700 ~/.ssh
          printf '%s\n' "$VPS_SSH_KEY"    > ~/.ssh/id_ed25519 && chmod 600 ~/.ssh/id_ed25519
          printf '%s\n' "$VPS_KNOWN_HOSTS" > ~/.ssh/known_hosts && chmod 600 ~/.ssh/known_hosts

      - name: Deploy
        env:
          VPS_HOST: ${{ secrets.VPS_HOST }}
          VPS_USER: ${{ secrets.VPS_USER }}
          IMAGE_TAG: ${{ needs.build.outputs.image_tag }}
        run: |
          ssh -i ~/.ssh/id_ed25519 \
              -o BatchMode=yes \
              -o StrictHostKeyChecking=yes \
              "$VPS_USER@$VPS_HOST" \
              "IMAGE_TAG='$IMAGE_TAG' bash -s" <<'REMOTE'
          set -eu
          cd <app-dir>

          docker pull "ghcr.io/<gh-user>/<app>:$IMAGE_TAG"
          export IMAGE_TAG

          docker compose -f docker-compose.yml up -d --no-deps <service>
          # scaled services example:
          # docker compose -f docker-compose.worker.yaml up -d --no-deps --scale worker=3 worker

          docker ps
          REMOTE
```

### Design notes

- `go mod tidy -diff` needs Go 1.23+. It fails the build if `go.mod` or `go.sum` are out of date. On older Go, use `go mod tidy` followed by `git diff --exit-code go.mod go.sum`.
- `StrictHostKeyChecking=yes` is the whole point of the host key secret. **Never** switch it to `no` to "make the error go away".
- `--no-deps` updates only the named service, so databases, caches and their volumes are never recreated.
- If your workflow file has a different name than your old test workflow, delete the old one or your tests will run twice.

### Optional: health-gated deploy

Add this to the remote script, right after `up -d`, to fail the deploy if the app doesn't come up. Change the port and path to match your health route:

```bash
ok=0
for i in $(seq 1 20); do
  if curl -fsS "http://127.0.0.1:<port>/<health-path>" >/dev/null; then ok=1; break; fi
  sleep 3
done
if [ "$ok" != 1 ]; then
  echo "Health check failed"
  docker compose -f docker-compose.yml logs --tail=100 <service>
  exit 1
fi
```

The check runs *on the VPS*, so the health endpoint doesn't need to be public. For services with no HTTP endpoint (workers), check `docker compose ps` shows them `running` and not restarting.

### Temporary debugging step for SSH problems

Drop this in the deploy job right after **Configure SSH**, and delete it once things work. It prints no secret material:

```yaml
      - name: Debug known_hosts (temporary)
        env:
          VPS_HOST: ${{ secrets.VPS_HOST }}
        run: |
          echo "lines: $(wc -l < ~/.ssh/known_hosts)"
          echo "fields per line:"; awk '{print NF, $2}' ~/.ssh/known_hosts
          echo "fingerprint:"; ssh-keygen -lf ~/.ssh/known_hosts || true
          ssh-keygen -F "$VPS_HOST" >/dev/null && echo "host: FOUND" || echo "host: NOT FOUND"
```

Healthy output is `fields per line: 3 ssh-ed25519`, the expected `SHA256:` fingerprint, and `host: FOUND`.

---

## 8. Phase 6: GHCR visibility

A **public repo does not make its container package public.** Packages have their own visibility, and the package doesn't exist until the first successful build.

Find it at: GitHub profile → **Packages** → `<app>` → **Package settings**, or directly:
`https://github.com/users/<gh-user>/packages/container/package/<app>`

Choose one:

- **Public**: the VPS pulls without logging in. Simplest, and fine if the image holds no secrets (config and credentials stay on the server).
- **Private**: create a Personal Access Token (**Settings → Developer settings → Personal access tokens**) with only `read:packages`, then on the VPS:
  ```bash
  echo "<TOKEN>" | docker login ghcr.io -u <gh-user> --password-stdin
  ```

Never put a registry token in Compose files or Git.

> **Expect the first deploy to fail at `docker pull`** if the package is private. Set visibility, then use **Re-run failed jobs**. The build is reused and only the deploy repeats.

---

## 9. Phase 7: First deploy and verification

### 9.1 Make the server a clean checkout *first*

The workflow doesn't copy Compose files to the server. If you run Compose from a Git clone on the VPS, keep that clone clean and in sync, or `git pull` will refuse:

```
error: Your local changes to the following files would be overwritten by merge
```

**Rule: change Compose files in the repo, then `git pull` on the server. Never hand-edit tracked files on the server.**

If you already have server-only edits:

```bash
cp docker-compose*.y*ml ~/compose-backup/     # safety copy
git --no-pager diff                           # see what's different
```

- Edits that already exist in the repo → `git checkout -- <file>` then `git pull`.
- Edits worth keeping → copy them into the repo from your laptop, commit and push, then `git checkout -- <file>` and `git pull` on the server.
- If you use `git stash pop` and hit a conflict, resolve the file, then `git add <file>` and `git reset -q` to clear the `UU` status.

Use `git --no-pager diff` so output doesn't drop you into a `less` pager (press `q` to leave one).

### 9.2 Push and watch

```bash
git add .
git commit -m "Add automated deployment"
git push origin main
```

Open the repo's **Actions** tab. Jobs run in order: unit tests, integration tests, build, deploy. A failed step stops everything after it, which is the gate working as intended.

### 9.3 Verify on the VPS

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'
docker compose -f <app-dir>/docker-compose.yml logs --tail=100 <service>
```

You should see:

- Your services running `ghcr.io/<gh-user>/<app>:<full-sha>`
- No unintended `0.0.0.0:` published ports (especially databases and caches)

### 9.4 Browse your images

Profile → **Packages** → `<app>`. Each successful build is one tag (a commit SHA). That list is your rollback menu.

---

## 10. Rollback

Every release is a SHA, so rolling back means redeploying an older tag:

```bash
cd <app-dir>
export IMAGE_TAG=<previous-commit-sha>

docker pull ghcr.io/<gh-user>/<app>:$IMAGE_TAG
docker compose -f docker-compose.yml up -d --no-deps <service>
```

Or revert the bad commit on `main` and let the pipeline redeploy. Do a rollback by hand once on a calm day, so you've already practiced it when you need it.

---

## 11. Troubleshooting: real errors and real fixes

| Symptom | Cause | Fix |
|---|---|---|
| `No ED25519 host key is known for *** and you have requested strict checking` | `VPS_KNOWN_HOSTS` is empty, usually because the secret is misnamed (`VPS_KNOWN_HOST`), in the wrong place, or malformed | Check the exact name, recreate it in the right environment, use the debug step |
| `choose_kex: unsupported KEX method sntrup761x25519-sha512@openssh.com` from `ssh-keyscan` | Old Windows OpenSSH client | Build the line from `/etc/ssh/ssh_host_ed25519_key.pub` as in section 5.3 |
| `Permission denied (publickey)` | Public key not in `authorized_keys`, wrong `VPS_USER`, or a truncated private key | Re-check all three; paste the whole key including BEGIN/END |
| `go mod tidy -diff` step fails | `go.mod` or `go.sum` drifted (old workflows hid this by running `go mod tidy`) | Run `go mod tidy` locally, then commit **both** files |
| `unauthorized` / `denied` / `pull access denied` | GHCR package is private and the VPS isn't logged in | Make it public, or `docker login ghcr.io` with a `read:packages` token |
| Deploy job can't see any secrets | `environment:` name mismatch, or secrets saved at the wrong level | Match the name exactly and put the secrets under that environment |
| `git pull` aborts with "local changes would be overwritten" | Server files were hand-edited | Section 9.1 |
| `UU` in `git status` after `stash pop` | Git still thinks a conflict is unresolved | `git add <file>` then `git reset -q` |
| Redis or DB reachable on the public IP | `ports:` publishes it, and the running container predates your fix | Remove `ports:`, then recreate with `up -d --no-deps <service>` and check `docker ps` |
| `--scale` errors | `container_name` or a fixed host port is set | Remove them from the scaled service |
| Orphan container warnings | Two Compose files share a project name | Ignore them. Do **not** use `--remove-orphans` |
| Build job skipped | A test job failed, or the run was a pull request | Fix the tests. PRs are meant to skip build and deploy |
| Image path rejected | Uppercase in the image name | Use all lowercase |

---

## 12. Golden rules

1. **Never disable host-key checking.** Fix the host key instead.
2. **Typos in secret names fail silently.** Verify names character by character, and keep the debug step handy.
3. **Environment names must match exactly**, in the workflow and in GitHub.
4. **Don't hand-edit tracked files on the server.** Change the repo, then pull.
5. **Don't publish database or cache ports.** Containers reach each other over the Docker network.
6. **Editing Compose doesn't change running containers.** Recreate the service and verify with `docker ps`.
7. **Deploy by commit SHA, never by `latest`.**
8. **Keep production secrets on the server and in the GitHub environment, never in Git or the image.**
9. **A green job isn't a healthy app.** Add a health check before trusting unattended deploys.
10. **Test rollback before you need it.**

---

## 13. Printable checklist

**VPS**
- [ ] Docker and Compose installed; `<deploy-user>` can run `docker ps`
- [ ] `<app-dir>` created and owned by `<deploy-user>`
- [ ] `.env`, config and credential files in place and untracked
- [ ] Firewall reviewed; no database or cache ports exposed
- [ ] Compose uses `${APP_IMAGE:-...}:${IMAGE_TAG:-latest}`; `config -q` passes
- [ ] Server repo clone is clean (`git status --short` is empty)

**SSH**
- [ ] Dedicated deploy key generated (empty passphrase)
- [ ] Public key in `authorized_keys`; passwordless login tested
- [ ] Host key line built; its `ssh-keygen -lf` fingerprint matches the VPS's

**GitHub**
- [ ] Environment `<env-name>` created; deployments limited to `main`
- [ ] Four secrets saved under the environment with **exact** names: `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, `VPS_KNOWN_HOSTS`
- [ ] Workflow has `environment: <env-name>`; old duplicate test workflow removed
- [ ] `go.mod` / `go.sum` committed tidy

**Go live**
- [ ] Push to `main`; all four jobs green
- [ ] GHCR package visibility set (or VPS logged in)
- [ ] `docker ps` shows the new SHA tag and no unintended published ports
- [ ] Health check added; rollback tried once
- [ ] Temporary debug step removed
