# Cluster Setup

This section gets a working Kubernetes cluster onto your hardware and gives you full control of it from your local machine. By the end you'll have K3S running, kubectl working from your own machine, and Helm installed - everything the rest of the guide builds on.

**Before you start:** You need at least one machine running a 64-bit Linux OS. If you haven't chosen hardware or installed an OS yet, start with [hardware-and-os.md](hardware-and-os.md).


| Step | File | Notes |
|---|---|---|
| 1 | [Hardware and OS options](hardware-and-os.md) | Required |
| 2 | [Prepare your nodes](prepare-nodes.md) | Required - do this on every node before installing K3S |
| 3 | [Install K3S](install-k3s/) | Required - see install-k3s/ for the setup path that matches your node count |
| 4 | [Configure kubectl](configure-kubectl.md) | Required to manage the cluster from your local machine |
| 5 | [Install Helm](install-helm.md) | Required if using Helm charts (ingress-nginx, monitoring stack) |

<br>

> [!NOTE]
> Upgrading K3S later? See [install-k3s/upgrading.md](install-k3s/upgrading.md).
