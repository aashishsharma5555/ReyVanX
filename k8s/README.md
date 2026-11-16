# NetBird on AKS

These manifests deploy the Compose services to the `netbird` namespace. Caddy serves plain HTTP on port 80 through an Azure `LoadBalancer` Service. Dashboard login is configured for local-only browser access through `kubectl port-forward`; it does not use a Kubernetes `Ingress` resource.

## Important behavior

- Management stays at one replica because its current SQLite data is stored on a persistent Azure disk.
- Only Caddy and dashboard autoscale from 2 to 10 replicas at 70% CPU. Signal stays at two replicas; relay and management stay at one replica.
- Workloads are pinned to `agentpool2`. That pool currently has two nodes and no node autoscaler, so pods can remain Pending if the pool runs out of capacity.
- The NetworkPolicies are included, but this AKS cluster has `networkPolicy: none`; they are not enforced until cluster networking is configured to enforce policies.
- The public IP is `20.62.177.218`, allocated by the Caddy LoadBalancer Service.
- HTTP is intentionally used to match the current Docker Compose setup. `USE_AUTH0=false`; the built-in NetBird management IdP remains enabled for dashboard login.
- Dashboard API and login URLs use `http://20.62.177.218`.
- HTTP sends traffic without encryption. Browsers may block NetBird login on a public-IP HTTP origin because it is not a secure context; use only for testing.
- Coturn/STUN is not deployed; peers may need to use the NetBird relay.
- Import credentials as Kubernetes Secrets; do not commit generated Secret YAML or copied database files.

## Apply

From this directory, apply the namespace, storage, and services first:

```powershell
kubectl apply -f .\00-namespace.yaml
kubectl apply -f .\01-storage.yaml
kubectl apply -f .\services\
kubectl get service caddy -n netbird --watch
```

Apply the Caddy ConfigMap:

```powershell
kubectl apply -f .\02-caddy-config.yaml
```

Create these Secrets from the configured files without writing their contents into this directory:

- `dashboard-env` from `dashboard.env`
- `relay-env` from `relay.env`
- `management-config` from `management.json`

For a migrated deployment, copy a consistent SQLite online backup into the `management-data` PVC before starting the management Deployment. Then apply:

```powershell
kubectl apply -f .\deployments\
kubectl apply -f .\autoscaling\
kubectl apply -f .\network-policies\
kubectl rollout status deployment/caddy -n netbird
kubectl rollout status deployment/dashboard -n netbird
kubectl rollout status deployment/signal -n netbird
kubectl rollout status deployment/relay -n netbird
kubectl rollout status deployment/management -n netbird
```

