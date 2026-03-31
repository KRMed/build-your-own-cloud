### High-availability setup (3 server nodes + embedded etcd)

**Use this instead of multi-node if you want true control plane fault tolerance.** Requires a minimum of 3 server nodes (etcd requires an odd number to maintain quorum): with 3 servers the cluster tolerates 1 failure, with 5 it tolerates 2.

> **Reference:** [K3S High Availability with Embedded etcd](https://docs.k3s.io/datastore/ha-embedded)

Before starting, choose a shared token; all nodes use this to authenticate to each other. Pick something long and random:

```bash
openssl rand -hex 32
```

Save this value. You will use it as `K3S_TOKEN` in every command below.

**On the first server node only:**

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<YOUR_TOKEN> sh -s - server \
  --cluster-init \
  --tls-san=<FIXED_IP>
```

> Add `--disable traefik --disable servicelb` to this command (and to each additional server command below) if applicable; the same rule from Section 3.1 applies here.

The `--cluster-init` flag bootstraps the embedded etcd cluster. `<FIXED_IP>` should be a stable address agents and your local kubectl can always reach, ideally a load balancer IP if you have one. In a home lab without a load balancer, use the first server node's static IP and add additional `--tls-san` flags for each other server IP so the certificate is valid from all of them.

Wait for the first server to show `Ready` before continuing:

```bash
sudo kubectl get nodes
```

**On each additional server node (2nd and 3rd server, not agents):**

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<YOUR_TOKEN> sh -s - server \
  --server https://<SERVER1_IP>:6443 \
  --tls-san=<FIXED_IP>
```

**On each agent node (workers only):**

```bash
curl -sfL https://get.k3s.io | K3S_TOKEN=<YOUR_TOKEN> sh -s - agent \
  --server https://<SERVER1_IP>:6443
```

Agents can point at any server node's IP. Verify all nodes have joined:

```bash
sudo kubectl get nodes
```

Server nodes will show the roles `control-plane,etcd,master`. Agent nodes show `<none>`.

> **Firewall note:** HA server nodes require etcd ports 2379–2380 open between server nodes only. This is already covered as a conditional step in Section 2.4.

**Next:** [../configure-kubectl.md](../configure-kubectl.md)
