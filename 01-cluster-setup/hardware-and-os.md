## Hardware and OS Options

**Required** - you need at least one machine to run a cluster on.

### What hardware works

K3S runs on any Linux machine with a 64-bit CPU (x86_64 or arm64) or 32-bit ARM (armhf). You are not limited to Raspberry Pi hardware. Common choices:

| Hardware | Architecture | Notes |
|---|---|---|
| Raspberry Pi 4 / 5 | arm64 | Great for low-power home clusters. 4GB+ RAM recommended for a monitoring stack |
| Any x86 mini PC (Intel NUC, Beelink, etc.) | amd64 | More headroom, usually cheaper per GB RAM than Pi at higher tiers |
| Old laptops or desktops | amd64 | Free if you have them. Disable sleep/suspend in the BIOS |
| VPS or cloud VM | amd64 or arm64 | Not bare-metal but K3S works fine - useful for testing before buying hardware |
| Mixed cluster | any combination | K3S supports heterogeneous nodes. In a mixed arm64/amd64 cluster, prefer the x86 node as the server (control plane) because it typically has more RAM and better single-thread performance for etcd. See the mixed cluster note below |

**Minimum hardware requirements** (from the [K3S documentation](https://docs.k3s.io/installation/requirements#hardware)):

| Node type | CPU | RAM |
|---|---|---|
| Server (control plane) | 2 cores | 2 GB |
| Agent (worker) | 1 core | 512 MB |

These are the minimums for K3S to run, not for the full stack described in this guide. A single node is perfectly viable on hardware that meets the server requirements: a Raspberry Pi 4 with 4GB RAM (4 cores, 4GB) runs the complete setup comfortably. If you plan to run the monitoring stack (Prometheus + Grafana + Alertmanager) alongside your apps, budget at least 4GB RAM on whichever node hosts those workloads. See the [K3S resource profiling page](https://docs.k3s.io/reference/resource-profiling) for measured results across different workload sizes.

> **K3S architecture support:** [docs.k3s.io/installation/requirements#architectures](https://docs.k3s.io/installation/requirements#architectures)

> **Mixed architecture clusters (arm64 + amd64):** K3S itself runs fine on heterogeneous nodes, but your container images must support every architecture present in the cluster, or you must pin workloads to nodes of the correct architecture using node labels and `nodeSelector` or `affinity` rules. If a pod is scheduled onto a node whose architecture does not match the image, it will fail with `ImagePullBackOff` or `exec format error`. Most major public images (nginx, Prometheus, Grafana, etc.) publish multi-arch manifests and work transparently. Your own application images must be built and pushed as multi-arch, see [Docker buildx](https://docs.docker.com/buildx/working-with-buildx/) for how to build for multiple architectures in a single push. If you're unable to build multi-arch images, use a `nodeSelector` to pin those workloads to nodes of the correct architecture:
> ```yaml
> spec:
>   nodeSelector:
>     kubernetes.io/arch: amd64   # or arm64
> ```

### How many nodes

**One node** is enough to run the full stack described here. A single node acts as both the control plane and the workload runner. The monitoring stack (Prometheus + Grafana + Alertmanager) is the most resource-intensive part with at least 3-4GB RAM minimum for a comfortable single-node setup

**Multiple nodes** give you fault tolerance and let you spread workloads. In a multi-node K3S setup, you designate one or more nodes as **servers or control nodes** (they run the control plane) and the rest as **agents or worker nodes** (they run workloads only). Adding nodes is covered in [multi-node setup](install-k3s/multi-node.md)

For most home lab setups, **one server node is enough**. The control plane is a single point of failure but in practice it only affects scheduling new workloads, not the running ones. If you have 3 or more nodes and care about the cluster staying fully operational through a node failure, consider running **3 server nodes** with embedded etcd (the minimum for a healthy HA quorum) and use any additional nodes as agents.

**Use an odd number of server nodes.** K3S HA uses a Raft consensus algorithm that requires a quorum (majority agreement) to elect a leader and make decisions. An odd number of nodes makes quorum calculations straightforward and prevents split-brain scenarios during network partitions or node failures. With 2 nodes, losing one means you immediately lose quorum. With 3 nodes, you can lose 1 and keep the cluster healthy. With 4 nodes, you can also only lose 1, so the extra node gives no fault tolerance improvement over 3; jump straight to 5 if you want to tolerate 2 failures.

K3S supports two HA datastore options. Which one you use depends on your setup:
- **Embedded etcd** (recommended for most home labs): K3S bundles etcd directly. No external database needed. See [K3S HA with embedded etcd](https://docs.k3s.io/datastore/ha-embedded)
- **External datastore** (MySQL, PostgreSQL, or etcd): For setups where you already have or want a separate database. See [K3S HA with an external datastore](https://docs.k3s.io/datastore/ha)

If you are unsure, use embedded etcd. The setup steps in [high-availability setup](install-k3s/ha-setup.md) use embedded etcd.

### Choosing an OS

Any modern 64-bit Linux distribution works. Tested and commonly used options:

- **Raspberry Pi OS Lite (64-bit)** - for Pi hardware only. Use the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to flash it
- **Ubuntu Server 22.04 LTS or 24.04 LTS** - works on any hardware, well-documented, good K3S compatibility
- **Debian 12** - minimal, stable, works well for always-on nodes

The rest of this guide uses generic Linux commands that work on any of these. OS-specific steps are called out where they exist

K3S itself runs on any modern 64-bit Linux distribution: Fedora, Rocky Linux, Alma Linux, Arch, openSUSE, and others all work. However, the OS-specific guidance in this guide (firewall, static IP, boot parameters) explicitly covers only Ubuntu, Debian, and Raspberry Pi OS. If you are on a different distro, you will need to translate those steps yourself. The [K3S installation requirements](https://docs.k3s.io/installation/requirements) page covers known distro-specific considerations

**Next:** [prepare-nodes.md](prepare-nodes.md)