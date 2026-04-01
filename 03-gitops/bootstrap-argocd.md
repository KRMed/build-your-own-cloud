## Bootstrap ArgoCD

**Required if you are using GitOps. Skip this section if you chose to manage deployments manually.**

ArgoCD watches a git repository and continuously reconciles the cluster state to match what is in git. You install it once manually - after that, it manages itself and everything else.

> **Before you start this section:** You need a git repository to point ArgoCD at. If you do not have one yet, create a new repository on [GitHub](https://github.com/new) or [GitLab](https://gitlab.com/projects/new) now. Set it to private. The directory structure ArgoCD expects is shown in [understanding-kustomize.md](understanding-kustomize.md); create those folders and files in your repo as you work through this section and [app-of-apps.md](app-of-apps.md).

> **Reference:** [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)

### Install ArgoCD

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> The ArgoCD docs note that using a pinned version (e.g. replacing `stable` with `v2.14.0`) is recommended for production. Check the [ArgoCD releases page](https://github.com/argoproj/argo-cd/releases) for the current stable version.

Watch the rollout:

```bash
kubectl rollout status deploy/argocd-server -n argocd --timeout=120s
```

### Install the ArgoCD CLI

Install this on your **local machine**, the same machine where you [configured kubectl](../01-cluster-setup/configure-kubectl.md). The CLI talks to the ArgoCD API server through the port-forward in the [Access ArgoCD for the first time](#access-argocd-for-the-first-time) section below.

> **Reference:** [ArgoCD CLI Installation](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

**Linux (amd64)** - for x86 laptops and desktops:

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd /usr/local/bin/argocd
```

**Linux (arm64)** - if your local machine is a Raspberry Pi or any other arm64 device (including Ubuntu on Pi):

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-arm64
sudo install -m 555 argocd /usr/local/bin/argocd
```

**macOS:**

```bash
brew install argocd
```

**Windows:** Download the binary from the [releases page](https://github.com/argoproj/argo-cd/releases/latest) (`argocd-windows-amd64.exe`) and add it to your PATH.

### Access ArgoCD for the first time

ArgoCD's ingress is not live yet - that comes after the GitOps sync in [app-of-apps.md](app-of-apps.md). Use port-forwarding to access the UI now:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open `https://localhost:8080` in your browser. Accept the self-signed cert warning - this is expected at this stage. The default username is `admin`.

Get the auto-generated initial password:

```bash
argocd admin initial-password -n argocd
```

Log in via CLI:

```bash
argocd login localhost:8080 \
  --username admin \
  --password <INITIAL_PASSWORD> \
  --insecure
```

> The `--insecure` flag skips TLS verification and is only acceptable here because you are connecting to localhost before a real certificate is provisioned. Do not use this flag once ArgoCD is behind a proper ingress with a valid certificate.

Change the password immediately:

```bash
argocd account update-password
```

**UI vs CLI:** Everything ArgoCD exposes through the CLI is also available in the web UI at `https://localhost:8080` (or `https://argocd.your-domain.com` once ingress is live). Use whichever you prefer; both are fully supported. The UI is particularly useful for visualizing app trees, inspecting resource diffs before a sync, and manually triggering syncs with one click. The CLI is more convenient for scripting and quick status checks. This guide uses CLI commands throughout because they're copy-pasteable, but every action shown has a UI equivalent.

> **Reference:** [ArgoCD Web UI overview](https://argo-cd.readthedocs.io/en/stable/getting_started/#opening-the-argo-cd-ui)

### Connect your git repository

If your infrastructure repo is private, ArgoCD needs credentials to clone it. SSH deploy keys are the recommended approach - read-only access scoped to a single repository.

```bash
# Generate a key pair on your local machine
ssh-keygen -t ed25519 -C "argocd-deploy-key" -f ~/.ssh/argocd-deploy-key -N ""
```

Add the **public key** as a deploy key in your repository:

- **GitHub:** Repository → Settings → Deploy keys → Add deploy key (read-only is sufficient)
- **GitLab:** Repository → Settings → Repository → Deploy keys
- **Gitea / self-hosted:** Repository → Settings → Deploy Keys

Register the **private key** with ArgoCD:

```bash
argocd repo add git@github.com:<YOUR_ORG>/<YOUR_REPO>.git \
  --ssh-private-key-path ~/.ssh/argocd-deploy-key
```

Verify the connection is successful:

```bash
argocd repo list
```

The status column should show `Successful`. If it shows `Failed`, check that the public key was added correctly to the repository and that the private key path is correct.

> **Troubleshooting repo connections:** ArgoCD supports SSH keys, HTTPS with username/password or tokens, and GitHub App authentication. If your connection is failing, the [ArgoCD private repositories documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/) covers all methods with troubleshooting steps for each. The most common failure causes are: wrong key format, the public key added to the wrong repository, or a firewall blocking outbound SSH (port 22) from the cluster, in which case switch to HTTPS.

**Next:** [App of Apps](app-of-apps.md)