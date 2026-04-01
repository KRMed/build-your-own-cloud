## Deploy Your Apps with GitOps (App of Apps)

**Required if you are using ArgoCD. Skip if deploying manually.**

This setup uses ArgoCD's [app of apps pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/#app-of-apps-pattern). A single **root application** points ArgoCD at a directory in your repo that contains `Application` manifests for every other service. When the root app syncs, ArgoCD discovers and deploys everything else automatically. You commit to git once and the entire cluster converges.

### Structure of a root application

Your root ArgoCD `Application` manifest (`bootstrap/root/root-app.yaml`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<YOUR_ORG>/<YOUR_REPO>.git
    targetRevision: main
    path: clusters/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true      # delete resources that are removed from git
      selfHeal: true   # revert manual changes made directly in the cluster
```

### Structure of a child application

Each service you want to deploy gets its own `Application` manifest in `clusters/prod/`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<YOUR_ORG>/<YOUR_REPO>.git
    targetRevision: main
    path: apps/my-app/base
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Apply the root app to start the cascade

```bash
kubectl apply -f bootstrap/root/root-app.yaml
```

Watch ArgoCD discover and sync everything:

```bash
watch argocd app list
```

### Writing ArgoCD Application manifests for Helm charts

The child application template above works for plain Kubernetes manifests (like your own apps). For Helm-based infrastructure like ingress-nginx and kube-prometheus-stack, the Application manifest looks slightly different: you point ArgoCD at a Helm chart directly rather than a local path:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ingress-nginx
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes.github.io/ingress-nginx
    chart: ingress-nginx
    targetRevision: "<CHART_VERSION>"   # find the latest at https://github.com/kubernetes/ingress-nginx/releases, use the chart version (e.g. "4.11.3"), not the controller version
    helm:
      valuesObject:
        controller:
          service:
            type: NodePort          # use LoadBalancer for Option B
          ingressClassResource:
            default: true           # makes nginx the cluster default IngressClass
          config:
            use-forwarded-headers: "true"          # pass real client IPs from Cloudflare headers
            compute-full-forwarded-for: "true"     # build full X-Forwarded-For chain
  destination:
    server: https://kubernetes.default.svc
    namespace: ingress-nginx
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

> **`use-forwarded-headers`** tells nginx to trust the `X-Forwarded-*` headers Cloudflare sends. Without it, your logs show Cloudflare's IP instead of the real client IP. This only applies when traffic arrives through a trusted proxy (Cloudflare or cloudflared); do not set it if your ingress is exposed directly with a public IP.
>
> **Reference:** [ingress-nginx ConfigMap options](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/) | [ArgoCD - Helm charts](https://argo-cd.readthedocs.io/en/stable/user-guide/helm/)

Use the same pattern for kube-prometheus-stack; swap the `repoURL`, `chart`, and `valuesObject` for that chart's values. See [monitoring-and-alerts.md](../05-extras/monitoring-and-alerts.md) for the monitoring-specific values to set.

### Expected sync order

Some components depend on others being ready first. A typical order:

1. **ingress-nginx** - must be running before anything that relies on `Ingress` resources works
2. **cloudflared** - requires the `tunnel-token` Secret from [cluster-secrets.md](cluster-secrets.md#cloudflare-tunnel-token)
3. **argocd-ingress** - once this syncs, ArgoCD gets a proper hostname (you can stop using port-forward)
4. **kube-prometheus-stack** - Helm chart, takes a few minutes on first sync
5. **monitoring extras** - ServiceMonitors that configure Prometheus scrape targets
6. **application workloads** - your apps

> **Enforcing sync order with sync waves:** The list above describes the recommended sequence, but ArgoCD syncs all apps in a root application concurrently by default. To guarantee ordering, add an `argocd.argoproj.io/sync-wave` annotation to each `Application` manifest. ArgoCD processes waves from lowest to highest and waits for each wave to be fully `Healthy` before starting the Next:
>
> ```yaml
> metadata:
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"   # lower numbers sync first; default is 0
> ```
>
> Assign wave `"0"` (or no annotation) to ingress-nginx and cloudflared, `"1"` to ArgoCD ingress, `"2"` to kube-prometheus-stack, and `"3"` to your application workloads. Resources within the same wave still sync in parallel.
>
> **Reference:** [ArgoCD Sync Waves](https://argo-cd.readthedocs.io/en/stable/user-guide/sync-waves/)

> **Single-node RAM note:** On a single node with 4GB RAM, running the full stack (ingress-nginx, cloudflared, ArgoCD, kube-prometheus-stack, and your apps) will push against the limit. If pods are evicted or stuck `Pending` due to memory pressure, the first thing to drop is kube-prometheus-stack; skip [monitoring](../05-extras/monitoring-and-alerts.md) and add it later when you have more headroom or a second node. ingress-nginx, cloudflared, and ArgoCD together use significantly less RAM and will run comfortably on 4GB without monitoring.

Apps will show `Progressing` while images are pulling. `Degraded` with `ImagePullBackOff` or `CreateContainerConfigError` means a Secret is missing - revisit [cluster-secrets.md](cluster-secrets.md).

<a id="checkpoint-4"></a>

> **Checkpoint:** Once everything shows `Healthy`, the cluster is fully GitOps-managed. ArgoCD is watching your repository and reconciling the cluster to match it continuously. From this point on, to change anything in the cluster, commit to git. If you manually edit a resource, ArgoCD reverts it. If a namespace gets deleted by accident, ArgoCD recreates it. The cluster is now self-healing.

**Next:** [Deploy Your First App](../04-deploy/)