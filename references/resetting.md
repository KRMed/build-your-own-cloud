## Resetting and Starting Over

**Skip unless you need it** - this section covers how to completely uninstall K3S if something went wrong and you want a clean slate.

K3S ships with uninstall scripts that remove all K3S components, configuration, and data. This is non-reversible - all cluster state, namespaces, and workloads will be gone.

> **Reference:** [K3S uninstall documentation](https://docs.k3s.io/installation/uninstall)

**On the server node:**

```bash
/usr/local/bin/k3s-uninstall.sh
```

**On each agent node:**

```bash
/usr/local/bin/k3s-agent-uninstall.sh
```

These scripts stop the K3S service, remove the binary, and clean up all state including namespaces, pods, and network interfaces. After running them, you can re-run the install steps from Step 3 with a completely clean starting point.

**Note:** Persistent volume data stored at `/var/lib/rancher/k3s/storage/` is removed by the uninstall script. If you want to keep that data, copy it somewhere else before uninstalling.
