### Option B: Direct public IP or home router port forwarding

**Choose this option if:** your machine has a static public IP (any cloud VM, dedicated server, or colocation setup), **or** you are on a home network and are comfortable opening ports 80 and 443 on your router.

If you are doing this, you can skip [Option A (Cloudflare Tunnel)](option-a-cloudflare-tunnel.md) and [Cloudflare Access](../05-extras/cloudflare-access.md) entirely.

#### Option B1 - Machine with a direct static public IP

1. Point your domain's DNS A record at the machine's public IP. You can use Cloudflare as your DNS provider (free, recommended for the CDN and DDoS protection) and just add an A record there instead of creating a tunnel

> **Important: you will repeat this for every subdomain.** Unlike Cloudflare Tunnel (which handles DNS automatically), Option B requires you to manually add a DNS A record pointing to your public IP for every subdomain you expose throughout this guide: Grafana, Prometheus, Alertmanager, ArgoCD, each application, and any other service you add later. They can all point to the same IP, but each hostname needs its own record.

2. Configure ingress-nginx to use `type: LoadBalancer` in your Helm values. K3S ships with a built-in load balancer called [Klipper](https://github.com/k3s-io/klipper-lb) that will bind ports 80 and 443 directly on the node

```yaml
# ingress-nginx Helm values for LoadBalancer mode
controller:
  service:
    type: LoadBalancer
```

#### Option B2 - Home router port forwarding

Your router has the public IP, not your nodes. The setup is the same as B1 but with two extra considerations:

1. **Dynamic IP:** Most home ISPs assign a dynamic public IP that changes periodically. If your IP changes, your DNS record breaks. Options:
   - Ask your ISP for a static IP (sometimes available for a fee)
   - Use a DDNS service like [DuckDNS](https://www.duckdns.org/) (free) or Cloudflare's free DDNS to automatically update your DNS record when your IP changes. If you use Cloudflare as your DNS provider you can run a small agent that updates the A record automatically; see [Cloudflare DDNS](https://developers.cloudflare.com/dns/manage-dns-records/how-to/managing-dynamic-ip-addresses/)

2. **Port forwarding rules:** In your router's admin panel, forward inbound ports 80 and 443 (TCP) to the LAN IP of the node running the ingress-nginx LoadBalancer service. With K3S Klipper, the ingress-nginx service binds to all nodes; pick the server node's LAN IP or whichever node you want as the entry point

   Point your domain's DNS A record at your **router's public IP** (not the node's LAN IP)

ingress-nginx Helm values are the same as B1; use `type: LoadBalancer`.

#### TLS certificates (required for both B1 and B2)

Without Cloudflare terminating TLS for you, your cluster needs to provision its own certificates. Use [cert-manager](https://cert-manager.io/docs/) with Let's Encrypt.

> **Important: port 80 must be open.** Let's Encrypt's HTTP-01 challenge verifies domain ownership by making an HTTP request to port 80 on your domain. If port 80 is blocked (by UFW, your provider's security group, or your router) certificate issuance will silently hang. Open port 80 even if you plan to serve all traffic on 443. cert-manager will use it only for the challenge and ingress-nginx will redirect HTTP to HTTPS for everything else.

1. Install cert-manager as an ArgoCD Application. Create `clusters/prod/cert-manager.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # sync after ingress-nginx, before apps that need TLS
spec:
  project: default
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: "<CHART_VERSION>"   # find the latest at https://github.com/cert-manager/cert-manager/releases, use the chart version (e.g. "v1.16.2")
    helm:
      valuesObject:
        installCRDs: true
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Add `cert-manager.yaml` to your `clusters/prod/kustomization.yaml` under `resources`.

2. Create a `ClusterIssuer` resource pointing at Let's Encrypt:
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: <YOUR_EMAIL>
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

3. Add an annotation to each `Ingress` resource to trigger automatic certificate provisioning:
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - app.your-domain.com
      secretName: app-tls
```

> **Reference:** [cert-manager Getting Started](https://cert-manager.io/docs/getting-started/) | [cert-manager NGINX Ingress tutorial](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/)

**Why you might still want a Cloudflare tunnel even with a public IP:** WAF, DDoS mitigation, and hiding your home IP in DNS. But it is not required.

**Next:** [GitOps](../03-gitops/)