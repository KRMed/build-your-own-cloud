## Install Helm

**Required if you are using Helm charts** (ingress-nginx, kube-prometheus-stack). Can be skipped if you are deploying only plain Kubernetes manifests.

Helm is the package manager for Kubernetes. Rather than writing every Kubernetes resource from scratch, Helm charts give you a configurable template for complex applications. Install it on your local machine.

> **Reference:** [Installing Helm](https://helm.sh/docs/intro/install/)

**Linux:**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

> Downloading the script before running it lets you inspect it first. Avoid piping directly to `bash` from the internet.

**macOS:**

```bash
brew install helm
```

**Windows:**

```powershell
winget install Helm.Helm
```

Verify:

```bash
helm version
```

**Next:** [Expose Your Services](../02-expose-services/)
