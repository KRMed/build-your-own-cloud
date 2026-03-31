# Extras

This section covers everything that makes a cluster production-quality: locking down public services, visibility into what's running, persistent storage for stateful apps, and CI gates that catch problems before they reach the cluster.

None of these are required to have a working cluster, but each one addresses a real gap you will notice quickly once your first app is running.

---

| File | When to use it |
|---|---|
| [Cloudflare Access](cloudflare-access.md) | You are using Option A (Cloudflare Tunnel) and want to put an auth wall in front of internal tools like ArgoCD and Grafana |
| [Monitoring and alerting](monitoring-and-alerts.md) | You want visibility into cluster health, resource usage, and alerts when something goes wrong |
| [Persistent storage](persistent-storage.md) | Any app that writes data to disk - databases, file stores, anything stateful |
| [CI/CD pipelines](cicd-pipelines.md) | You want to catch broken manifests or leaked secrets before they are committed to git |

---

> [!TIP]
> If you are resource-constrained on a single node, add these in order of impact: Cloudflare Access first (no resource cost), then CI/CD (runs on GitHub, not your cluster), then persistent storage (only needed if you have stateful apps), then monitoring last (the heaviest - typically 800MB to 1.5GB RAM).
