# Deploying AI-Scan-Repos to K3s: the complete guide

This guide takes you from an empty server to the running app in a browser.
Follow it top to bottom. Every command says **where** it runs, **what** it
does, and **what you should see** when it worked.

It was written from a deployment that worked on K3S-SRV on 2026-09-30.

---

## 0. The big picture (read once)

```
Your browser (on the tailnet)
      │  http://100.111.216.122/
      ▼
Traefik Ingress (built into K3s) ──┬── /auth, /reviews, /providers, /webhook, /health ──▶ gateway ──▶ Postgres
                                   └── everything else ─────────────────────────────────▶ frontend (React UI)
gateway ──▶ review-api ──creates──▶ one scanner-worker Job per review
                 │                         │
                 └──── reports volume ◀────┘  (the Job writes the report, review-api reads it)
review-api and the Jobs ──▶ Service "lmstudio" ──▶ Tailscale ──▶ LM Studio on the laptop (100.93.244.39:1234)
```

What runs where:

| Where | What |
|---|---|
| **K3s server** (K3S-SRV, tailnet IP `100.111.216.122`) | K3s, Docker, and everything in the diagram except LM Studio |
| **Laptop** (Windows) | LM Studio with the model, Tailscale, your git checkout |
| **GitHub** | the code, and the OAuth App used for "Login with GitHub" |
| **Tailscale admin console** | the tailnet policy and the operator's OAuth client |

Everything the cluster runs lives in **one namespace: `ai-scan-repos`**.
The code and the Kubernetes files are split over six repositories in the
DevOpsCourse22 organization, all cloned side by side into **one folder**
(this guide uses `~/Desktop/Final-Project-Repos` on the server):

| Repository | What it holds |
|---|---|
| `ai-scanner-backend` | review-api, the review engine (`mcp-server/`), the scanners (`scanner/`) and the Job image (`scanner-worker/`) |
| `ai-scanner-gateway` | the gateway (login, sessions, settings) |
| `ai-scanner-frontend` | the React website |
| `ai-scanner-backend-deployment` | **this repository**: namespace, storage, Postgres, review-api, Ingress, Tailscale, test Jobs, the kustomization and this guide |
| `ai-scanner-gateway-deployment` | the gateway's manifests (Deployment, admins ConfigMap) |
| `ai-scanner-frontend-deployment` | the website's manifest |

Each `*-deployment` repository is **complete on its own** (its own
`kustomization.yaml` and its own image tag), because Argo CD deploys one
repository per Application: `ai-scan-backend`, `ai-scan-gateway` and
`ai-scan-frontend`. Without Argo CD, apply the three by hand with
`kubectl apply -k <repo>`, **backend first** (it creates the namespace).
Once Argo CD manages them, change the cluster only by committing to these
repositories.

### How to read the commands

- `🖥️ K3s server (bash)`: SSH into the server first: `ssh elad@100.111.216.122`.
- `💻 Laptop (PowerShell)` or `💻 Laptop (Git Bash)`: on your Windows laptop.
- `🌐 Browser`: a web page.
- Values in `<angle brackets>` are yours to fill in. **Never** paste a password,
  token or secret into a chat, a file in git, or a command line that is saved
  in history. This guide always reads them with a hidden prompt (`read -rsp`).

---

## Part A: One-time setup

Do Part A once per server. If the server already has something, the check
command tells you, and you skip that step.

### A1. Tools on the K3s server

🖥️ K3s server (bash). Check what is already installed:

```bash
k3s --version          # K3s itself
kubectl version --client
docker --version
docker buildx version  # needed to build the images
helm version           # needed for the Tailscale operator
git --version
openssl version        # used to generate passwords
tailscale ip -4        # must print 100.111.216.122: the address this guide uses everywhere
```

If `tailscale ip -4` prints a different address, use yours wherever this
guide says `100.111.216.122`, and change it in `ai-scanner-gateway-deployment/deployment.yaml`
(see "Changing the public address" at the end).

Each should print a version. Install only what is missing:

| Missing | Install command | Check |
|---|---|---|
| K3s | `curl -sfL https://get.k3s.io \| sh -` | `sudo k3s kubectl get nodes` shows the node `Ready` |
| Docker (+ buildx) | `curl -fsSL https://get.docker.com \| sh` then `sudo usermod -aG docker $USER`, then log out and back in | `docker run --rm hello-world` prints "Hello from Docker!" |
| Helm | `curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \| bash` | `helm version` |
| git, openssl | `sudo apt update && sudo apt install -y git openssl` | `git --version` |

### A2. Let `kubectl` talk to K3s without sudo

🖥️ K3s server (bash). K3s writes its admin kubeconfig to a root-only file.
Make a copy for your user:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
kubectl get nodes
```

✅ Worked if `kubectl get nodes` shows one node, `Ready`.

> ⚠️ This file is the cluster's admin key. Never commit it, never paste it
> anywhere.

### A3. Get the code on the server

🖥️ K3s server (bash):

```bash
mkdir -p ~/Desktop/Final-Project-Repos && cd ~/Desktop/Final-Project-Repos
for r in ai-scanner-backend ai-scanner-gateway ai-scanner-frontend \
         ai-scanner-backend-deployment ai-scanner-gateway-deployment ai-scanner-frontend-deployment; do
  [ -d "$r" ] || git clone "https://github.com/DevOpsCourse22/$r.git"   # skip the ones already there
  git -C "$r" pull --ff-only
done
```

The repositories are private, so `git clone` asks for a GitHub username and a
**personal access token** (GitHub no longer accepts your account password for
git). Type it at the prompt; never put it in a file or on the command line.

✅ Worked if `ls` shows the six folders, and
`ls ai-scanner-backend-deployment` lists `kustomization.yaml`, `README.md`,
`DEPLOY-K3S.md` and the folders `review-api`, `ingress`, `postgres`,
`storage`, `tailscale-operator`, `job-test`.

### A4. LM Studio on the laptop

💻 Laptop:

1. Open LM Studio and load **exactly one** model:
   `google/gemma-4-26b-a4b-qat`. Two loaded models make everything ~30× slower.
2. Developer tab → **Serve on Local Network** (so it listens on `0.0.0.0:1234`,
   not only localhost).
3. Tailscale tray icon → **Run unattended**, so the route survives closing
   the tray app.

💻 Laptop (PowerShell) check:

```powershell
netstat -ano | findstr :1234
```

✅ Worked if a line shows `0.0.0.0:1234 ... LISTENING`.

### A5. Tailscale: policy and OAuth client

🌐 Browser, https://login.tailscale.com/admin:

1. **Access controls → JSON editor**: paste the policy from
   `ai-scanner-backend-deployment/tailscale-operator/tailnet-policy.hujson` and save.
   It lets the operator tag its devices and lets `tag:k8s` reach LM Studio on
   tcp:1234.
2. **Settings → Trust credentials → Generate OAuth client**: give it `write` on
   General/Services, Devices/Core and Keys/Auth Keys, each with tag
   `tag:k8s-operator`. Keep the **client ID** and **client secret** for A6.

### A6. Install the Tailscale operator

This is what gives the cluster its private route to LM Studio.

🖥️ K3s server (bash). First the operator's credential (you paste the two
values from A5 at hidden prompts):

```bash
kubectl create namespace tailscale
read -rp  "Tailscale OAuth client ID: " TS_ID
read -rsp "Tailscale OAuth client secret: " TS_SECRET; echo
kubectl -n tailscale create secret generic operator-oauth \
  --from-literal=client_id="$TS_ID" --from-literal=client_secret="$TS_SECRET"
unset TS_ID TS_SECRET
```

✅ Prints `secret/operator-oauth created`.

Then the operator itself, with a pinned version:

```bash
cd ~/Desktop/Final-Project-Repos/ai-scanner-backend-deployment
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  -n tailscale --version 1.102.4 -f tailscale-operator/values.yaml --wait
```

✅ Worked if `kubectl -n tailscale get pods` shows `operator-…` as `Running`.

### A7. GitHub OAuth App (for "Login with GitHub")

🌐 Browser: GitHub → your avatar → **Settings → Developer settings → OAuth
Apps → New OAuth App**:

| Field | Value |
|---|---|
| Application name | `AI-Scan-Repos (K3s)` |
| Homepage URL | `http://100.111.216.122` |
| Authorization callback URL | `http://100.111.216.122/auth/github/callback` |

Click **Register application**, then **Generate a new client secret**.
Keep the **Client ID** and the **client secret** for step B4. GitHub shows
the secret only once; save it in a password manager.

> ⚠️ The callback URL must match `GITHUB_OAUTH_REDIRECT_URI` in
> `ai-scanner-gateway-deployment/deployment.yaml` **exactly**: `http`, not `https`, and no
> trailing slash.

---

## Part B: Deploy (the first time)

All of Part B: 🖥️ **K3s server (bash)**, starting in the folder that holds
the six repositories:

```bash
cd ~/Desktop/Final-Project-Repos
```

### B1. Build the four images

K3s can only run images that are in **its own** image store on this server
(there is no registry). So you build here, then hand the images to K3s.

Each image is tagged with the short commit of the repository it is built
from, so the three tags usually differ:

```bash
API_TAG=$(git -C ai-scanner-backend rev-parse --short HEAD)
GW_TAG=$(git -C ai-scanner-gateway rev-parse --short HEAD)
FE_TAG=$(git -C ai-scanner-frontend rev-parse --short HEAD)
echo "Building review-api:$API_TAG review-gateway:$GW_TAG review-frontend:$FE_TAG"

bash ai-scanner-backend/build.sh --tag "$API_TAG"               # review-api (slow: installs the scanners)
docker build -t "review-gateway:$GW_TAG" ai-scanner-gateway      # gateway
docker build -t "review-frontend:$FE_TAG" ai-scanner-frontend    # React UI
bash ai-scanner-backend/scanner-worker/build.sh --tag 0.5.0      # the Job image (slow)
```

✅ Worked if this lists 4 images:

```bash
docker images | grep -E 'review-|scanner-worker'
```

`review-api`, `review-gateway` and `review-frontend` tagged with their tags, and
`scanner-worker` tagged `0.5.0`.

> Why is scanner-worker always `0.5.0`? review-api reads that name from the
> env var `SCANNER_WORKER_IMAGE` in `ai-scanner-backend-deployment/review-api/deployment.yaml`. Kustomize
> does not rewrite env vars. When you change the worker's code, build a new
> number (`0.4.1`, ...) and change it in `review-api/deployment.yaml` and in
> `job-test/*.yaml`.

### B2. Hand the images to K3s

Docker and K3s keep **separate** image stores. This copies them over (type
your sudo password):

```bash
docker save "review-api:$API_TAG" "review-gateway:$GW_TAG" "review-frontend:$FE_TAG" scanner-worker:0.5.0 \
  | sudo k3s ctr images import -
```

✅ Worked if this shows the same 4 images, each starting with
`docker.io/library/` (that prefix is normal):

```bash
sudo k3s ctr images ls -q | grep -E 'review-|scanner-worker'
```

### B3. Point the manifests at the tag you built

Each deployment repository sets the tag of its own image:

```bash
sed -i "s/^    newTag: [^ ]*/    newTag: $API_TAG/" ai-scanner-backend-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $GW_TAG/"  ai-scanner-gateway-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $FE_TAG/"  ai-scanner-frontend-deployment/kustomization.yaml
grep -H newTag ai-scanner-*-deployment/kustomization.yaml
```

✅ Each file shows the tag of its own image.

> This edits a file on the server. Make the same change on your laptop and
> commit it there (`chore(k8s): point image tags at <tag>`). Before your next
> `git pull` on the server, undo the local copy with
> `git checkout kustomization.yaml`.

### B4. Create the namespace and the two Secrets

Secrets are **never** in git. You create them once, from the command line.

**1. The namespace** (the Secrets need it to exist):

```bash
kubectl apply -f namespace.yaml
```

✅ `namespace/ai-scan-repos created` (or `configured`/`unchanged`).

**2. Postgres credentials.** A random password goes into a variable. It is
never shown and never typed:

```bash
PG_PW=$(openssl rand -hex 24)
kubectl -n ai-scan-repos create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=ai_scan_repos \
  --from-literal=POSTGRES_PASSWORD="$PG_PW" \
  --from-literal=POSTGRES_DB=ai_scan_repos
```

✅ `secret/postgres-credentials created`.

> Hex characters only, on purpose: the password also goes inside a URL (next
> step), where `@ : / # ?` would break it.

**3. Gateway secrets.** In the **same terminal** (so `$PG_PW` is still set),
paste the OAuth values from A7 at the hidden prompts:

```bash
read -rsp "GitHub OAuth client ID: " GH_ID; echo
read -rsp "GitHub OAuth client secret: " GH_SECRET; echo
kubectl -n ai-scan-repos create secret generic gateway-secrets \
  --from-literal=database-url="postgresql+psycopg://ai_scan_repos:${PG_PW}@postgres:5432/ai_scan_repos" \
  --from-literal=github-oauth-client-id="$GH_ID" \
  --from-literal=github-oauth-client-secret="$GH_SECRET" \
  --from-literal=github-webhook-secret="$(openssl rand -hex 32)" \
  --from-literal=settings-key="$(openssl rand -hex 32)"
unset PG_PW GH_ID GH_SECRET
```

✅ `secret/gateway-secrets created`.

> Opened a new terminal and lost `$PG_PW`? Read it back from its Secret
> without printing it:
> `PG_PW=$(kubectl -n ai-scan-repos get secret postgres-credentials -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d)`

**4. Check both** (shows key names and sizes only, never values):

```bash
kubectl -n ai-scan-repos describe secret postgres-credentials gateway-secrets
```

✅ 3 keys + 5 keys, and **no key shows `0 bytes`**. If one does, delete that
Secret (`kubectl -n ai-scan-repos delete secret <name>`) and create it again.

### B5. Apply everything

```bash
cd ~/Desktop/Final-Project-Repos
kubectl kustomize ai-scanner-backend-deployment | less     # optional: read what will be applied (q to quit)
kubectl apply -k ai-scanner-backend-deployment             # first: it creates the namespace
kubectl apply -k ai-scanner-gateway-deployment
kubectl apply -k ai-scanner-frontend-deployment
```

With Argo CD (step 2 of PLATFORM-TOOLS-GUIDE.md), the three Applications
do this instead; press Sync on `ai-scan-backend` first.

✅ One line per object: `namespace/…`, `storageclass/…`, `service/…`,
`deployment.apps/…`, `statefulset.apps/…`, `ingress/…`, each `created`,
`configured` or `unchanged`.

### B6. Wait for the Pods

```bash
kubectl -n ai-scan-repos get pods -w
```

✅ After a minute or two, all four are `Running` and `1/1`:

```
frontend-…     1/1   Running
gateway-…      1/1   Running
postgres-0     1/1   Running
review-api-…   1/1   Running
```

Press **Ctrl+C** to stop watching. The gateway may restart once or twice while
Postgres starts. That's normal.

### B7. Check it end to end

🖥️ K3s server (bash):

```bash
# the Ingress routes to the gateway: prints {"status":"ok"}
curl -s http://100.111.216.122/health; echo

# the Ingress routes to the UI: prints the start of the HTML page
curl -s http://100.111.216.122/ | head -3

# the Tailscale route: EXTERNAL-IP is ts-lmstudio-….tailscale.svc.cluster.local, NOT "placeholder"
kubectl -n ai-scan-repos get svc lmstudio

# review-api can reach the model: prints the model ids
kubectl -n ai-scan-repos exec deploy/review-api -- python -c "import json,urllib.request as u; [print(m['id']) for m in json.load(u.urlopen('http://lmstudio:1234/v1/models',timeout=10))['data']]"

# review-api may create Jobs (yes) but may not read Pods (no)
kubectl auth can-i create jobs -n ai-scan-repos --as=system:serviceaccount:ai-scan-repos:review-api
kubectl auth can-i list pods   -n ai-scan-repos --as=system:serviceaccount:ai-scan-repos:review-api
```

🌐 Browser (on a device in the tailnet): open **http://100.111.216.122/**,
click **Login with GitHub**, approve, and start a **quick** review of a small
repository.

🖥️ K3s server (bash), while the review runs:

```bash
kubectl -n ai-scan-repos get jobs -w
```

✅ A Job appears, then shows `1/1` completions, and the report appears in the UI
**with scanner findings**, not only AI ones. That is the whole system working.

---

## Part C: Redeploy after a code change

🖥️ K3s server (bash):

```bash
cd ~/Desktop/Final-Project-Repos
for r in ai-scanner-*-deployment; do git -C "$r" checkout kustomization.yaml; done   # drop local tag edits from last time
for r in ai-scanner-*; do git -C "$r" pull --ff-only; done
```

Then rebuild **only the repository that changed**, import that image, and
point its tag at it (one row per image; the others keep their tags):

| You changed | Rebuild and import |
|---|---|
| `ai-scanner-backend` (review-api, `mcp-server/` or `scanner/`) | `API_TAG=$(git -C ai-scanner-backend rev-parse --short HEAD)`, `bash ai-scanner-backend/build.sh --tag "$API_TAG"`, `docker save "review-api:$API_TAG" \| sudo k3s ctr images import -` |
| `ai-scanner-backend/scanner-worker/`, `mcp-server/` or `scanner/` | a new number, e.g. `bash ai-scanner-backend/scanner-worker/build.sh --tag 0.5.1`, import it, and update `SCANNER_WORKER_IMAGE` in `review-api/deployment.yaml` |
| `ai-scanner-gateway` | `GW_TAG=$(git -C ai-scanner-gateway rev-parse --short HEAD)`, `docker build -t "review-gateway:$GW_TAG" ai-scanner-gateway`, `docker save "review-gateway:$GW_TAG" \| sudo k3s ctr images import -` |
| `ai-scanner-frontend` | `FE_TAG=$(git -C ai-scanner-frontend rev-parse --short HEAD)`, `docker build -t "review-frontend:$FE_TAG" ai-scanner-frontend`, `docker save "review-frontend:$FE_TAG" \| sudo k3s ctr images import -` |
| only YAML in a `*-deployment` repository | nothing to build: go straight to applying that repository |

Then set the tag in the deployment repository of each image you rebuilt
(the matching `sed` line from B3) and apply that repository, for example for
review-api:

```bash
sed -i "s/^    newTag: [^ ]*/    newTag: $API_TAG/" ai-scanner-backend-deployment/kustomization.yaml
kubectl kustomize ai-scanner-backend-deployment >/dev/null && kubectl apply -k ai-scanner-backend-deployment
kubectl -n ai-scan-repos get pods -w
```

With Argo CD: commit and push the `newTag` change instead, and Argo CD
applies it.

A new tag makes Kubernetes roll each Deployment to new Pods by itself.

> **A review is running?** Pause it on its page first, redeploy, then press
> Resume. review-api keeps its list of reviews in memory, so a new review-api
> Pod forgets a running one: its Job carries on and the report still lands,
> but the site can only show it as "running in a job this server no longer
> tracks", and it cannot be paused from there. A paused review survives a
> redeploy (and a server reboot) because its state lives on the reports
> volume, and Resume continues from the last checkpoint on the same commit.

**Restart without changing anything** (for example after changing a Secret):

```bash
kubectl -n ai-scan-repos rollout restart deployment
kubectl -n ai-scan-repos delete pod postgres-0     # the StatefulSet recreates it; data is kept
```

---

## Part D: When something goes wrong

First look: `kubectl -n ai-scan-repos get pods`. Then, for the Pod that is not
`Running 1/1`:

```bash
kubectl -n ai-scan-repos describe pod <pod-name> | tail -15   # the Events at the bottom say why
kubectl -n ai-scan-repos logs <pod-name> --tail=50            # what the app printed
```

| You see | Why | Fix |
|---|---|---|
| `ImagePullBackOff` / `ErrImagePull` | K3s doesn't have that exact image and tag, so it tried Docker Hub | Compare `kubectl -n ai-scan-repos get pods -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'` with `sudo k3s ctr images ls -q \| grep review`. Then redo B2 (import) or B3 (tag) and `kubectl apply -k .` |
| `CreateContainerConfigError` | a Secret it needs doesn't exist (`secret "…" not found` in Events) | Create it (B4). The Pod starts by itself |
| gateway log: `password authentication failed` | the password in `database-url` ≠ the Postgres password | Re-create `gateway-secrets` with the password read back from `postgres-credentials` (tip in B4), then `kubectl -n ai-scan-repos rollout restart deployment gateway` |
| gateway log: `No module named 'psycopg2'` | the database URL starts with `postgresql://` | It must start with `postgresql+psycopg://`. Re-create `gateway-secrets` |
| GitHub: `redirect_uri is not associated with this application` | the OAuth App's callback URL ≠ `GITHUB_OAUTH_REDIRECT_URI` | Make them identical (A7) |
| review-api `0/1` Ready | `/readyz` fails: `/reports` or `/tmp` not writable | `kubectl -n ai-scan-repos describe pod -l app=review-api` |
| `lmstudio` EXTERNAL-IP is `placeholder` | the Tailscale operator isn't running | `kubectl -n tailscale get pods`; `kubectl -n tailscale logs deploy/operator --tail=50` |
| Reviews fail with a connection error to the model | the laptop is asleep, offline, or LM Studio isn't serving on the network | Check A4 on the laptop |
| `violates PodSecurity "restricted"` | a Pod without the required security settings | The Event names the missing field. All our manifests already comply |
| Browser can't open `http://100.111.216.122/` | the device isn't on the tailnet, or Traefik isn't running | `tailscale status` on that device; `kubectl -n kube-system get pods \| grep traefik` |

> ⚠️ **Changing the Postgres password later is not enough on its own.**
> Postgres reads `POSTGRES_PASSWORD` only on its very first start, on an
> empty volume. To really change it you must also delete its volume, which
> wipes the database: `kubectl -n ai-scan-repos delete statefulset postgres`,
> then `kubectl -n ai-scan-repos delete pvc data-postgres-0`, then re-create
> the Secrets and `kubectl apply -k .`

---

## Part E: Remove the app

🖥️ K3s server (bash). This keeps K3s and the Tailscale operator:

```bash
kubectl delete namespace ai-scan-repos        # everything inside, including the Secrets
kubectl delete storageclass reports-noexec
kubectl get pv                                # reports volumes stay "Released" (reclaimPolicy: Retain)
kubectl delete pv <name>                      # for each one that belonged to ai-scan-repos
```

The report files stay on disk in `/var/lib/rancher/k3s/storage/`. To stay
reversible, move them aside instead of deleting them:
`mkdir -p ~/old-reports && sudo mv /var/lib/rancher/k3s/storage/<folder> ~/old-reports/`.

To deploy again: Part B, from B4 (the images are still in K3s).

---

## Appendix: what each file does

| File | What it creates |
|---|---|
| `kustomization.yaml` | the list of this repository's files below, and the review-api image tag. `kubectl apply -k .` (or Argo CD) reads it |
| `namespace.yaml` | namespace `ai-scan-repos`, with Pod Security `restricted` |
| `storage/storageclass-reports.yaml` | StorageClass `reports-noexec` |
| `storage/pvc-reports.yaml` | the reports volume, shared by review-api and every Job |
| `postgres/postgres.yaml` | Postgres (StatefulSet + headless Service) with its own 5 GiB volume |
| `review-api/rbac.yaml` | review-api's identity and permissions (create/delete Jobs and Secrets, read Job status) |
| `review-api/lmstudio-egress.yaml` | Service `lmstudio`: the Tailscale route to LM Studio |
| `review-api/deployment.yaml` | review-api (internal API; creates the Jobs) |
| ai-scanner-gateway-deployment: `admins-configmap.yaml` | ConfigMap `gateway-admins`: the GitHub usernames allowed to change the AI model in Settings |
| ai-scanner-gateway-deployment: `deployment.yaml` (+ its own `kustomization.yaml` with the review-gateway tag) | the gateway (login, sessions, forwards to review-api). **The public address is set here** |
| ai-scanner-frontend-deployment: `deployment.yaml` (+ its own `kustomization.yaml` with the review-frontend tag) | the React UI (nginx) |
| `ingress/ingress.yaml` | the Traefik routes from the browser to the gateway and the UI |
| `tailscale-operator/` | Helm values and the tailnet policy (used in A5 and A6, not by `apply -k`) |
| `job-test/` | 4 manual scanner-worker Jobs for testing (`kubectl apply -f`, not in `apply -k`) |

Every line of every manifest has a comment explaining its value.

### AI model and admins

Which AI service every review uses (LM Studio on your network, or a cloud
service such as OpenAI or Anthropic, with its API key) is chosen in the app,
under **Settings > AI model**. Only site admins can change it; everyone else
sees it read-only. Until an admin saves something, reviews use review-api's
defaults (`LLM_PROVIDER`, `LMSTUDIO_BASE_URL`, `LMSTUDIO_MODEL` in
`review-api/deployment.yaml`).

**Who is an admin** is the ConfigMap `gateway-admins`
(`admins-configmap.yaml` in ai-scanner-gateway-deployment), one GitHub username per line. The gateway
re-reads it by itself, so no restart is needed.

🖥️ K3s server (bash):

```bash
kubectl -n ai-scan-repos edit configmap gateway-admins
```

This opens the ConfigMap in an editor (`vi` by default; `KUBE_EDITOR=nano`
in front of the command to use nano). Add or remove names under `admins: |`,
indented like the existing one, save and quit.
✅ `configmap/gateway-admins edited`. Within about a minute, the new admin
reloads Settings and sees the AI model form instead of the read-only line.
To keep the change in git too, edit that `admins-configmap.yaml` the same
way (a later apply or Argo CD sync of ai-scanner-gateway-deployment puts the file's list back).

**Where API keys are stored.** The key typed in Settings is saved in Postgres,
encrypted with `settings-key` from the Secret `gateway-secrets`. The page
never shows it again (only its last four characters). If `gateway-secrets`
was created before this key existed, add it without touching the others:

```bash
kubectl -n ai-scan-repos patch secret gateway-secrets \
  -p "{\"stringData\":{\"settings-key\":\"$(openssl rand -hex 32)\"}}"
kubectl -n ai-scan-repos rollout restart deployment gateway
```

✅ `secret/gateway-secrets patched` and `deployment.apps/gateway restarted`.
The value is generated on the server and never printed. Without it, Settings
still works for LM Studio but says it "can't store API keys yet" for cloud
services. Changing `settings-key` later makes a saved API key unreadable: an
admin then pastes the key again in Settings.

The key reaches a review Job only through that review's own Secret (as
`LLM_API_KEY`, `docs/job-container-contract.md`), never through the Job's
environment as written in its spec, and is deleted with the review.

### Changing the public address

The app answers on any address that reaches Traefik, but the login needs one
fixed address in **four** places that must match exactly:

1. `GATEWAY_FRONTEND_URL` in `ai-scanner-gateway-deployment/deployment.yaml`
2. `GATEWAY_CORS_ORIGINS` in `ai-scanner-gateway-deployment/deployment.yaml`
3. `GITHUB_OAUTH_REDIRECT_URI` in `ai-scanner-gateway-deployment/deployment.yaml` (+ `/auth/github/callback`)
4. the OAuth App's callback URL on GitHub (A7)

With HTTPS, also set `GATEWAY_COOKIE_SECURE` to `"true"` and add `tls:` to the
Ingress.
