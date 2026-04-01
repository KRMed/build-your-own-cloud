## Deploying Your First App End-to-End

This section puts everything together with a concrete example: deploying a simple nginx static site and getting it live in a browser at a real URL. Even if your actual app is different, the flow is identical: write manifests → commit → ArgoCD syncs → verify live.

> **Skipped ArgoCD?** Replace Steps 5 and 6 with:
> ```bash
> kubectl apply -f apps/hello/base/
> ```
> Once the command succeeds, skip to [Verify it's live](#step-7-verify-its-live).

### Step 1: Create the manifest files

Create this directory structure in your repo:

```
apps/
└── hello/
    └── base/
        ├── kustomization.yaml
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

**`apps/hello/base/deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: hello
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: nginx:stable-alpine
          ports:
            - containerPort: 80
```

**`apps/hello/base/service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: hello
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 80
```

**`apps/hello/base/ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello
  namespace: hello
spec:
  ingressClassName: nginx
  rules:
    - host: hello.your-domain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello
                port:
                  number: 80
```

**`apps/hello/base/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
```

### Step 2: Add the ArgoCD Application

Create `clusters/prod/hello.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: hello
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  project: default
  source:
    repoURL: git@github.com:<YOUR_ORG>/<YOUR_REPO>.git
    targetRevision: main
    path: apps/hello/base
  destination:
    server: https://kubernetes.default.svc
    namespace: hello
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Add `hello.yaml` to your `clusters/prod/kustomization.yaml` under `resources`.

### Step 3: Create the namespace (if not using CreateNamespace)

The `CreateNamespace=true` syncOption handles this automatically. If you want to pre-create it manually:

```bash
kubectl create namespace hello
```

### Step 4: Make it reachable

How you do this depends on which [exposure option](../02-expose-services/) you chose:

**Option A - Cloudflare Tunnel:** Go to your tunnel's public hostname config and add a new entry:

| Public hostname | Type | URL |
|---|---|---|
| `hello.your-domain.com` | HTTP | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` |

Cloudflare manages the DNS record automatically when you save. Your app will be live at `https://hello.your-domain.com`.

**Option B - Direct public IP or port forwarding:** Add a DNS A record for `hello.your-domain.com` pointing to your public IP now if you haven't already. The Ingress resource you created in Step 1 is enough; NGINX will route traffic for that hostname to your service. If you set up [cert-manager](../02-expose-services/option-b-public-ip.md#tls-certificates-required-for-both-b1-and-b2), add the annotation and `tls` block to your Ingress and it will provision a certificate automatically. Your app will be live at `https://hello.your-domain.com` once the certificate issues.

**Option C1 - NodePort (LAN-only):** Skip this step; there is no domain. Find the NodePort assigned to the ingress-nginx service and access your app at `http://<NODE_IP>:<NODEPORT>` from any device on your network:
```bash
kubectl get svc -n ingress-nginx
```
Look for the `ingress-nginx-controller` service and note the port mapped to `80/TCP` in the 30000–32767 range.

**Option C2 - Local DNS (LAN-only):** Add a DNS record in Pi-hole or your local resolver pointing `hello.your-domain.com` (or whatever local hostname you chose) at your node's LAN IP. Your app will be live at `http://hello.your-domain.com` from any device on your network.

### Step 5: Open a PR and merge

```bash
git checkout -b deploy-hello
git add apps/hello clusters/prod/hello.yaml clusters/prod/kustomization.yaml
git commit -m "deploy hello app"
git push origin deploy-hello
# open PR on GitHub, CI gates validate manifests and scan for secrets
# after both checks pass, merge
```

### Step 6: Watch ArgoCD sync

```bash
watch argocd app get hello
```

Or open the ArgoCD UI at `https://argocd.your-domain.com`. The `hello` app will appear, go `Progressing` while the nginx image pulls, then turn `Healthy`.

### Step 7: Verify it's live

Open `https://hello.your-domain.com` in a browser. You should see the nginx welcome page. If using Option C, access your app at the NodePort URL or local hostname you configured in Step 4 instead. If something is wrong, start here:

```bash
kubectl get pods -n ingress-nginx
kubectl describe ingress hello -n hello   # confirms the ingress was picked up by nginx
```

**If using Option A (Cloudflare Tunnel):** a 502 means the tunnel is up but not routing correctly. Check the tunnel connector and its logs:

```bash
kubectl get pods -n cloudflared
kubectl logs -n cloudflared -l app=cloudflared
```

**If using Option B (direct public IP):** a 502 or unreachable domain usually means the DNS A record has not propagated yet (wait a few minutes and retry), or the cert-manager certificate has not issued:

```bash
kubectl describe certificate hello -n hello
kubectl get certificaterequest -n hello
```

<a id="checkpoint-5"></a>

> **Checkpoint:** Your first app is live. You have working hardware, a running Kubernetes cluster, a GitOps pipeline managing it, a tunnel exposing it, and a real app serving traffic at your own domain. Everything from here is adding more apps, more monitoring, and more customization on top of a working foundation.

---

### Adding subsequent apps

Once the first app is working, every new app follows the same pattern:

1. Create `apps/<your-app>/base/` with the four files above, modified for your app
2. Add an ArgoCD `Application` in `clusters/prod/`
3. Add it to `clusters/prod/kustomization.yaml`
4. If the app needs secrets, create them in-cluster first ([cluster-secrets.md](../03-gitops/cluster-secrets.md#application-secrets))
5. Add a Cloudflare tunnel hostname if it needs a public URL
6. Open a PR, CI gates validate everything, merge, ArgoCD syncs

### Adding a new public hostname to an existing app

1. Add the new hostname to your app's `Ingress` resource in git
2. Add the hostname in the Cloudflare tunnel dashboard pointing at the ingress controller
3. If it needs access protection, add a [Cloudflare Access](../05-extras/cloudflare-access.md) policy for it
4. Merge the PR; DNS propagates automatically via Cloudflare

**Next:** [Extras](../05-extras/)