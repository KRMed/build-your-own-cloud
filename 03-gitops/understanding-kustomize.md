## Understanding Kustomize

**Required if you are using the GitOps setup in [bootstrap-argocd.md](bootstrap-argocd.md) and [app-of-apps.md](app-of-apps.md). Can be skipped if you are deploying manifests manually with `kubectl apply`.**

Kustomize is a tool for customizing Kubernetes manifests without templating. Instead of using variables and `{{ }}` syntax like Helm, Kustomize works by layering patches on top of base YAML files. It is built into kubectl and also available as a standalone binary.

> **Reference:** [Kustomize documentation](https://kubectl.docs.kubernetes.io/references/kustomize/)

### The core concept

Every directory managed by Kustomize contains a `kustomization.yaml` file that lists what resources belong in that directory and what transformations to apply. When you run `kustomize build <directory>`, it reads that file and outputs a single combined YAML document with all resources merged.

A minimal `kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

That is enough to tell Kustomize "combine these three files into one output." From there you can add patches, common labels, namespace overrides, and more - but none of that is required to start.

### Why this setup uses Kustomize

ArgoCD uses Kustomize to render the manifests in your git repo before applying them to the cluster. When ArgoCD syncs an application, it runs `kustomize build` on the configured path and applies whatever comes out. This means you never manually run `kubectl apply` - you commit to git and ArgoCD handles the rest.

### Typical directory structure

```
your-repo/
├── bootstrap/
│   └── root/
│       ├── root-app.yaml        # The ArgoCD Application that bootstraps everything
│       └── kustomization.yaml
├── clusters/
│   └── prod/
│       ├── kustomization.yaml   # Lists all ArgoCD Application manifests
│       ├── ingress-nginx.yaml   # ArgoCD Application for ingress-nginx
│       ├── monitoring.yaml      # ArgoCD Application for kube-prometheus-stack
│       └── my-app.yaml          # ArgoCD Application for your app
├── apps/
│   └── my-app/
│       └── base/
│           ├── kustomization.yaml
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
└── infra/
    └── cloudflared/
        └── base/
            ├── kustomization.yaml
            ├── deployment.yaml
            └── namespace.yaml
```

The `clusters/prod/kustomization.yaml` lists the ArgoCD Application manifests in that directory:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ingress-nginx.yaml
  - monitoring.yaml
  - my-app.yaml
```

Each app directory has its own `kustomization.yaml` listing its own manifests. Kustomize treats these independently - changing one app's manifests does not affect the others.

**Next:** [Create In-Cluster Secrets](cluster-secrets.md)