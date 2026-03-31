## What is GitOps and Why Use It

**Recommended. Can be skipped if you prefer to deploy apps manually with `kubectl apply` and do not want ArgoCD.**

Before setting up ArgoCD, understand what problem GitOps solves and why you might or might not want it.

### The problem with manual `kubectl apply`

When you deploy something by running `kubectl apply -f my-app.yaml` directly, that state lives only in the cluster. Over time, people make changes via `kubectl edit`, patch things in emergencies, or apply manifests that were never committed anywhere. The cluster drifts from whatever you thought it was running. When something breaks, you may not know what changed or when.

### What GitOps does instead

In a GitOps setup, your git repository is the single source of truth for everything running in the cluster. You never apply manifests manually - you commit changes to git, and a controller (in this case ArgoCD) watches the repository and reconciles the cluster to match it continuously.

The practical effects of this:

- **Drift prevention:** If someone manually edits a resource in the cluster, ArgoCD detects the diff and reverts it. The cluster always matches git
- **Auditability:** Every change has a git commit with an author, timestamp, and message
- **Recovery:** If a node is wiped or a namespace is accidentally deleted, ArgoCD will re-apply everything from git automatically
- **Rollback:** Rolling back is a git revert - the cluster follows

If you are just experimenting or running a single app, `kubectl apply` is simpler and perfectly valid - add ArgoCD later when you feel the pain of tracking state manually. ArgoCD adds meaningful overhead; on a constrained node (2–4GB RAM) verify you have headroom before adding the full monitoring stack alongside it. It also adds an abstraction layer that can slow down debugging early on.

> **Reference:** [ArgoCD Overview](https://argo-cd.readthedocs.io/en/stable/) | [OpenGitOps - What is GitOps?](https://opengitops.dev/)

→ **Next:** [Bootstrap ArgoCD](bootstrap-argocd.md)