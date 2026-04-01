# Stop Paying for Hosting. Build Your Own Cloud Instead.

Maybe you're tired of paying $20/month for a VPS to host a side project. Maybe you want a real environment to deploy things to instead of everything living on localhost. Maybe you have a spare Raspberry Pi collecting dust and want to finally do something cool with it. Or maybe you just want to understand how production infrastructure actually works by building it from scratch instead of reading a blog post about it

This guide walks you through all of it, starting from bare hardware and ending with a self-hosted Kubernetes cluster with GitOps, monitoring, automated CI/CD, and zero open ports on your router

This covers most of what went into building my personal cloud project firsthand. Use it to get something running, then go read the official docs, experiment, break things, and learn from them. Every reference link throughout the guide is a good next step or starting point to learn from

**Beginners are welcome.** No prior experience with any of these technologies is required.

<div align="center">

**What you'll be working with**

[![K3S](https://img.shields.io/badge/K3S-Kubernetes-FFC61C?style=flat-square&logo=k3s&logoColor=white)](https://k3s.io)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-GitOps-EF7B4D?style=flat-square&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/en/stable/)
[![Helm](https://img.shields.io/badge/Helm-Package_Manager-0F1689?style=flat-square&logo=helm&logoColor=white)](https://helm.sh)
[![Kustomize](https://img.shields.io/badge/Kustomize-Config-326CE5?style=flat-square&logo=kubernetes&logoColor=white)](https://kustomize.io)
[![NGINX](https://img.shields.io/badge/NGINX-Ingress-009639?style=flat-square&logo=nginx&logoColor=white)](https://kubernetes.github.io/ingress-nginx/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel_&_Access-F38020?style=flat-square&logo=cloudflare&logoColor=white)](https://developers.cloudflare.com/cloudflare-one/)
[![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C?style=flat-square&logo=prometheus&logoColor=white)](https://prometheus.io)
[![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800?style=flat-square&logo=grafana&logoColor=white)](https://grafana.com)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=flat-square&logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![Trivy](https://img.shields.io/badge/Trivy-Security-1904DA?style=flat-square&logo=aquasecurity&logoColor=white)](https://aquasecurity.github.io/trivy)
[![gitleaks](https://img.shields.io/badge/gitleaks-Secret_Scanning-FF1F1F?style=flat-square&logo=git&logoColor=white)](https://github.com/gitleaks/gitleaks)
[![ntfy](https://img.shields.io/badge/ntfy-Alerting-338DF7?style=flat-square&logo=ntfy&logoColor=white)](https://ntfy.sh)

</div>

---

## Checkpoints

| | Folder | What's inside | After this section |
|---|---|---|---|
| 🟢 | [01-Cluster Setup (Start here)](01-cluster-setup/) | Hardware options, OS prep, K3S install (single-node, multi-node, HA), kubectl, Helm | K3S running, kubectl working from your local machine |
| 🟢 | [02-Expose Services](02-expose-services/) | Cloudflare Tunnel, public IP/VPS, and LAN-only access options | Services reachable from your LAN or the internet |
| 🟢 | [03-Gitops](03-gitops/) | Kustomize, cluster secrets, ArgoCD bootstrap, app-of-apps pattern | ArgoCD managing your cluster - commit to git, the cluster follows |
| 🟢 | [04-Deploy](04-deploy/) | Writing app manifests, deploying your first app end-to-end | A real app live in a browser at your own domain |
| 🟢 | [05-Extras](05-extras/) | Cloudflare Access, monitoring & alerting, persistent storage, CI/CD pipelines | Auth, observability, and CI gates in place |
| | [references](references/) | Troubleshooting, useful tools, resetting your cluster, reference links | |

---

## Feedback and Contributions

Found something wrong, outdated, or confusing? Have a suggestion for what should be added or changed?

Open an issue or start a discussion on the [GitHub repository](https://github.com/KRMed/build-your-own-cloud/issues): corrections, improvements, and questions are all welcome. If you built something based on this guide and ran into a gap, that feedback is especially useful

You can also reach me directly at [guide-support@krmed.cloud](mailto:guide-support@krmed.cloud)
