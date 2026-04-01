# Multi-node vs High-Availability (HA)

If you are choosing between multi-node and HA, this page explains the practical difference before you commit to a setup.

> **References:** [K3S Architecture](https://docs.k3s.io/architecture) | [K3S HA with embedded etcd](https://docs.k3s.io/datastore/ha-embedded)

---

## What actually runs on each node type

**Server nodes** run the Kubernetes control plane: the API server, scheduler, controller manager, and the datastore (either SQLite for single-node or etcd for HA). They also run workloads by default - unlike standard Kubernetes where control plane nodes are tainted, K3S server nodes are schedulable unless you explicitly add a taint.

**Agent nodes** run only the kubelet and container runtime. No control plane components, no datastore. They connect to the server via a websocket with a built-in client-side load balancer, which means in an HA setup agents tolerate individual server failures without dropping existing connections.

---

## Multi-node (1 server + agents)

One server runs the control plane. Additional nodes are agents that run workloads only.

- Good default when you have 2+ machines and do not need strict uptime
- Lets you spread workloads and avoid resource contention on the control plane node
- The server is a single point of failure for the control plane

**What actually happens when the server goes down:**

- Running pods on agent nodes keep running - existing workloads are not interrupted
- New pods cannot be scheduled - the scheduler is on the server
- `kubectl` stops working - the API server is on the server
- The cluster recovers automatically when the server comes back online

> **Reference:** [K3S Architecture - Server nodes](https://docs.k3s.io/architecture#k3s-server-nodes)

---

## High-Availability (3+ server nodes with embedded etcd)

Three or more server nodes each run both the control plane and a member of an embedded etcd cluster. Agents connect to all servers through the built-in load balancer.

- Removes the control plane single point of failure
- Requires a minimum of 3 server nodes - etcd needs an odd number to maintain quorum
- More hardware and more operational surface than multi-node

**Quorum and fault tolerance:**

etcd uses a Raft consensus algorithm. A cluster can tolerate failures as long as a majority (quorum) of nodes are healthy. The formula is `floor(n/2) + 1` nodes needed for quorum:

| Server nodes | Quorum needed | Failures tolerated |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Always use an odd number of server nodes.** With an even number (e.g. 4), you tolerate the same number of failures as the odd number below it (3) but with more hardware. Jump from 3 to 5 if you need to tolerate 2 failures.

> **Reference:** [K3S HA with embedded etcd](https://docs.k3s.io/datastore/ha-embedded)

---

## Can I migrate from multi-node to HA later?

Yes. Restart the existing server node with `--cluster-init` to convert it to embedded etcd, then join additional servers. You do not need to rebuild the cluster.

> **Reference:** [Migrating from external datastore to embedded etcd](https://docs.k3s.io/datastore/ha-embedded#existing-clusters)

---

## Quick decision

- **Multi-node** - learning, limited hardware, or downtime during server failures is acceptable
- **HA** - you need the control plane to survive a node failure and can run at least 3 server nodes

---

**Back to:** [single-node.md](single-node.md) | [multi-node.md](multi-node.md) | [ha-setup.md](ha-setup.md)
