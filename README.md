# AI-Scan-Repos on Kubernetes

Every manifest for the cluster lives in this repository and its two siblings
(ai-scanner-gateway-deployment, ai-scanner-frontend-deployment). Each of the
three is complete on its own, with its own `kustomization.yaml` and image
tag, so Argo CD can deploy each as one Application. Everything runs in
**one namespace, `ai-scan-repos`**. (Until 2026-09-30 there were two folders:
`k8s/` in `ai-scan-repos` and `K8S-Tailcale-Egress/` in `code-review`. They
were merged here. The scanner Jobs must mount the reports PVC, and a PVC can
only be mounted from its own namespace.)

```
browser ──▶ Ingress (Traefik) ──┬─▶ frontend (React UI, everything else)
                                └─▶ gateway (/auth, /reviews, /providers, /webhook, /health) ──▶ review-api ──creates──▶ scanner-worker Job (one per review)
                                              │             │                        │
                                           Postgres      reports PVC ◀───────────────┘ writes report / error / progress
                                                            │
                                  review-api and Jobs ──▶ Service "lmstudio" ──▶ Tailscale egress proxy ──▶ LM Studio (100.93.244.39:1234)
```

## Layout

| Path | What it is | Applied by |
|---|---|---|
| `kustomization.yaml` | the list of files below and the review-api image tag | `kubectl apply -k` / Argo CD |
| `namespace.yaml` | the `ai-scan-repos` namespace, Pod Security `restricted` | `apply -k` |
| `storage/storageclass-reports.yaml` | StorageClass `reports-noexec` (local-path) | `apply -k` |
| `storage/pvc-reports.yaml` | PVC `ai-scan-repos-reports`, shared by review-api and every Job | `apply -k` |
| `postgres/postgres.yaml` | Postgres StatefulSet and headless Service | `apply -k` |
| `review-api/rbac.yaml` | ServiceAccount + Role: create/delete Jobs and Secrets, read Job status | `apply -k` |
| `review-api/lmstudio-egress.yaml` | Service `lmstudio`, the Tailscale route to LM Studio | `apply -k` |
| `review-api/deployment.yaml` | review-api Deployment + Service, kubernetes mode | `apply -k` |
| (repo ai-scanner-gateway-deployment) | gateway admins ConfigMap, Deployment + Service, review-gateway tag. **The public address is set here** (3 env vars) | its own `apply -k` / Argo CD |
| (repo ai-scanner-frontend-deployment) | the React UI (nginx, uid 101) Deployment + Service, review-frontend tag | its own `apply -k` / Argo CD |
| `ingress/ingress.yaml` | Traefik routes: the gateway's paths to the gateway, everything else to the UI | `apply -k` |
| `tailscale-operator/values.yaml` | Helm values for the Tailscale operator | `helm` (step 4) |
| `tailscale-operator/tailnet-policy.hujson` | the tailnet policy, as a record | the Tailscale admin console |
| `job-test/*.yaml` | 4 manual scanner-worker Jobs, one per outcome | `kubectl apply -f`, by hand |

Every line of every manifest has a comment that says what the value means.

**New here? Follow [`DEPLOY-K3S.md`](DEPLOY-K3S.md)**: the complete step-by-step deploy guide, from an empty server to the app in a browser.

Background: [`docs/21`](https://github.com/hadsahar/DevSecOps-Final-Project/blob/main/docs/21-kubernetes-manifests-guide.md) (writing
the manifests), [`docs/20` §6](https://github.com/hadsahar/DevSecOps-Final-Project/blob/main/docs/20-phase-3-plan.md) (Tailscale inside
Kubernetes), [`docs/job-container-contract.md`](https://github.com/hadsahar/DevSecOps-Final-Project/blob/main/docs/job-container-contract.md)
(what a Job gets and returns).

---

## Before the first deploy (once)

**On the model host (the laptop running LM Studio)**

- LM Studio's server listens on the network, not just localhost: Developer tab
  → **Serve on Local Network**, or `lms server start --bind 0.0.0.0 --port 1234`.
- Tailscale is connected and set to **Run unattended** (tray menu).
- Exactly one model loaded.

**In the Tailscale admin console**

- Access controls → **JSON editor**: the policy in
  `tailscale-operator/tailnet-policy.hujson`.
- Settings → **Trust credentials** → new OAuth client, `write` on
  General/Services, Devices/Core and Keys/Auth Keys, each with tag
  `tag:k8s-operator`. Keep the ID and secret for step 3, nowhere else.

**On the K3s server**

- `kubectl get nodes` works, Docker with buildx, Helm 3.

---

## Deploy / redeploy

Run everything **on the K3s server, in bash**, from the folder that holds the
six repositories side by side (`~/Desktop/Final-Project-Repos`; cloning them:
DEPLOY-K3S.md, A3). Steps 3 and 4 are once per cluster; the rest is safe
to repeat.

**1. Get the code**

```
for r in ai-scanner-*; do git -C "$r" pull --ff-only; done
```

**2. Build the images and hand them to K3s**

```
API_TAG=$(git -C ai-scanner-backend rev-parse --short HEAD)
GW_TAG=$(git -C ai-scanner-gateway rev-parse --short HEAD)
FE_TAG=$(git -C ai-scanner-frontend rev-parse --short HEAD)
bash ai-scanner-backend/build.sh --tag "$API_TAG"
docker build -t "review-gateway:$GW_TAG" ai-scanner-gateway
docker build -t "review-frontend:$FE_TAG" ai-scanner-frontend
bash ai-scanner-backend/scanner-worker/build.sh --tag 0.5.0
docker save "review-api:$API_TAG" "review-gateway:$GW_TAG" "review-frontend:$FE_TAG" scanner-worker:0.5.0 | sudo k3s ctr images import -
sed -i "s/^    newTag: [^ ]*/    newTag: $API_TAG/" ai-scanner-backend-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $GW_TAG/"  ai-scanner-gateway-deployment/kustomization.yaml
sed -i "s/^    newTag: [^ ]*/    newTag: $FE_TAG/"  ai-scanner-frontend-deployment/kustomization.yaml
```

Docker and K3s keep separate image stores, hence the import. The `sed` lines
point each deployment repository's `newTag` at the commit of the repository
that image was built from; commit those changes (with Argo CD, the commit IS
the deploy). The scanner-worker tag is fixed (`0.5.0`) because review-api reads it
from the `SCANNER_WORKER_IMAGE` env var, which Kustomize does not rewrite.
Bump it in `review-api/deployment.yaml` and in `job-test/` when you build a
new worker.

The frontend is built with an empty `VITE_GATEWAY_URL` (the Dockerfile's
default), so the UI calls the gateway on its own address, through the
Ingress. Nothing about the address is baked into the image.

**3. The Tailscale operator's credential (once per cluster)**

```
kubectl create namespace tailscale
read -rp "Client ID: " TS_ID; read -rsp "Client secret: " TS_SECRET; echo
kubectl -n tailscale create secret generic operator-oauth --from-literal=client_id="$TS_ID" --from-literal=client_secret="$TS_SECRET"
unset TS_ID TS_SECRET
```

`read -s` keeps the secret off the screen and out of shell history.

**4. The Tailscale operator (once per cluster)**

```
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update
helm upgrade --install tailscale-operator tailscale/tailscale-operator -n tailscale --version 1.102.4 -f tailscale-operator/values.yaml --wait
```

**5. The application's Secrets (once, before the first apply)**

The namespace must exist first, so apply only it, then create the Secrets:

```
kubectl apply -f namespace.yaml
read -rsp "Postgres password: " PG_PW; echo
kubectl -n ai-scan-repos create secret generic postgres-credentials \
  --from-literal=POSTGRES_USER=ai_scan_repos \
  --from-literal=POSTGRES_PASSWORD="$PG_PW" \
  --from-literal=POSTGRES_DB=ai_scan_repos
unset PG_PW
```

Then the GitHub OAuth App (GitHub → Settings → Developer settings → OAuth
Apps → New): **Homepage URL** `http://100.111.216.122`, **Authorization
callback URL** `http://100.111.216.122/auth/github/callback`. Both must match
the public address in `ai-scanner-gateway-deployment/deployment.yaml`. Then `gateway-secrets`, the
same way (you type each value, nothing is echoed):

```
read -rsp "Postgres password (same as above): " PG_PW; echo
read -rsp "OAuth client ID: " GH_ID; echo
read -rsp "OAuth client secret: " GH_SECRET; echo
kubectl -n ai-scan-repos create secret generic gateway-secrets \
  --from-literal=database-url="postgresql+psycopg://ai_scan_repos:${PG_PW}@postgres:5432/ai_scan_repos" \
  --from-literal=github-oauth-client-id="$GH_ID" \
  --from-literal=github-oauth-client-secret="$GH_SECRET" \
  --from-literal=github-webhook-secret="$(openssl rand -hex 32)" \
  --from-literal=settings-key="$(openssl rand -hex 32)"
unset PG_PW GH_ID GH_SECRET
```

`database-url` must start with `postgresql+psycopg://`: the gateway uses
psycopg 3, and a plain `postgresql://` makes SQLAlchemy look for psycopg2 and
crash. If the password contains `@`, `:` or `/`, pick another one (or
URL-encode it), or the URL breaks. The webhook secret is random because
webhooks cannot reach a tailnet address anyway. `settings-key` encrypts the
AI service API key an admin saves in Settings; admins themselves are listed in
the ConfigMap `gateway-admins` (`admins-configmap.yaml` in ai-scanner-gateway-deployment, change it
with `kubectl -n ai-scan-repos edit configmap gateway-admins`, no restart
needed). Details: DEPLOY-K3S.md, "AI model and admins".

**6. The application**

```
cd ~/Desktop/Final-Project-Repos
kubectl apply -k ai-scanner-backend-deployment     # first: it creates the namespace
kubectl apply -k ai-scanner-gateway-deployment
kubectl apply -k ai-scanner-frontend-deployment
kubectl -n ai-scan-repos rollout status statefulset/postgres
kubectl -n ai-scan-repos rollout status deploy/review-api
kubectl -n ai-scan-repos rollout status deploy/gateway
kubectl -n ai-scan-repos rollout status deploy/frontend
```

The first command only prints what will be applied. The gateway may restart
once or twice while Postgres is still starting; that is expected.

Then open **http://100.111.216.122/** in a browser on the tailnet and log in
with GitHub.

---

## Verify

| Check | Command | Expected |
|---|---|---|
| everything up | `kubectl -n ai-scan-repos get pods` | `frontend-…`, `gateway-…`, `review-api-…` and `postgres-0` all `Running`, `1/1` ready |
| the Ingress routes | `curl -s http://100.111.216.122/health` and `curl -s http://100.111.216.122/ \| head -3` | the gateway's health JSON, then the UI's HTML |
| operator and proxy up | `kubectl -n tailscale get pods` | `operator-…` and `ts-lmstudio-…-0`, both `Running` |
| the operator took the Service | `kubectl -n ai-scan-repos get svc lmstudio` | EXTERNAL-IP `ts-lmstudio-….tailscale.svc.cluster.local`, **not** `placeholder` |
| a pod reaches the model | see below (first command) | the model ids, one per line |
| review-api reaches the model | see below | `http://lmstudio:1234/v1 True` |
| review-api may create Jobs | `kubectl auth can-i create jobs -n ai-scan-repos --as=system:serviceaccount:ai-scan-repos:review-api` | `yes` |
| …but may not read Pods | same, with `list pods` | `no` |

```
kubectl -n ai-scan-repos exec deploy/review-api -- python -c "import json,urllib.request as u; [print(m['id']) for m in json.load(u.urlopen('http://lmstudio:1234/v1/models',timeout=10))['data']]"
kubectl -n ai-scan-repos exec deploy/review-api -- python -c "import json,urllib.request as u; p=json.load(u.urlopen('http://127.0.0.1:8000/api/v1/health?probe=1',timeout=30))['probe']; print(p['base_url'], p['reachable'])"
```

Both run inside the review-api Pod. A quick `kubectl run --image=curlimages/curl`
test Pod, as used in `code-review` before, is refused here: Pod Security
`restricted` requires a securityContext that `kubectl run` does not set.

## Retire the old `code-review` namespace (once `ai-scan-repos` works)

The old review-api `5fc02a9` and the old `lmstudio` Service still run there.
Scale it down first rather than deleting, so you can go back:

```
kubectl -n code-review scale deploy/review-api --replicas=0
```

When you are sure, `kubectl delete namespace code-review` removes it and
everything in it (including the old test PVC `pvctest` and its reports).

---

## Changing the public address

The app answers on any address that reaches Traefik, but the login flow needs
ONE address in three places, which must agree exactly:
`GATEWAY_FRONTEND_URL`, `GATEWAY_CORS_ORIGINS` and `GITHUB_OAUTH_REDIRECT_URI`
in `ai-scanner-gateway-deployment/deployment.yaml`, and the OAuth App's callback URL on GitHub.
Default: `http://100.111.216.122` (tailnet only). With HTTPS, also set
`GATEWAY_COOKIE_SECURE` to `"true"` and add `tls:` to the Ingress.

## Never commit

- the OAuth client IDs or secrets, the Postgres password, or `kubectl get secret … -o yaml` output;
- any kubeconfig (`~/.kube/config`, `k3s.yaml`): it holds the cluster's admin key.

## Behaviour worth knowing

- **Re-applying resets the lmstudio Service for a few seconds.** The file says
  `externalName: placeholder`; the operator replaces it. Every `apply` puts
  `placeholder` back until the operator corrects it. Under ArgoCD, add an
  `ignoreDifferences` entry for `/spec/externalName` on this Service.
- **review-api keeps review state in memory.** Restarting its Pod mid-scan
  loses the running review's poller. Reports and checkpoints on the PVC
  survive.
- **The model host is outside the cluster.** Laptop asleep, Tailscale
  disconnected or LM Studio stopped = reviews fail with an error.
- **`noexec` is not enforced** by local-path (measured 2026-09-21).

## When it goes wrong

| Symptom | Cause | Look at |
|---|---|---|
| `Could not resolve host: lmstudio` | EXTERNAL-IP is still `placeholder`: the operator is not running, or the Service is in another namespace | `kubectl -n tailscale logs deploy/operator --tail=50 \| grep lmstudio` |
| proxy `Running`, but the request times out | no `tag:k8s` grant for tcp:1234, LM Studio on localhost only, or the laptop is offline | admin console; on the laptop `netstat -ano \| findstr :1234` shows `0.0.0.0` |
| review-api `0/1` ready | `/readyz` fails: `/reports` or `/tmp` not writable | `kubectl -n ai-scan-repos describe pod -l app=review-api` |
| `ErrImagePull` / `ErrImageNeverPull` | the tag in `kustomization.yaml` or `SCANNER_WORKER_IMAGE` names an image K3s does not have | `sudo k3s crictl images \| grep -E 'review-api\|scanner-worker\|review-gateway'` |
| Pod `CreateContainerConfigError` | a Secret it reads does not exist yet | `kubectl -n ai-scan-repos get secrets` |
| `violates PodSecurity "restricted"` | a Pod without runAsNonRoot, seccompProfile, allowPrivilegeEscalation: false or drop ALL | the event names the missing field |

## Moving to EKS

The image is pushed to ECR and pulled by the nodes: set `newName` next to
`newTag` in `kustomization.yaml`, give `SCANNER_WORKER_IMAGE` the full ECR name,
and drop the `k3s ctr` import. The reports PVC needs `ReadWriteMany` (EFS),
because review-api and the Jobs may land on different nodes.
