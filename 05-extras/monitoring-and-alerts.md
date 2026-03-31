## Monitoring and Alerting

**Recommended. Can be skipped if you do not need visibility into cluster health.**

The monitoring stack is `kube-prometheus-stack`, a Helm chart that bundles Prometheus (metrics storage), Grafana (dashboards), and Alertmanager (alert routing) into a single deployable unit. It is a large chart and takes a few minutes to pull on first deploy, but it gives you production-grade observability with minimal configuration.

> **Reference:** [kube-prometheus-stack on GitHub](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

**Skip this section if:** you are resource-constrained (in practice the stack typically uses approximately 800MB–1.5GB RAM on a lightly loaded cluster, though this varies and no official figure is provided by the chart), you are running a quick experiment, or you plan to use an external monitoring service instead.

### ArgoCD Application for kube-prometheus-stack

Create this in `clusters/prod/monitoring.yaml`. The `valuesObject` below wires in the Grafana admin secret you created in [cluster-secrets.md](../03-gitops/cluster-secrets.md#monitoring-credentials) and sets up persistent storage for Alertmanager via the K3S local-path provisioner.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "2"
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "<CHART_VERSION>"   # find the latest at https://github.com/prometheus-community/helm-charts/releases, use the chart version (e.g. "69.3.3"), not the app version
    helm:
      valuesObject:
        grafana:
          admin:
            existingSecret: grafana-admin-credentials   # Secret from cluster-secrets.md
            userKey: admin-user
            passwordKey: admin-password
          ingress:
            enabled: true
            ingressClassName: nginx
            hosts:
              - grafana.your-domain.com
        prometheus:
          prometheusSpec:
            retention: 15d
          ingress:
            enabled: true
            ingressClassName: nginx
            hosts:
              - prometheus.your-domain.com
        alertmanager:
          ingress:
            enabled: true
            ingressClassName: nginx
            hosts:
              - alertmanager.your-domain.com
          alertmanagerSpec:
            storage:
              volumeClaimTemplate:
                spec:
                  storageClassName: local-path
                  accessModes: ["ReadWriteOnce"]
                  resources:
                    requests:
                      storage: 5Gi
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true   # required: kube-prometheus-stack CRDs are too large for client-side apply
```

> **`ServerSideApply=true`** is required because several kube-prometheus-stack CRDs exceed the annotation size limit that ArgoCD uses in client-side apply mode. Without it, sync will fail with a "metadata annotations too long" error. This is a known limitation documented in the [kube-prometheus-stack chart README](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#multiple-releases).
>
> Replace the hostnames above with your actual subdomains. If you are protecting these with [Cloudflare Access](cloudflare-access.md), do not add any auth annotations here; the protection happens at Cloudflare's edge, not the ingress.

### What gets scraped automatically

kube-prometheus-stack ships with built-in scrape configurations for:

- Node metrics (CPU, memory, disk, network) via `node-exporter`
- Kubernetes API server, scheduler, and controller-manager
- Container and pod metrics via `kube-state-metrics`

To scrape your own apps or add-ons (ArgoCD, ingress-nginx), create a `ServiceMonitor` resource. The `release` label on the ServiceMonitor must match the Helm release name of kube-prometheus-stack - this is the most common reason scrape targets are not picked up.

> **Reference:** [Prometheus Operator - ServiceMonitor](https://prometheus-operator.dev/docs/operator/design/#servicemonitor)

### Verify scrape targets

Once Prometheus is running, go to `https://prometheus.your-domain.com → Status → Targets`. Every target should show `UP`. A `DOWN` target means either the service is not running, the ServiceMonitor label selector does not match, or the service port name is wrong.

### Custom alert rules

Add your own alert rules in your `kube-prometheus-stack` Helm values file:

```yaml
additionalPrometheusRulesMap:
  custom-rules:
    groups:
      - name: availability
        rules:
          - alert: AppDown
            expr: up{job="my-app"} == 0
            for: 2m
            labels:
              severity: critical
            annotations:
              summary: "App has been down for 2 minutes"
```

### Push notifications

**Optional** - Alertmanager can route firing alerts to your phone. Alertmanager speaks a webhook format that most notification services don't accept directly, so you typically need a small relay. A common setup is [ntfy](https://ntfy.sh/) (free hosted tier, iOS/Android app) with [ntfy-alertmanager](https://github.com/pinpox/ntfy-alertmanager) as the bridge.

Deploy ntfy-alertmanager in your cluster (see the [ntfy-alertmanager deployment manifest](https://github.com/pinpox/ntfy-alertmanager)). Then wire Alertmanager to it by adding a receiver to your kube-prometheus-stack `valuesObject`:

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: ntfy
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
    receivers:
      - name: ntfy
        webhook_configs:
          - url: http://ntfy-alertmanager.monitoring.svc.cluster.local:8080/hook
            send_resolved: true
```

The URL assumes ntfy-alertmanager is deployed in the `monitoring` namespace on port 8080; adjust if your deployment differs. Set your ntfy topic and server URL in the ntfy-alertmanager config map per the project's README.

> **Reference:** [ntfy-alertmanager](https://github.com/pinpox/ntfy-alertmanager) | [ntfy docs](https://docs.ntfy.sh/)