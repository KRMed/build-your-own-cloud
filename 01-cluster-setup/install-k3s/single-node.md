### Single-node setup

SSH into your node and run the install script. The `--tls-san` flag adds your node's IP to the TLS certificate that K3S generates for the API server. Without it, kubectl connections from any machine other than the node itself will be rejected with a TLS error because the cert will only be valid for `127.0.0.1`.

```bash
curl -sfL https://get.k3s.io | sh -s - --tls-san <NODE_IP>
```

If your node has a hostname you want to use instead of or in addition to the IP:

```bash
curl -sfL https://get.k3s.io | sh -s - --tls-san <NODE_IP> --tls-san <NODE_HOSTNAME>
```

> **Reference:** [`--tls-san` flag documentation](https://docs.k3s.io/cli/server#listeners)

> **If you plan to use ingress-nginx (as this guide does):** K3S ships with Traefik as its built-in ingress controller and ServiceLB (Klipper) as its built-in load balancer. Running Traefik alongside ingress-nginx wastes resources and can cause confusion. Pass `--disable traefik` to remove it. If you are following **Option A (Cloudflare Tunnel)** or **Option C (LAN-only)** where ingress-nginx runs as `NodePort`, also pass `--disable servicelb`, since Klipper has nothing to do in those setups:
>
> ```bash
> # Option A or C: tunnel or LAN-only access (NodePort ingress)
> curl -sfL https://get.k3s.io | sh -s - --tls-san <NODE_IP> --disable traefik --disable servicelb
> ```
>
> If you chose **Option B (LoadBalancer)** and want Klipper to bind ports 80/443 on the node, omit `--disable servicelb` and only pass `--disable traefik`.
>
> **Reference:** [K3S: Disabling embedded components](https://docs.k3s.io/networking/basic-network-options#disabling-the-embedded-components)

This installs K3S as a systemd service that starts on boot. Verify it is running:

```bash
sudo systemctl status k3s
sudo kubectl get nodes
```

The node should show `Ready` within 30-60 seconds. If it stays `NotReady`, check the logs:

```bash
sudo journalctl -u k3s -f
```

<a id="checkpoint-1"></a>

> **Checkpoint:** Your cluster is running. K3S has started containerd (the container runtime that pulls and runs images), CoreDNS (internal DNS so pods can find each other by name), flannel (the network overlay that routes traffic between pods and nodes), and the local-path storage provisioner. The API server is live on port 6443. Every app you deploy from this point runs on top of this.

**Next:** [../configure-kubectl.md](../configure-kubectl.md)