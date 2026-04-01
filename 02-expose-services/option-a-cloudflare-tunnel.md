### Option A: Cloudflare Tunnel (no open ports, no public IP needed)

**Recommended for home networks and dynamic IPs.**

The traditional approach to exposing a home server is port forwarding - opening ports 80 and 443 on your router. This works, but it exposes your home IP address in DNS, requires managing firewall rules, and breaks when your ISP changes your IP.

A Cloudflare Tunnel takes a different approach: your cluster opens an outbound encrypted connection to Cloudflare's edge. All inbound traffic flows back through that tunnel. Your router never opens any ports. Your home IP is never visible in DNS. TLS terminates at Cloudflare before anything reaches your machine.

> **Reference:** [Create a tunnel (dashboard)](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/get-started/create-remote-tunnel/)

**Prerequisites:**

- A domain registered anywhere. It must be delegated to Cloudflare nameservers - this is free. See [Cloudflare DNS setup](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/) for how to point your domain's nameservers to Cloudflare
- A [Cloudflare account](https://www.cloudflare.com) with Zero Trust enabled (the free tier covers everything in this guide)

**Create the tunnel:**

1. Log in to the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com) and navigate to **Networks → Tunnels** (the exact menu path may shift as Cloudflare updates its UI; use the reference link above if you cannot find it)
2. Click **Create a tunnel** and select **Cloudflared**
3. Give it a name that identifies your cluster (e.g. `home-cluster`)
4. Cloudflare will display a connector token. **Copy and save this token** - it is the credential your cluster uses to authenticate to Cloudflare. You will create a Kubernetes Secret from it in [create in-cluster secrets](../03-gitops/cluster-secrets.md#cloudflare-tunnel-token)

**Add public hostnames:**

In the tunnel configuration, add a hostname entry for each service you want to expose. Point all of them at your ingress controller:

| Public hostname | Type | URL |
|---|---|---|
| `app.<your-domain>.com` | HTTP | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` |
| `argocd.<your-domain>.com` | HTTP | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` |
| `grafana.<your-domain>.com` | HTTP | `http://ingress-nginx-controller.ingress-nginx.svc.cluster.local` |

All hostnames point to the same backend - the ingress controller. NGINX reads the `Host` header on each incoming request and routes it to the correct service inside the cluster. This means the tunnel only needs to know about one backend, and all routing logic lives in your Kubernetes `Ingress` resources.

> [!NOTE]
> Use your domain’s actual top-level domain (TLD), which may not be .com (ex. .cloud, .org, .net)

> If you are not using ingress-nginx, set the URL to `http://<NODE_IP>:<NODEPORT>` for each service directly.

**How traffic flows end-to-end:**

```
Browser
  → Cloudflare Edge (TLS terminates here)
  → Cloudflared pod inside your cluster (outbound tunnel)
  NGINX Ingress Controller (routes by hostname)
  → ClusterIP Service
  → Your app pods
```

Public apps go straight through. Protected apps (ArgoCD, Grafana) get a Cloudflare Access policy added in [Cloudflare Access](../05-extras/cloudflare-access.md) that gates authentication at Cloudflare's edge before the request ever reaches your cluster.

**Deploy cloudflared in your cluster:**

The tunnel only becomes active when a `cloudflared` pod is running inside your cluster and connecting outbound to Cloudflare using the token from [cluster secrets](../03-gitops/cluster-secrets.md#cloudflare-tunnel-token). Create the following manifest at `infra/cloudflared/base/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest   # pin to a specific version in production and find tags at https://github.com/cloudflare/cloudflared/releases
          args:
            - tunnel
            - --no-autoupdate
            - run
          env:
            - name: TUNNEL_TOKEN
              valueFrom:
                secretKeyRef:
                  name: tunnel-token   # Secret from cluster-secrets.md
                  key: token
```

Running 2 replicas gives you redundancy: if one pod restarts, the tunnel stays active through the other. Two replicas is enough; Cloudflare load-balances across all connected connectors automatically.

You also need two more files to complete the `infra/cloudflared/base/` directory.

**`infra/cloudflared/base/namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cloudflared
```

**`infra/cloudflared/base/kustomization.yaml`**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - deployment.yaml
```

> **Reference:** [cloudflared Deployment - Kubernetes](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/configure-tunnels/tunnel-run-parameters/)

<a id="checkpoint-3"></a>

> **Checkpoint:** Once the tunnel shows **Active** in the Cloudflare dashboard, your cluster is reachable from the internet with no open ports on your router. Cloudflare owns the DNS record and terminates TLS at its edge. The cloudflared pod inside your cluster holds a persistent outbound connection to Cloudflare - all inbound traffic travels back through that connection. Your home IP never appears in DNS.

**Next:** [GitOps](../03-gitops/)
