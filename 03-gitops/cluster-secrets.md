## Create In-Cluster Secrets

**Required before ArgoCD syncs anything.**

When ArgoCD deploys your manifests, it creates pods that reference Secrets by name. If a Secret does not exist when a pod tries to start, the pod will fail immediately with `CreateContainerConfigError`. Secrets must be created manually in the cluster before the GitOps sync begins because they are never stored in git.

> **Why not store secrets in git?** If a secret is committed to a repository - even a private one - it can be leaked through git history, forks, accidental exposure, or compromised access tokens. Keeping secrets out of git entirely is the only safe approach. For more on this pattern, see [Kubernetes Secrets documentation](https://kubernetes.io/docs/concepts/configuration/secret/).

### Create your namespaces

Create all namespaces upfront so you can create Secrets in them before ArgoCD deploys anything:

```bash
kubectl create namespace argocd
kubectl create namespace cloudflared
kubectl create namespace ingress-nginx
kubectl create namespace monitoring
# Add one namespace per application you plan to deploy
kubectl create namespace <YOUR_APP_NAMESPACE>
```

### Cloudflare tunnel token

**Required only if using [Option A (Cloudflare Tunnel)](../02-expose-services/option-a-cloudflare-tunnel.md).**

Use the token you copied when creating the tunnel:

```bash
kubectl create secret generic tunnel-token \
  --from-literal=token=<YOUR_CLOUDFLARE_TUNNEL_TOKEN> \
  -n cloudflared
```

> **Reference:** [Tunnel run parameters - TUNNEL_TOKEN](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/configure-tunnels/tunnel-run-parameters/)

The cloudflared deployment references this Secret as an environment variable. If the Secret is missing or the token value is wrong, cloudflared pods will crash and the tunnel will never establish a connection.

### Container registry pull secret

**Required only if your container images are in a private registry.**

If your images are public, skip this section. If they are private, your cluster needs credentials to pull them. The pattern is the same regardless of which registry you use:

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=<REGISTRY_HOSTNAME> \
  --docker-username=<YOUR_USERNAME> \
  --docker-password=<YOUR_TOKEN_OR_PASSWORD> \
  -n <YOUR_APP_NAMESPACE>
```

Common values for `--docker-server`:
- GitHub Container Registry (GHCR): `ghcr.io`
- Docker Hub: `https://index.docker.io/v1/`
- Any self-hosted registry: the registry's hostname

Kubernetes Secrets are namespace-scoped. If you have multiple app namespaces that pull from the same private registry, create this secret in each namespace.

For GHCR, generate a [Personal Access Token](https://github.com/settings/tokens) with `read:packages` scope and use it as the password value.

> **Reference:** [Pull an Image from a Private Registry](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/)

### Monitoring credentials

**Required if you are deploying kube-prometheus-stack.**

Grafana expects its admin credentials in a Secret rather than in the Helm values file (which ends up in git):

```bash
kubectl create secret generic grafana-admin-credentials \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=<YOUR_GRAFANA_PASSWORD> \
  -n monitoring
```

The Secret name and key names must match what your Helm `values.yaml` references under `grafana.admin.existingSecret`. If you change the Secret name here, update your values file to match.

### Application secrets

**Required for any app that reads credentials at runtime.**

Look at each app's deployment manifest for `secretKeyRef` or `envFrom.secretRef` entries - those tell you exactly what Secret names and key names the app expects.

```bash
kubectl create secret generic <SECRET_NAME> \
  --from-literal=<KEY_NAME>=<VALUE> \
  --from-literal=<ANOTHER_KEY>=<ANOTHER_VALUE> \
  -n <APP_NAMESPACE>
```

→ **Next:** [What is GitOps and Why Use It](what-is-gitops.md)