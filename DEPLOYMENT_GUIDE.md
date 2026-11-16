# NetBird on AKS: Zero-to-End Deployment Guide

This guide documents the repository's NetBird Docker Compose setup, its AKS
manifests, the Azure workflow used for the current cluster, and a repeatable
two-node AKS example.

> **Important: repository state and example commands are not identical.** The
> existing cluster inspected during this work is
> `netbird-cloud4green-aks` in resource group
> `netbird-cloud4green-aks_group`, in `eastus`. It currently has two pools,
> `agentpool` and `agentpool2`, with two nodes each. The manifests select
> `agentpool2`. The requested `az aks create` command below instead creates a
> new, single-pool cluster named `agentpool`. Do not run that create command
> against an existing cluster or apply the current manifests to a new
> `agentpool`-only cluster without first reconciling the node selectors.

## 1. Architecture at a glance

The AKS deployment runs these workloads in the `netbird` namespace:

| Component | Current replicas | Kubernetes exposure | Notes |
|---|---:|---|---|
| Caddy | 2; HPA min 2, max 10 | Azure `LoadBalancer`, TCP 80 | Public HTTP entry point and reverse proxy |
| Dashboard | 2; HPA min 2, max 10 | ClusterIP, TCP 80 | Uses `dashboard-env` Secret |
| Signal | 2, fixed | ClusterIP, TCP 80 and 10000 | 10000 is the gRPC backend used by Caddy |
| Relay | 1, fixed | ClusterIP, TCP 80 | Uses `relay-env` Secret |
| Management | 1, fixed | ClusterIP, TCP 80 | Embedded IdP and SQLite data on a 10 GiB Azure disk |
| Coturn | Not deployed to AKS | None | Present in Compose only; its UDP port range needs separate AKS networking |

The browser reaches Caddy on the public IP. Caddy routes dashboard, management,
Signal, and relay HTTP/gRPC paths to in-cluster Services. The management
Deployment uses a `Recreate` strategy because the SQLite PVC is a single-writer
`ReadWriteOnce` disk.

The current public endpoint is `http://20.62.177.218`. This is plain,
unencrypted HTTP. NetBird login uses browser cryptography APIs that are not
available to all browsers on a public-IP HTTP origin. The observed login error
was `window.crypto.subtle is unavailable`, followed by `Unauthenticated`.
Changing Kubernetes routing does not make a public HTTP origin a secure browser
context. Use HTTPS with a trusted hostname/certificate for public production
access; plain HTTP should be limited to isolated testing.

## 2. Repository files used

- [`docker-compose.yml`](./docker-compose.yml): source Compose services and image
  versions.
- [`getting-started.sh`](./getting-started.sh): generates the Compose
  configuration. Its `use-ip` branch uses HTTP and relay protocol `rel`; a
  domain branch selects HTTPS and `rels`.
- [`Caddyfile`](./Caddyfile): Compose reverse-proxy routes.
- [`dashboard.env`](./dashboard.env), [`management.json`](./management.json),
  [`relay.env`](./relay.env), and [`turnserver.conf`](./turnserver.conf):
  Compose settings. These can contain credentials and must be treated as
  sensitive.
- [`k8s/00-namespace.yaml`](./k8s/00-namespace.yaml): `netbird` namespace.
- [`k8s/01-storage.yaml`](./k8s/01-storage.yaml): 10 GiB management PVC using
  `managed-csi`.
- [`k8s/02-caddy-config.yaml`](./k8s/02-caddy-config.yaml): active HTTP-only
  Caddy ConfigMap.
- [`k8s/Caddyfile`](./k8s/Caddyfile): standalone Caddyfile; the Deployment
  mounts the ConfigMap from `02-caddy-config.yaml`.
- [`k8s/dashboard-env.yaml`](./k8s/dashboard-env.yaml): dashboard Secret
  manifest. Do not publish secret values from it.
- [`k8s/deployments/`](./k8s/deployments/): one Deployment manifest per service.
- [`k8s/services/`](./k8s/services/): one Service manifest per service.
- [`k8s/autoscaling/`](./k8s/autoscaling/): HPAs for Caddy and dashboard only.
- [`k8s/network-policies/`](./k8s/network-policies/): optional ingress/egress
  policies.
- [`k8s/caddy-config-backup.yaml`](./k8s/caddy-config-backup.yaml): backup
  configuration, not the active HTTP ConfigMap. Do not apply it over the active
  Caddy ConfigMap.
- [`k8s/README.md`](./k8s/README.md): shorter, repository-specific apply notes.

Avoid applying every YAML in `k8s` as a single directory without inspecting
which files are active. In particular, the backup Caddy config and Secret
manifests are not interchangeable with the active configuration.

## 3. Local Git and Azure CLI setup

Install Git, Azure CLI, Docker (for the Compose workflow), and `kubectl`. Sign
in to Azure and select the intended subscription:

```powershell
az login
az account list --output table
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
az account show --output table
az version
kubectl version --client
```

### Clone an existing Git repository

Use this when the remote repository already exists:

```powershell
git clone "<REPO_URL>" netbird
Set-Location .\netbird
git remote -v
git branch --show-current
```

### Initialize a new local repository and connect it to an existing remote

Use this instead of `git clone` when files already exist locally and no `.git`
directory exists:

```powershell
Set-Location "<PATH_TO_LOCAL_NETBIRD_REPOSITORY>"
git init
git remote add origin "<REPO_URL>"
git branch -M main
git remote -v
```

If `origin` already exists, inspect it with `git remote -v` and update it with
`git remote set-url origin "<REPO_URL>"`; do not add a duplicate remote.

## 4. Create a resource group and a two-node AKS cluster

These are **new-cluster example commands**. Choose a subscription, region,
resource group, cluster name, Kubernetes version, VM size, and networking
settings appropriate for the environment.

```powershell
$RESOURCE_GROUP_NAME = "<RESOURCE_GROUP_NAME>"
$LOCATION = "<LOCATION>"
$AKS_CLUSTER_NAME = "<AKS_CLUSTER_NAME>"

az group create `
  --name $RESOURCE_GROUP_NAME `
  --location $LOCATION

az aks create `
  --resource-group $RESOURCE_GROUP_NAME `
  --name $AKS_CLUSTER_NAME `
  --node-count 2 `
  --nodepool-name agentpool `
  --generate-ssh-keys
```

The requested command creates a single agent pool named `agentpool` with two
nodes. Kubernetes schedules pods onto eligible nodes; a Service distributes
requests among ready pod endpoints. The scheduler may place replicas on both
nodes, but these manifests do not define topology-spread constraints, so an
even split is not guaranteed. Use `kubectl get pods -n netbird -o wide` to
observe actual placement.

### Important node-selector compatibility

Every current workload manifest has this selector:

```yaml
nodeSelector:
  kubernetes.azure.com/agentpool: agentpool2
```

This matches the existing `agentpool2` pool, not the new `agentpool` in the
example creation command. For a new one-pool cluster, either:

1. Change the node selector in each Deployment to `agentpool`; or
2. Create/use a pool named `agentpool2`.

Otherwise pods will remain `Pending` with a node-affinity or selector mismatch.
The current inspected cluster has two fixed pools (`agentpool` and
`agentpool2`), each with two nodes; cluster and node-pool autoscaling were
disabled at inspection time.

### Connect `kubectl` to the cluster

```powershell
az aks get-credentials `
  --resource-group "<RESOURCE_GROUP_NAME>" `
  --name "<AKS_CLUSTER_NAME>"

kubectl config current-context
kubectl get nodes -o wide
kubectl get namespaces
```

For the existing cluster, the values observed were:

```powershell
az aks get-credentials `
  --resource-group "netbird-cloud4green-aks_group" `
  --name "netbird-cloud4green-aks"
```

The AKS-managed resource group for that cluster was
`MC_netbird-cloud4green-aks_group_netbird-cloud4green-aks_eastus`. Azure
creates and manages this node resource group; do not use it as the application
resource group.

## 5. Plain-HTTP and embedded authentication settings

The current Kubernetes configuration intentionally serves HTTP on port 80:

- [`k8s/02-caddy-config.yaml`](./k8s/02-caddy-config.yaml) contains
  `auto_https off` and a `:80` site.
- [`k8s/services/caddy.yaml`](./k8s/services/caddy.yaml) exposes TCP port 80 as
  a `LoadBalancer`.
- The Caddy Deployment listens on container port 80 only.
- The dashboard endpoint variables use `http://<PUBLIC_IP>`.
- The dashboard uses `USE_AUTH0=false`, so it is not configured for Auth0.
- `LETSENCRYPT_DOMAIN=none` disables the dashboard's LetsEncrypt setting.
- The management configuration has `Signal.Proto: http` and an embedded IdP.
- The management arguments include `--port 80`,
  `--single-account-mode-domain=netbird.selfhosted`,
  `--dns-domain=netbird.selfhosted`, and
  `--idp-sign-key-refresh-enabled`.

**No external identity provider is configured, but this is not “OIDC disabled.”**
The repository's `getting-started.sh` describes the dashboard as “OIDC - using
embedded IdP”; `AUTH_AUTHORITY` points to `/oauth2`, and
`management.json` has `EmbeddedIdP.Enabled: true`. NetBird's embedded IdP is the
local identity provider used for the login flow. `USE_AUTH0=false` disables
Auth0-specific frontend behavior; it does not turn off the embedded IdP or the
OIDC-based browser login flow.

The Compose setup also includes Coturn, but AKS does not currently have a
Coturn Deployment or UDP LoadBalancer Service. `turnserver.conf` specifies
UDP/TCP ports 3478 and 5349 and a UDP allocation range of 49152-65535; it also
references certificate/key paths. This component requires explicit UDP
networking and certificate decisions before it can be considered deployed.

### Configure Kubernetes Secrets

The Deployments expect Secrets named `dashboard-env`, `relay-env`, and
`management-config`. `dashboard-env.yaml` defines the dashboard Secret, but do
not commit real credentials in Secret YAML. A safer pattern is to create or
update the Secrets from protected local files or a secret manager:

```powershell
kubectl create secret generic dashboard-env `
  --namespace netbird `
  --from-env-file=".\dashboard.env" `
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic relay-env `
  --namespace netbird `
  --from-env-file=".\relay.env" `
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic management-config `
  --namespace netbird `
  --from-file=management.json=".\management.json" `
  --dry-run=client -o yaml | kubectl apply -f -
```

Before creating Secrets, ensure that management, dashboard, and relay all use
the same public endpoint and matching relay authentication secret. Keep
`DataStoreEncryptionKey` unchanged when reusing an existing management data
directory. Do not print Secret values in logs or screenshots.

## 6. Apply the Kubernetes manifests

Run commands from the repository root. Apply explicit directories/files so
backup YAML or credentials are not accidentally applied:

```powershell
kubectl apply -f .\k8s\00-namespace.yaml
kubectl apply -f .\k8s\01-storage.yaml
kubectl apply -f .\k8s\02-caddy-config.yaml

kubectl apply -f .\k8s\services\
kubectl get service caddy -n netbird --watch

kubectl apply -f .\k8s\deployments\
kubectl apply -f .\k8s\autoscaling\
kubectl apply -f .\k8s\network-policies\
```

The `caddy` Service requests an Azure LoadBalancer and Azure assigns a public
IP. The current observed IP is `20.62.177.218`; for another cluster, use the IP
reported by Kubernetes. Apply/update the dashboard and management Secret
endpoints to match that IP and the chosen HTTP/HTTPS scheme, then restart
affected Deployments:

```powershell
kubectl rollout restart deployment/dashboard deployment/management deployment/relay -n netbird
```

For a fresh deployment, check the PVC and rollout status:

```powershell
kubectl get pvc -n netbird
kubectl rollout status deployment/caddy -n netbird
kubectl rollout status deployment/dashboard -n netbird
kubectl rollout status deployment/signal -n netbird
kubectl rollout status deployment/relay -n netbird
kubectl rollout status deployment/management -n netbird
```

For migration from Docker, do not copy SQLite database files while they are
being actively written. Use SQLite's online backup API (or stop management
briefly and make a cold backup), verify each backup with `PRAGMA integrity_check`,
then copy the backups into the mounted `management-data` PVC before starting
the AKS management pod. Preserve the existing management encryption key and
relay secret. The PVC is `ReadWriteOnce`; management uses a `Recreate` rollout
so the old pod releases the disk before its replacement starts. A management
restart may briefly interrupt management API and login.

### Verify scheduling, Services, and logs

```powershell
kubectl get nodes -o wide
kubectl get pods -n netbird -o wide
kubectl get svc -n netbird
kubectl get endpointslices -n netbird
kubectl get pvc -n netbird
kubectl get events -n netbird --sort-by=.lastTimestamp
```

`kubectl get pods -o wide` shows which node runs each pod. The current manifests
pin pods to `agentpool2`; Services select pods by their `app` label and balance
traffic across ready endpoints.

```powershell
kubectl logs deployment/caddy -n netbird --all-pods=true --tail=100
kubectl logs deployment/dashboard -n netbird --all-pods=true --tail=100
kubectl logs deployment/management -n netbird --tail=100
kubectl describe pod "<POD_NAME>" -n netbird
kubectl describe service caddy -n netbird
```

## 7. Horizontal Pod Autoscaling

The current repository has two HPA manifests:

- [`k8s/autoscaling/caddy.yaml`](./k8s/autoscaling/caddy.yaml)
- [`k8s/autoscaling/dashboard.yaml`](./k8s/autoscaling/dashboard.yaml)

Each specifies `minReplicas: 2`, `maxReplicas: 10`, and a CPU average utilization
target of 70%. CPU utilization is calculated against container CPU requests.
The Caddy and dashboard containers request `100m` CPU.

The same configuration can be created imperatively:

```powershell
kubectl autoscale deployment caddy `
  --namespace netbird `
  --cpu-percent=70 `
  --min=2 `
  --max=10

kubectl autoscale deployment dashboard `
  --namespace netbird `
  --cpu-percent=70 `
  --min=2 `
  --max=10

kubectl get hpa -n netbird
kubectl describe hpa caddy -n netbird
kubectl top pods -n netbird
```

Use either `kubectl autoscale` or the HPA YAML as the primary definition for a
given Deployment; do not maintain conflicting definitions. The manifests do
**not** autoscale Signal, Relay, Management, or Coturn. Because the limits are
per Deployment, Caddy plus Dashboard could reach 20 pods combined, not 10 total.
The current `agentpool2` has two fixed nodes and no cluster autoscaler, so pods
can become unschedulable before an HPA reaches its maximum. Configure enough
node capacity for the desired pod count.

## 8. Azure Load Balancer in the Portal

### Recommended AKS-managed workflow

The Caddy Service already has `type: LoadBalancer`. AKS's cloud provider
creates and reconciles the Azure Load Balancer frontend, backend pool, health
probe, and rule. Find it in the AKS **node resource group**, normally named
`MC_<RESOURCE_GROUP>_<CLUSTER_NAME>_<LOCATION>`:

1. Open **Azure Portal → Kubernetes services → your AKS cluster → Properties**
   and note the node resource group.
2. Open **Resource groups → `MC_<RESOURCE_GROUP>_<CLUSTER_NAME>_<LOCATION>`**.
3. Open the **Load balancer** associated with the cluster.
4. Under **Frontend IP configurations**, verify the public IP matches
   `kubectl get service caddy -n netbird`.
5. Under **Backend pools**, verify the AKS node NICs/instances are registered.
   The Kubernetes Service may use all eligible cluster nodes even though the
   application pods are pinned to `agentpool2`.
6. Under **Health probes**, inspect the probe generated for the Caddy Service.
   The current external Service is TCP port 80; it does not serve HTTPS.
7. Under **Load balancing rules**, inspect the rule for frontend TCP port 80
   and its backend pool/probe. `kubectl get service caddy -n netbird -o wide`
   shows the Service port and allocated NodePort.
8. Under **Networking / Network security groups**, verify the Service's
   required inbound port is allowed. Avoid opening management, dashboard, or
   Signal ClusterIP ports directly to the Internet; Caddy is the public entry
   point.

**Do not manually add duplicate rules or probes to an AKS-managed Load
Balancer.** AKS reconciles resources based on Kubernetes Services, and manual
changes can be overwritten or break health checking. To expose another port,
declare it on the appropriate Kubernetes `LoadBalancer` Service and let AKS
create the Azure resources.

### If a manually managed Load Balancer is explicitly required

This is not the current architecture and is not recommended alongside the
AKS-managed Caddy `LoadBalancer` Service. First change the Kubernetes exposure
design so only one controller owns the frontend/rules. Then, in the Portal:

1. Create/select the frontend public IP and register the intended AKS node
   backend pool.
2. Read the actual Kubernetes NodePort from `kubectl get svc -n netbird`; do
   not guess or hard-code it.
3. Create one TCP rule from frontend port 80 to the Caddy NodePort, associated
   with the selected backend pool and health probe.
4. Configure a TCP probe for that backend NodePort (or the probe mechanism
   explicitly required by the chosen AKS networking mode).
5. Do not create separate public rules for dashboard/management/Signal
   ClusterIP Services. They are routed internally by Caddy.
6. Validate rule health, NSG access, and the public HTTP response.

Manual Load Balancer rules do not provide HTTPS, application authentication, or
encryption. Configure a trusted TLS endpoint before public production use.

## 9. Common errors and troubleshooting

| Symptom | Likely cause | What to check/do |
|---|---|---|
| `az login` opens the wrong tenant/account | Multiple Azure accounts or tenants | Run `az account list --output table`, then explicitly set the subscription with `az account set`. |
| `az aks get-credentials` returns authorization errors | Wrong subscription/RG/cluster or insufficient Azure RBAC | Confirm `az account show`, resource group, cluster name, and AKS cluster-user credentials permission. |
| `kubectl` shows an unexpected cluster | Kubeconfig context was not switched | Check `kubectl config current-context` and retrieve credentials for the intended AKS cluster. |
| `node(s) didn't match Pod's node affinity/selector` | Manifest selects `agentpool2`, but the cluster has only `agentpool` | Align all Deployment selectors with the actual pool name or add the expected pool. |
| Pods remain `Pending` with insufficient CPU/memory | Two nodes cannot fit pod requests or HPA has scaled out | Check `kubectl describe pod`, node allocatable resources, and requests/limits; add node capacity. |
| HPA reports `cpu: <unknown>` | Metrics Server has no current pod metrics, pods are unready, or CPU requests are missing | Check `kubectl top pods`, Metrics Server, HPA events, and each target container's CPU requests. |
| PVC remains `Pending` | CSI provisioning failure, unsupported StorageClass, or no first consumer for `WaitForFirstConsumer` | Check `kubectl describe pvc`, `kubectl get storageclass`, CSI events, quotas, and Azure storage policy. Do not assume a StorageClass is usable because it exists. |
| Azure Files provisioning says non-TLS endpoints are denied | Subscription/storage policy blocks the dynamically requested storage account | Do not use that PVC for Caddy; current Caddy needs no persistent storage for HTTP. Keep the management Azure Disk PVC. |
| `Multi-Attach` or volume attach timeout | `ReadWriteOnce` management disk is still attached to the old pod/node | Wait for old pod termination and use the Deployment's `Recreate` strategy. Do not run multiple management replicas on the same SQLite volume. |
| Management starts with empty state | Existing Docker SQLite data was not copied to the PVC, wrong PVC mounted, or data dir/key mismatch | Verify PVC and mount path `/var/lib/netbird`; restore a consistent SQLite backup and original `DataStoreEncryptionKey`. |
| Dashboard displays `Unauthenticated` / `window.crypto.subtle is unavailable` | Browser blocks Web Crypto on public-IP HTTP | Public HTTP may serve the page but cannot guarantee the login flow. Use a secure browser origin (trusted HTTPS) for public access. |
| Dashboard login loops or discovery fails | Dashboard `AUTH_AUTHORITY`, management issuer, redirect URI, and browser origin do not agree | Verify `/oauth2/.well-known/openid-configuration`; match all URLs and scheme/host, then restart Dashboard and Management. |
| Dashboard points at an old/private IP | Secret was not updated or Deployment did not restart | Inspect variable names without printing values, update `dashboard-env`, and run `kubectl rollout restart deployment/dashboard -n netbird`. |
| Management/relay auth fails | Relay secret mismatch between `management-config` and `relay-env` | Recreate both Secrets with the same generated relay secret; restart Management and Relay. Never print the secret. |
| `no endpoints available` / Caddy returns 502 | Target pods are not Ready, Service selector/labels differ, or container port is wrong | Check `kubectl get endpointslices`, pod labels, readiness probes, and Services. |
| Caddy returns 404 for expected NetBird path | Wrong Caddyfile, path matcher, or stale ConfigMap/pod | Verify active `02-caddy-config.yaml`, ConfigMap content, and Caddy logs; restart Caddy after changing mounted ConfigMap data. |
| Load Balancer IP remains `<pending>` | Azure quota, policy, subnet, public IP, or cloud-provider provisioning issue | Describe the Service and inspect events; verify Azure quotas and node resource group. |
| Load Balancer is provisioned but not reachable | NSG/firewall blocks port 80 or probe/rule/backend is unhealthy | Inspect Service events and Portal probe/rule health; allow only the required frontend traffic. |
| Manually created Azure rule disappears | AKS controller reconciled the managed Load Balancer | Declare the intended port on a Kubernetes Service instead of editing the managed rule. |
| NetworkPolicy YAML applies but traffic is still open | AKS network policy enforcement is disabled (`networkPolicy: none`) | Enable a supported AKS policy implementation and validate it; YAML alone does not enforce policies. |
| NetworkPolicy blocks application traffic | Policy selector/ports/DNS egress do not match the actual CNI or pod traffic | Inspect policies and endpoints; allow DNS and only the required application ports. |
| Peers cannot establish direct connectivity | Coturn is not deployed; UDP STUN/TURN ports are unavailable | Configure/deploy Coturn with external UDP networking and valid credentials, or verify relay fallback. |
| Relay advertised address/port is unreachable | `NB_EXPOSED_ADDRESS` differs from public IP/service/listener | Match management Relay address, relay `NB_EXPOSED_ADDRESS`, public IP, and externally reachable port. |
| Changes to Secret have no effect | Secret values are consumed as environment variables at pod startup | Restart the affected Deployment and wait for rollout. |
| `kubectl apply -f .\k8s\` applies stale or sensitive YAML | Directory includes backup configs and Secret manifests | Apply explicit active paths as shown above; inspect manifests and staged Git files before use. |

Useful focused checks:

```powershell
kubectl describe pod "<POD_NAME>" -n netbird
kubectl describe pvc management-data -n netbird
kubectl describe service caddy -n netbird
kubectl describe hpa caddy -n netbird
kubectl get events -n netbird --sort-by=.lastTimestamp
kubectl logs deployment/management -n netbird --tail=100
kubectl logs deployment/caddy -n netbird --all-pods=true --tail=100
```

## 10. Git review, commit, and push

**Security gate:** this repository contains configuration files that can hold
live credentials, including `management.json`, `relay.env`,
`turnserver.conf`, and Kubernetes Secret YAML such as
`k8s/dashboard-env.yaml`. Do not push those values to a public or shared remote.
Move secrets to a protected secret manager/CI variables, rotate credentials
that have already been exposed, and ensure secret files are excluded from the
commit before staging. A `.gitignore` rule does not remove a file already
tracked by Git.

Review the working tree and the exact staged file list:

```powershell
git status --short
git diff --check
git add .
git diff --cached --name-only
git diff --cached --check
```

If the staged list contains credentials, Secret manifests with real values, or
local backups, stop and unstage/exclude them before committing. Do not paste
secret values into tickets or logs. Once the staged change has been reviewed:

```powershell
git commit -m "Add NetBird AKS deployment manifests and zero-to-end guide"
git push -u origin main
```

If the branch already tracks `origin/main`, `git push` is sufficient after the
first upstream setup. Never force-push shared deployment history without an
explicit recovery plan.

## 11. Post-deployment checklist

- [ ] Azure CLI is using the intended subscription and AKS context.
- [ ] Workload node selectors match a real node pool.
- [ ] Caddy has a public IP; only intended public ports are open.
- [ ] All Deployments have the expected ready replicas and probes.
- [ ] Caddy and Dashboard HPAs show valid CPU metrics, min 2, max 10.
- [ ] Management PVC is `Bound`, and its SQLite data/key are backed up.
- [ ] Dashboard, management issuer/redirects, and relay addresses use matching endpoints.
- [ ] Embedded IdP is understood; `USE_AUTH0=false` does not disable internal OIDC.
- [ ] Plain HTTP is restricted to testing; use trusted HTTPS for public production.
- [ ] Coturn is either intentionally omitted or separately configured with UDP ports.
- [ ] NetworkPolicy enforcement is confirmed, not inferred from YAML being applied.
- [ ] No credential-bearing files or Secret values are included in Git commits.
