## 20. Troubleshooting

### Pod issues

| Symptom | Cause | Fix |
|---|---|---|
| `CreateContainerConfigError` | Referenced Secret does not exist | `kubectl describe pod <pod> -n <ns>` - find the Secret name and create it following Step 8 |
| `ImagePullBackOff` | Missing registry pull secret or wrong image tag | Create `registry-credentials` secret in the pod's namespace (Step 8.3), or verify the image tag exists |
| `ImagePullBackOff` or `exec format error` on specific nodes only | Image is single-arch and was scheduled onto a node of the wrong architecture | Add a `nodeSelector: kubernetes.io/arch: <arch>` to pin the pod to the correct node type, or rebuild the image as a multi-arch manifest using `docker buildx` |
| `CrashLoopBackOff` | App is starting and crashing | `kubectl logs <pod> -n <ns>` and `kubectl logs <pod> -n <ns> --previous` to see the error |
| Pod stuck `Pending` | Not enough CPU or memory on any node | `kubectl describe pod <pod>` - check the Events section for "Insufficient memory" or "Insufficient cpu" |
| `TLS certificate signed by unknown authority` | `--tls-san` was not passed during K3S install | Reinstall K3S with the flag: `curl -sfL https://get.k3s.io \| sh -s - --tls-san <NODE_IP>` |

### K3S and node issues

| Symptom | Cause | Fix |
|---|---|---|
| `kubectl` connection refused from local machine | Firewall blocking port 6443 | Open port 6443 on the node (Step 2.4) |
| Agent node not joining | Firewall blocking flannel (UDP 8472) | Open port 8472/udp between all nodes (Step 2.4) |
| Node shows `NotReady` | K3S process crashed or failed to start | `sudo journalctl -u k3s -f` on the node to see the error |
| `kubectl logs` or `exec` fails | Firewall blocking port 10250 | Open port 10250/tcp (Step 2.4) |
| K3S fails to start or resource limits are silently ignored | Memory cgroups not enabled (common on Raspberry Pi) | Add `cgroup_memory=1 cgroup_enable=memory` to `/boot/firmware/cmdline.txt` and reboot (Step 1) |
| Cluster stops working after a node IP change | All TLS certificates and kubeconfig entries reference the old IP | Set a static IP on each node before installing K3S. To recover, uninstall and reinstall K3S with the new IP (see Resetting) |

### ArgoCD issues

| Symptom | Cause | Fix |
|---|---|---|
| App stuck `Progressing` | Helm chart slow to download, or CRDs not ready | Wait 3-5 min; check `kubectl get events -n <ns> --sort-by='.lastTimestamp'` |
| App shows `OutOfSync` immediately after sync | A controller is modifying the resource after ArgoCD sets it | Add an `ignoreDifferences` entry for that field in the Application spec |
| `ComparisonError` | ArgoCD cannot read a CRD it needs | Make sure the CRD-installing app (e.g. ingress-nginx) syncs before apps that use those CRDs |
| Repo shows `Failed` in `argocd repo list` | SSH key not added to the repository, wrong key path, or outbound SSH (port 22) is blocked by your firewall | Re-run the `argocd repo add` command with the correct key; if the key is correct, check that outbound port 22 is not blocked on the node |

### Ingress and routing

| Symptom | Cause | Fix |
|---|---|---|
| Hostname returns 404 from nginx | `ingressClassName: nginx` missing from Ingress spec, or ingress-nginx is not set as the default IngressClass | Add `ingressClassName: nginx` to the Ingress spec, or verify `ingressClassResource.default: true` is set in the ingress-nginx Helm values |
| Hostname returns 503 Service Unavailable | Ingress is routing but the backend Service or pod is not healthy | `kubectl get pods -n <ns>` and `kubectl describe svc <svc> -n <ns>`; check the Endpoints field on the service shows pod IPs |
| Hostname unreachable but pod and service are healthy | Ingress resource was not picked up by nginx | `kubectl describe ingress <name> -n <ns>`; check the `Address` field is populated and `Events` shows no errors |

### cert-manager (Option B users)

| Symptom | Cause | Fix |
|---|---|---|
| Certificate stuck in `False` / `Issuing` | ACME HTTP-01 challenge cannot reach the challenge URL | Verify port 80 is open and forwarded to your node, and that the Ingress is routing `/.well-known/acme-challenge/` correctly. `kubectl describe certificate <name> -n <ns>` shows the exact failure reason |
| `CertificateRequest` shows `InvalidDNSNames` | Domain in the cert does not match the Ingress `host` | Make sure the `spec.commonName` or `spec.dnsNames` in the Certificate resource matches the hostname in your Ingress exactly |
| Rate limit error from Let's Encrypt | Too many certificate requests for the same domain | Let's Encrypt enforces [rate limits](https://letsencrypt.org/docs/rate-limits/). Switch your `ClusterIssuer` to the staging server (`acme-staging-v02.api.letsencrypt.org`) for testing, then switch back to production once everything works |

### Cloudflare tunnel

| Symptom | Cause | Fix |
|---|---|---|
| Tunnel shows `Inactive` in dashboard | `tunnel-token` Secret missing or incorrect | Delete and recreate the Secret with the correct token value |
| 502 Bad Gateway | Tunnel is up but backend URL is wrong | Check the tunnel's public hostname config - the backend should point at the ingress controller with `http://` scheme |
| Cloudflare error 1033 | Tunnel connector is not running | `kubectl logs -n cloudflared -l app=cloudflared` |

### Monitoring

| Symptom | Cause | Fix |
|---|---|---|
| Grafana shows no data | ServiceMonitor not being picked up | Check that the `release` label on the ServiceMonitor matches the Helm release name of kube-prometheus-stack |
| Prometheus target `DOWN` | Service not running, or port name mismatch | Verify the service is running and the port name in the ServiceMonitor matches the port name in the Service spec |
| Alertmanager not sending alerts | Wrong webhook URL or topic | Test manually: `curl -d "test alert" https://ntfy.sh/<YOUR_TOPIC>` |
| kube-prometheus-stack ArgoCD sync fails with `Too many objects`, `metadata.annotations: Too long`, or field manager conflict | CRDs are large and require server-side apply | Add `ServerSideApply=true` to the Application's `syncOptions` (covered in Section 13) |

### Persistent storage

| Symptom | Cause | Fix |
|---|---|---|
| PVC stuck in `Pending` | Storage class not found | `kubectl get storageclass` - verify `local-path` exists |
| Pod fails with `permission denied` writing to mounted volume | Container user does not own the mount path | Add `securityContext.fsGroup: <GID>` to the pod spec to set ownership on mount |
| Pod rescheduled to a different node cannot access its data | `local-path` PVCs are tied to the node where they were first created | This is a limitation of local-path storage. Either pin the pod to the same node with `nodeSelector`, or switch to a distributed storage solution like Longhorn |