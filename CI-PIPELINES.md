# CI pipelines: Jenkins builds, Argo CD deploys

Every push to `main` of a code repository runs its Jenkins pipeline. The
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
| `platform/jenkins/webhook-ingress.yaml` (this repo) | publishes ONLY `/github-webhook/` on the internet (Tailscale Funnel) |

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

Fine-grained tokens (github.com/settings/personal-access-tokens/new),
**Resource owner: DevOpsCourse22**:

| Jenkins credential ID | Repositories | Permissions | Used for |
|---|---|---|---|
| `github-ci` | the 3 code repos (`ai-scanner-backend`, `-gateway`, `-frontend`) | Contents: **Read-only**, Commit statuses: **Read and write** | Jenkins reads the code and shows ✓/✗ on commits |
| `github-deploy` | the 3 `*-deployment` repos | Contents: **Read and write** | the Deploy stage pushes the new tag |

In Jenkins: Manage Jenkins > Credentials > System > Global credentials >
**Add Credentials** > Kind **Username with password**: Username `Elad-Eytan`,
Password = the token, ID exactly as in the table. (Typed into Jenkins's own
form, so the token never touches a file or a chat.)

### 4. The three jobs

For each code repo: Jenkins > **New Item** > name `ai-scanner-backend` (then
`ai-scanner-gateway`, `ai-scanner-frontend`) > **Multibranch Pipeline** > OK.
Under **Branch Sources** > Add source > **GitHub**:
Credentials `github-ci`, Repository HTTPS URL
`https://github.com/DevOpsCourse22/<repo>.git`, **Validate** (✅ "Credentials
ok"). Build Configuration: by Jenkinsfile. **Save**.

✅ Jenkins scans the repo, finds `main`, and starts a first build. Watch it:
Test, Build, Scan, Push, Deploy turn green one by one. After the Push stage:
`curl -s http://10.0.0.39:30500/v2/_catalog` lists the image.

### 5. Argo CD: deploy automatically

The Deploy stage only commits. For Argo CD to apply the commit by itself,
turn on automatic sync in each of the three Applications (`spec.syncPolicy`):
`automated:` with `prune: true` and `selfHeal: true` (keep backend's
`syncOptions`), then `kubectl apply -f` each Application file. Argo CD checks
git every 3 minutes.

✅ `argocd app list` shows `Auto-Prune` in SYNCPOLICY for all three.

### 6. The webhook: every push starts the pipeline

GitHub is on the internet, K3S-SRV is not. Tailscale Funnel publishes only
Jenkins's webhook path:

1. 🌐 Tailscale admin console > **Access controls**: in `nodeAttrs`, add
   `"tag:k8s"` to the Funnel rule's `target` (it lists only
   `autogroup:member` today). Then **DNS** > **HTTPS Certificates** > Enable
   (if it isn't). Copy the policy back into `tailscale-operator/tailnet-policy.hujson`.
2. 🖥️ `kubectl apply -f ai-scanner-backend-deployment/platform/jenkins/webhook-ingress.yaml`
   then `kubectl -n jenkins get ingress jenkins-github-webhook` until ADDRESS
   shows `jenkins-webhook.tail3228b6.ts.net` (a minute or two).
   ✅ From any machine, even off the tailnet:
   `curl -s -o /dev/null -w '%{http_code}\n' https://jenkins-webhook.tail3228b6.ts.net/github-webhook/`
   prints `405` or `200` (Jenkins answering a GET), and
   `https://jenkins-webhook.tail3228b6.ts.net/login` prints `404` (the UI is NOT exposed).
3. 🌐 GitHub > **DevOpsCourse22** organization > Settings > **Webhooks** > Add
   webhook (one webhook for all repos):
   Payload URL `https://jenkins-webhook.tail3228b6.ts.net/github-webhook/`,
   Content type `application/json`, events **Pushes** and **Pull requests**.
   ✅ "Recent Deliveries" shows a green ✓ for the ping.

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

## Notes

- The first scan of `review-api` found no fixable CRITICAL issues, but 37
  fixable HIGH ones, all in the bundled `gitleaks` (33) and `trivy` (4)
  binaries (Go standard library and gRPC). Bumping those two versions in the
  Dockerfile clears them.
- Images are scanned before they are pushed: the Push stage rebuilds from
  BuildKit's cache (same inputs, so the same image) and only uploads.
