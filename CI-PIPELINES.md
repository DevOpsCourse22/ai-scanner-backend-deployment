# CI pipelines: Jenkins builds, Argo CD deploys

Every push to `main` of a code repository runs its Jenkins pipeline.
The images go to **our own registry on K3S-SRV** (Docker Registry), not to
Docker Hub or any public registry; on EKS they go to private ECR repositories. The
pipeline tests the code, builds the image, scans it, pushes it to the image
registry and writes the new image tag into the matching deployment
repository. Argo CD sees that commit and updates the cluster.

```
git push ──webhook──▶ Jenkins: Test ▶ Build ▶ Scan ▶ Push ──▶ registry
                                                   └─ Deploy: commit newTag ──▶ *-deployment repo ──▶ Argo CD ──▶ cluster
```

## The three pipelines, in plain words

| Pipeline (file) | This pipeline does | Using |
|---|---|---|
| **ai-scanner-backend** (`Jenkinsfile`) | Runs all backend tests (review-api, engine, scanner, scanner-worker, security-tests), builds **two** images (`review-api` and `scanner-worker`), scans them, pushes them, and points `ai-scanner-backend-deployment` at the new tag (the review-api image and the `SCANNER_WORKER_IMAGE` the Jobs use) | pytest, BuildKit, Trivy, git |
| **ai-scanner-gateway** (`Jenkinsfile`) | Runs the gateway tests, builds `review-gateway`, scans, pushes, points `ai-scanner-gateway-deployment` at the new tag | pytest, BuildKit, Trivy, git |
| **ai-scanner-frontend** (`Jenkinsfile`) | Installs, lints and builds the website (that is its test), builds `review-frontend`, scans, pushes, points `ai-scanner-frontend-deployment` at the new tag | npm, BuildKit, Trivy, git |

Each stage, in one line:

| Stage | Does | Using |
|---|---|---|
| Test | runs the tests; a failing test stops everything | `python:3.12` / `node:22` container |
| Build | builds the image from the repo's Dockerfile into a file (`<image>.tar`) | **BuildKit** (what `docker build` uses), rootless, no Docker daemon needed |
| Scan | checks the image for known vulnerabilities; a fixable **CRITICAL** one stops the pipeline, the full HIGH+CRITICAL report is saved with the build | **Trivy** |
| Push | uploads the image to the registry, tagged with the commit (`a1b2c3d`) — `main` only | BuildKit |
| Deploy | commits `newName`/`newTag` to the deployment repo — `main` only | git + a token |

Other branches and pull requests run Test, Build and Scan only: nothing is
pushed or deployed.

**One pipeline, two registries.** Nothing in the Jenkinsfiles names a
registry. Two Jenkins global variables decide where images go:

| | `REGISTRY_KIND` | `REGISTRY` | login |
|---|---|---|---|
| K3S-SRV (now) | `local` | `10.0.0.39:30500` | none (private network, plain HTTP) |
| EKS (later) | `ecr` | `<account>.dkr.ecr.<region>.amazonaws.com` | `aws ecr get-login-password`, a 12-hour token; on EKS the Jenkins Pod gets AWS access through IRSA, no keys stored |

Each pipeline runs in its own Pod (`ci/pod.yaml`): one container per tool,
deleted when the build ends.

## Files

| Where | What |
|---|---|
| `ai-scanner-backend/Jenkinsfile`, `ai-scanner-gateway/Jenkinsfile`, `ai-scanner-frontend/Jenkinsfile` | the pipelines |
| `<code repo>/ci/pod.yaml` | the build Pod: python or node, buildkit, trivy, aws, git |
| `<code repo>/ci/deploy-lib.sh` | the Deploy stage's helpers (update the tag, commit, push with retry) |
| `platform/registry/registry.yaml` (this repo) | the local registry: Namespace, 50Gi volume, Deployment, NodePort 30500 |
| `platform/registry/registries.yaml` (this repo) | tells K3s to pull from that registry over HTTP |
| `platform/jenkins/values.yaml` (this repo) | Helm values for Jenkins |
| `platform/jenkins/smee-client.yaml` (this repo) | the Pod that relays GitHub's webhooks from smee.io to Jenkins |

`platform/` is not in `kustomization.yaml`: Argo CD does not manage it, you
install it once.

## Setup, in order

All commands: 🖥️ K3s server (bash), in `~/Desktop/Final-Project-Repos`
(clone the six repos there first: DEPLOY-K3S.md, A3).

### 1. The local registry

```bash
kubectl apply -f ai-scanner-backend-deployment/platform/registry/registry.yaml
kubectl -n registry rollout status deploy/registry
curl -s http://10.0.0.39:30500/v2/_catalog
```

✅ `deployment "registry" successfully rolled out` and `{"repositories":[]}`.

Tell K3s to trust it (this is the one step that needs sudo; running Pods keep
running while K3s restarts):

```bash
sudo cp ai-scanner-backend-deployment/platform/registry/registries.yaml /etc/rancher/k3s/registries.yaml
sudo systemctl restart k3s
sleep 20; kubectl get nodes
```

✅ The node is `Ready`. **Do this before the first pipeline deploys**:
without it, Pods using a pushed image fail with `ImagePullBackOff`.

### 2. Jenkins

```bash
kubectl create namespace jenkins
kubectl -n jenkins create secret generic jenkins-admin \
  --from-literal=jenkins-admin-user=admin \
  --from-literal=jenkins-admin-password="$(openssl rand -base64 24)"
helm repo add jenkins https://charts.jenkins.io && helm repo update
helm upgrade --install jenkins jenkins/jenkins -n jenkins --version 5.9.68 \
  -f ai-scanner-backend-deployment/platform/jenkins/values.yaml --wait --timeout 10m
kubectl -n jenkins get pods
```

✅ `jenkins-0` is `2/2 Running`. Open `http://jenkins.100-111-216-122.sslip.io`
and log in as `admin` (password, read in your terminal only:
`kubectl -n jenkins get secret jenkins-admin -o jsonpath='{.data.jenkins-admin-password}' | base64 -d; echo`).

> Correction to my 2026-10-06 instructions: do **not** set
> `controller.admin.createSecret: false`. With an `existingSecret`, that flag
> is what makes the chart mount it; false leaves Jenkins without a password.

### 3. Two GitHub tokens, stored in Jenkins

**Why two, and not one per repository?** One fine-grained token can cover
several repositories (you tick them when you create it). The split is by
**permission**, so each token can do only what its job needs:

| Jenkins credential ID | Repositories | Can do | Used by |
|---|---|---|---|
| `github-ci` | the 3 code repos (`ai-scanner-backend`, `-gateway`, `-frontend`) | read the code; set the ✓/✗ build status on commits | Jenkins, to fetch the code and run the pipelines |
| `github-deploy` | the 3 `*-deployment` repos | push commits | the Deploy stage, to commit the new image tag |

The token Jenkins uses all the time cannot change anything; the one that can
push never reaches the source code.

**A. Create `github-ci`** (🌐 github.com, as Elad-Eytan):
https://github.com/settings/personal-access-tokens/new

- Token name `jenkins-ci-read`; Resource owner **DevOpsCourse22**;
  Expiration at least a month after the course (e.g. 2026-12-31).
- Repository access: **Only select repositories** > the 3 code repos.
- Repository permissions: **Contents: Read-only**, **Commit statuses: Read
  and write** (Metadata: Read-only is added by itself). Everything else: No access.
- **Generate token**, copy it (`github_pat_...`, shown once) straight into Jenkins (C).

**B. Create `github-deploy`**: the same form, token name `jenkins-deploy-write`,
the 3 `*-deployment` repos, **Contents: Read and write** only.

If a token shows **Pending**: DevOpsCourse22 > Settings > Personal access
tokens > Pending requests > approve (you are an owner).

**C. Store them in Jenkins**: Manage Jenkins > Credentials > **(global)** >
Add Credentials, twice:

| Field | First | Second |
|---|---|---|
| Kind | Username with password | Username with password |
| Scope | Global | Global |
| Username | `Elad-Eytan` | `Elad-Eytan` |
| Password | token A | token B |
| ID | `github-ci` | `github-deploy` |
| Description | GitHub read: code repos | GitHub write: deployment repos |

✅ The Credentials page lists both, passwords shown as `****`.

**D. Optional check** (🖥️ K3s server; the token is read at a hidden prompt
and never printed). GitHub answers what the token may do per repository:

```bash
read -rsp "Token: " T; echo; for r in ai-scanner-backend ai-scanner-gateway ai-scanner-frontend ai-scanner-backend-deployment ai-scanner-gateway-deployment ai-scanner-frontend-deployment; do printf '%-32s ' $r; curl -s -H "Authorization: Bearer $T" https://api.github.com/repos/DevOpsCourse22/$r | python3 -c "import json,sys; d=json.load(sys.stdin); p=d.get('permissions'); print('no access' if not p else ('WRITE' if p['push'] else 'read'))"; done; unset T
```

✅ `github-ci`: `read` on the 3 code repos, `no access` on the others.
`github-deploy`: `WRITE` on the 3 deployment repos, `no access` on the others.

### 4. The three jobs

**What a job is.** A Jenkins *job* is "a thing Jenkins can build". You make
one per code repository, of the type **Multibranch Pipeline**:

- It watches **one GitHub repository** (using the `github-ci` token).
- It looks at every branch (and pull request) and, for each one that has a
  `Jenkinsfile`, creates a sub-job for that branch automatically. So you
  never create "the main job" by hand: you get `ai-scanner-frontend » main`,
  and a branch `feature-x` you push later appears by itself.
- The **Jenkinsfile in the repository** is the pipeline. The job only says
  *where* the code is; *what* to do (Test, Build, Scan, Push, Deploy) comes
  from the file, so changes to the pipeline are reviewed and versioned like
  code.
- Each build runs in a fresh Pod in the `jenkins` namespace, made from the
  repo's `ci/pod.yaml`, deleted afterwards.

Push and Deploy run only for `main` (the Jenkinsfile says `when { branch 'main' }`);
other branches and pull requests are tested, built and scanned only.

**Create the first job** (🌐 `http://jenkins.100-111-216-122.sslip.io`, as admin):

1. Dashboard > **New Item**.
2. Name: `ai-scanner-backend` (the repository's name keeps things clear).
   Choose **Multibranch Pipeline** > **OK**.
3. **General**: Display Name empty (the name is enough); Description
   e.g. "review-api + scanner-worker: test, build, scan, push, deploy".
4. **Branch Sources** > **Add source** > **GitHub**:
   - **Credentials**: `Elad-Eytan/****** (GitHub read: code repos)`, i.e. `github-ci`.
     (Never `github-deploy` here: reading code needs only the read token.)
   - **Repository HTTPS URL**: `https://github.com/DevOpsCourse22/ai-scanner-backend.git`
   - Click **Validate**. ✅ "Credentials ok. Connected to https://github.com/DevOpsCourse22/ai-scanner-backend."
     If it says the repository was not found: the token does not include
     that repository, or it is still Pending.
   - **Behaviours**: the defaults are right, with one change: **delete
     "Discover pull requests from forks"** (the red X). A fork's pull
     request would run someone else's code with our credentials. The two
     that stay: "Discover branches" and "Discover pull requests from origin".
5. **Build Configuration**: Mode **by Jenkinsfile**, Script Path `Jenkinsfile`
   (the defaults).
6. **Scan Repository Triggers**: tick **Periodically if not otherwise run**,
   Interval **1 day**. The webhook (step 6) is what starts builds right away;
   this daily scan only catches anything a missed webhook skipped.
7. **Orphaned Item Strategy**: tick **Discard old items** (a deleted branch's
   sub-job is removed). Leave the numbers empty.
8. **Save**.

**What happens when you click Save.** Jenkins runs a **Scan Repository**:
it lists the branches, finds `Jenkinsfile` on `main`, creates the sub-job
`main` and starts build #1 at once. Follow it: click `main` > `#1` >
**Pipeline Overview** (the stage graph) or **Console Output** (the full log).

The first build is slow (5-15 minutes): the Pod pulls its images (python,
buildkit, trivy, aws-cli, git), BuildKit starts with an empty cache, and the
backend installs its test dependencies. Later builds are faster.

✅ When it is done:
- All five stages are green: Test, Build, Scan, Push, Deploy.
- Build #1 page > **Tests**: the number of tests passed.
- Build #1 page > **Build Artifacts**: `trivy-review-api.txt`,
  `trivy-scanner-worker.txt` (the scan reports).
- 🖥️ `curl -s http://10.0.0.39:30500/v2/_catalog` lists `review-api` and
  `scanner-worker`, and `curl -s http://10.0.0.39:30500/v2/review-api/tags/list`
  shows the commit as the tag.
- On GitHub, `ai-scanner-backend-deployment` has a new commit
  `ci(deploy): review-api and scanner-worker -> <tag>` by "Jenkins CI".
- The commit on `ai-scanner-backend` shows a green ✓ (the build status).

Until step 5 turns on automatic sync, Argo CD shows `ai-scan-backend` as
**OutOfSync** after that commit: it noticed, and waits for you.

**Repeat** for the other two, changing only the name, the description and
the URL:

| Job name | Repository HTTPS URL |
|---|---|
| `ai-scanner-gateway` | `https://github.com/DevOpsCourse22/ai-scanner-gateway.git` |
| `ai-scanner-frontend` | `https://github.com/DevOpsCourse22/ai-scanner-frontend.git` |

**If a build fails**: the stage that turned red is where to look. Its log
(click the stage) shows the last command and its error. Common first-run
causes: a test failure (Test), BuildKit not allowed to start (Build: the
log says `permission denied` or `operation not permitted`), the registry not
reachable (Push), a wrong or Pending `github-deploy` token (Deploy:
`Authentication failed`). Send the last 30 lines of the red stage's log.

### 5. Argo CD: deploy automatically

The Deploy stage only commits. For Argo CD to apply the commit by itself,
turn on automatic sync in each of the three Applications (`spec.syncPolicy`):
`automated:` with `prune: true` and `selfHeal: true` (keep backend's
`syncOptions`), then `kubectl apply -f` each Application file. Argo CD checks
git every 3 minutes.

✅ `argocd app list` shows `Auto-Prune` in SYNCPOLICY for all three.

### 6. The webhook: every push starts the pipeline

**Why.** Without it Jenkins does not know you pushed: it builds only when
you click Build Now or at its daily scan. A webhook is GitHub calling Jenkins
the moment you push. GitHub is on the internet and Jenkins only on the
tailnet, so the call goes through smee.io, a free relay:

```
git push -> GitHub --POST--> smee.io channel <--open connection-- smee-client Pod --POST--> Jenkins /github-webhook/
```

The Pod only connects out: nothing on the server is opened to the internet.

**6.1 A channel.** 🌐 Open https://smee.io/new . The page redirects to a new
address, `https://smee.io/` + about 16 random characters: that address in the
browser's address bar is your channel. (Not the text `smee.io/new` itself.)
Keep the page open: it shows each event live. Treat the URL like a password.

**6.2 Store it and start the client** (🖥️ the URL is read at a hidden prompt):

```bash
read -rsp "smee.io channel URL: " SMEE_URL; echo
kubectl -n jenkins create secret generic smee-channel --from-literal=url="$SMEE_URL"; unset SMEE_URL
kubectl apply -f ai-scanner-backend-deployment/platform/jenkins/smee-client.yaml
kubectl -n jenkins logs deploy/smee-client --tail=3
```

✅ `Connected to https://smee.io/<your channel>`. To replace a wrong URL:
`kubectl -n jenkins delete secret smee-channel`, create it again, then
`kubectl -n jenkins rollout restart deploy/smee-client` (the Pod reads the
Secret only when it starts).

**6.3 A webhook secret**, so Jenkins accepts only events GitHub really
signed (a leaked channel URL then cannot start builds):

```bash
openssl rand -hex 32 > ~/webhook-secret && chmod 600 ~/webhook-secret
cat ~/webhook-secret     # only when pasting it into the two forms below
```

- Jenkins: Manage Jenkins > Credentials > (global) > Add Credentials: Kind
  Secret text, Secret = the value, ID `github-webhook-secret`.
- Jenkins: Manage Jenkins > System > GitHub > Advanced > Shared secrets >
  Add > `github-webhook-secret` > Save.

**6.4 The GitHub webhook.** 🌐 github.com/DevOpsCourse22 > Settings (the
organization's) > Webhooks > Add webhook: Payload URL = the channel URL,
Content type `application/json`, Secret = the same value, SSL verification
enabled, events: "Let me select individual events" > Pushes and Pull
requests only, Active. One organization webhook covers every repository.
Then `rm ~/webhook-secret`.

**6.5 Check.** GitHub > the webhook > Recent Deliveries: the ping has a green
✓; the smee.io page shows it; `kubectl -n jenkins logs deploy/smee-client --tail=3`
shows `POST http://jenkins.jenkins.svc.cluster.local:8080/github-webhook/ - 200`.

## Demonstration

1. Change something visible, for example a word in
   `ai-scanner-frontend/src/...`, commit and push to `main`.
2. Within seconds Jenkins starts `ai-scanner-frontend » main` (the webhook).
3. The run goes green: Test, Build, Scan, Push, Deploy.
4. `ai-scanner-frontend-deployment` has a new commit
   `ci(deploy): review-frontend -> <tag>` by "Jenkins CI".
5. Within 3 minutes Argo CD shows `ai-scan-frontend` Syncing, then Synced to
   that commit; `kubectl -n ai-scan-repos get pods` shows a new frontend Pod.
6. Reload the site: the change is live.

## Moving to EKS + ECR

- Create one ECR repository per image (`review-api`, `scanner-worker`,
  `review-gateway`, `review-frontend`).
- In `platform/jenkins/values.yaml`, JCasC: `REGISTRY_KIND=ecr`,
  `REGISTRY=<account>.dkr.ecr.<region>.amazonaws.com`, add `AWS_REGION`.
- Give Jenkins's agent service account an IAM role (IRSA) that may push to
  those repositories. The Jenkinsfiles need no change.
- The registry and `registries.yaml` are not needed there; EKS nodes pull
  from ECR with their node role.
- The smee relay is not needed there either: Jenkins gets a public address
  (load balancer), and the GitHub webhook points straight at
  `https://<jenkins address>/github-webhook/`.

## Notes

- The first scan of `review-api` found no fixable CRITICAL issues, but 37
  fixable HIGH ones, all in the bundled `gitleaks` (33) and `trivy` (4)
  binaries (Go standard library and gRPC). Bumping those two versions in the
  Dockerfile clears them.
- Images are scanned before they are pushed: the Push stage rebuilds from
  BuildKit's cache (same inputs, so the same image) and only uploads.
