### Multi-node setup (server + agents)

**Skip this section if you are running a single-node setup.** Your cluster from [single-node setup](single-node.md) is already complete; proceed to [configure kubectl](../configure-kubectl.md).

In a multi-node cluster, one node is the **server** (control plane) and the rest are **agents** (workers). Install the server first, then join agents one at a time.

> **Mixed architecture clusters:** If you have both arm64 and amd64 nodes, use your x86 (amd64) node as the server. The control plane runs etcd and the Kubernetes API server, which benefit from the stronger single-thread performance and typically higher RAM of x86 hardware. Your Raspberry Pi nodes join as agents and run workloads.

**On the server node:**

```bash
curl -sfL https://get.k3s.io | sh -s - --tls-san <SERVER_IP>
```

> Add `--disable traefik --disable servicelb` here if applicable; the same rule from [single-node setup](single-node.md) applies to multi-node servers.

After install, get the node token - agents need this to authenticate when joining:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

**On each agent node:**

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

Replace `<SERVER_IP>` with the server's static IP and `<NODE_TOKEN>` with the token from above.

Verify from the server node that all agents have joined:

```bash
sudo kubectl get nodes
```

All nodes should show `Ready`. Agent nodes show the role `<none>` - this is normal and just means no special role label has been assigned yet. You can assign labels and taints to control which workloads land where, but this is not required to get started.

> For high-availability control planes with multiple server nodes, see the [K3S HA documentation](https://docs.k3s.io/datastore/ha).

**Next:** [../configure-kubectl.md](../configure-kubectl.md)