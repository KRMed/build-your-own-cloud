### Option C: LAN-only access (no public exposure)

**Valid choice if you only need to access your services from inside your home network.**

This is the simplest option and requires no Cloudflare account, no domain, and no open ports anywhere. Your services are only reachable from devices on the same network as the cluster.

Skip [Option A (Cloudflare Tunnel)](option-a-cloudflare-tunnel.md), [Cloudflare Access](../05-extras/cloudflare-access.md), and any Cloudflare-specific steps throughout this guide.

To reach your services from other devices on your LAN, you have two options:

**Option C1 - Access via NodePort directly:**

K3S assigns a NodePort (a port in the 30000-32767 range) to every service exposed that way. You can reach a service at `http://<NODE_IP>:<NODEPORT>` from any device on your network. Not pretty, but it works immediately with no additional setup.

```bash
# Find the NodePort assigned to a service
kubectl get svc -n <NAMESPACE>
```

**Option C2 - Local DNS with ingress (cleaner):**

Install ingress-nginx as in the full setup. Then configure your home router's DNS or a local DNS resolver like [Pi-hole](https://pi-hole.net/) or [AdGuard Home](https://adguard.com/en/adguard-home/overview.html) to resolve your chosen hostnames (e.g. `app.home.local`) to your node's LAN IP. This gives you clean URLs without port numbers.

> **Reference:** [Pi-hole Local DNS Records](https://docs.pi-hole.net/ftldns/dns-records/)

**Next:** [GitOps](../03-gitops/)
