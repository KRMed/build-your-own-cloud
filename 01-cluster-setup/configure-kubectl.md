## Configure kubectl on Your Local Machine

**Required** - this is how you interact with the cluster from your own machine without SSHing in every time.

> **Single-node users:** K3S installs kubectl directly on the node and it is available as `sudo kubectl` without any extra setup. If you only ever manage the cluster from the node itself, you can skip the kubeconfig copy steps below and use `sudo kubectl` on the node directly. If you want to manage the cluster from a separate laptop or desktop, continue with this section.

> **Reference:** [Install kubectl - Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/) | [macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) | [Windows](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

### Install kubectl

**Linux (amd64)** - for x86 laptops, desktops, and VMs:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**Linux (arm64)** - if your local machine is a Raspberry Pi or any other arm64 device:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/arm64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

**macOS:**

```bash
brew install kubectl
```

**Windows:**

```powershell
winget install Kubernetes.kubectl
```

### Copy the kubeconfig from your server node

```bash
mkdir -p ~/.kube
scp <YOUR_USERNAME>@<SERVER_IP>:/etc/rancher/k3s/k3s.yaml ~/.kube/config
```

> **Windows users:** `scp` is available natively in Windows Terminal and PowerShell on Windows 10 (build 1809) and later via the built-in OpenSSH client. If you are on an older system or prefer a GUI, [WinSCP](https://winscp.net/) is a free alternative.

> If you are unsure of your Pi's username, run `whoami` while SSH'd into the node.


The copied file has `127.0.0.1` as the server address. Replace it with your node's actual IP:

**Linux:**

```bash
sed -i 's/127.0.0.1/<SERVER_IP>/g' ~/.kube/config
```

**macOS** - the `-i` flag on macOS requires an explicit backup extension argument (even an empty one):

```bash
sed -i '' 's/127.0.0.1/<SERVER_IP>/g' ~/.kube/config
```

**Windows** - open `%USERPROFILE%\.kube\config` in any text editor (Notepad works) and manually replace `127.0.0.1` with your server's IP address.

Test that it works:

```bash
kubectl get nodes
kubectl get pods -A
```

You should see K3S system pods running in `kube-system`. If you get a TLS error here, double-check that you passed `--tls-san <NODE_IP>` during [K3S install](install-k3s/).

<a id="checkpoint-2"></a>

> **Checkpoint:** You are now talking to the cluster from your own machine. Every `kubectl` command you run hits the K3S API server on port 6443, authenticates using the certificate in your kubeconfig, then reads or writes cluster state. You will not need to SSH into the node again for routine cluster work.

### Managing multiple clusters

If you already have a kubeconfig for another cluster, merge them rather than overwriting. See the [kubectl kubeconfig merge docs](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/#merging-kubeconfig-files) for how to combine multiple cluster configs and switch between them with `kubectl config use-context`.

→ **Next:** [Install Helm](install-helm.md)