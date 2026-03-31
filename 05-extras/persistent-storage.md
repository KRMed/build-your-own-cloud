## Persistent Storage

**Required if any of your apps need to write data to disk** (databases, file stores, anything stateful). **Skip if all your apps are stateless** (static sites, APIs with external databases, etc.).

By default, a container's filesystem is ephemeral - anything written to it is lost when the pod restarts. For stateful apps like PostgreSQL, MySQL, or any app that stores data locally, you need a `PersistentVolume`.

K3S ships with the [local-path provisioner](https://github.com/rancher/local-path-provisioner) pre-installed. This means you can request persistent storage in your manifests and it works immediately, without any additional setup. The provisioner creates directories on the node's filesystem and mounts them into your pods.

> **Reference:** [Kubernetes Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)

### How to request storage in a manifest

A `PersistentVolumeClaim` (PVC) is how a pod requests storage. Create one alongside your deployment:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
  namespace: my-app
spec:
  accessModes:
    - ReadWriteOnce   # one pod reads/writes at a time
  storageClassName: local-path   # uses the K3S built-in provisioner
  resources:
    requests:
      storage: 5Gi
```

Then mount it in your pod's container spec:

```yaml
spec:
  containers:
    - name: my-app
      image: my-app:latest
      volumeMounts:
        - name: data
          mountPath: /var/lib/my-app/data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-app-data
```

Verify the PVC was provisioned:

```bash
kubectl get pvc -n my-app
```

The status should show `Bound`. If it shows `Pending`, check that the `local-path` storage class exists:

```bash
kubectl get storageclass
```

### Where data actually lives

The local-path provisioner stores data on the node's filesystem at `/var/lib/rancher/k3s/storage/` by default. This means:

- **Data is tied to a specific node.** If the pod moves to a different node (e.g. after a failure in a multi-node setup), the data will not follow it automatically
- **Data survives pod restarts** on the same node
- **Data does not survive node loss** unless you have separate backups

For a home lab single-node setup this is usually fine. For a multi-node setup with stateful apps you have two options:

**Option 1 - Node pinning (simple, no extra software):** Label the node where the PVC data lives, then pin the stateful pod to that node with a `nodeSelector`. The pod will always schedule onto the node that holds its data:

```bash
# Label the node that has the storage (run once)
kubectl label node <NODE_NAME> storage=stateful-app
```

```yaml
# In your pod/Deployment spec
spec:
  nodeSelector:
    storage: stateful-app
```

This is the low-overhead approach. The tradeoff is that if that node goes down, the pod cannot reschedule until the node comes back, but the data is safe on disk.

**Option 2 - Distributed storage (Longhorn):** [Longhorn](https://longhorn.io/) replicates volume data across multiple nodes, so a pod can reschedule onto any node and still reach its data. More resilient, but adds meaningful resource overhead and operational complexity.

> **Reference:** [K3S local-path provisioner](https://github.com/rancher/local-path-provisioner) | [Longhorn documentation](https://longhorn.io/docs/)

### Backing up persistent data

The local-path provisioner does not handle backups. Back up `/var/lib/rancher/k3s/storage/` to an external drive or remote location. A simple rsync to an external drive mounted at `/mnt/backup`:

```bash
# One-time backup
rsync -av /var/lib/rancher/k3s/storage/ /mnt/backup/k3s-storage/

# Scheduled daily at 2am. Add to crontab with: crontab -e
0 2 * * * rsync -a /var/lib/rancher/k3s/storage/ /mnt/backup/k3s-storage/
```

`rsync -a` copies files recursively while preserving permissions, timestamps, and symlinks. The trailing slash on the source path (`storage/`) is important: it means "copy the contents of this directory", not the directory itself. Without it, you get a nested `storage/storage/` structure in the destination. `-v` (verbose) adds per-file output, useful for one-time runs but noisy in cron; omit it from the cron entry.

For remote backups, replace the destination with an SSH target: `user@remote-host:/path/to/backup/`.

`crontab -e` opens your user's crontab for editing. The five fields before the command are: `minute hour day-of-month month day-of-week`. `0 2 * * *` means "at 02:00, every day". Use [crontab.guru](https://crontab.guru/) to verify or build a cron schedule interactively.

> **References:** [`rsync` manual](https://linux.die.net/man/1/rsync) | [`crontab` manual](https://linux.die.net/man/1/crontab) | [crontab.guru](https://crontab.guru/)

> **Single-node users:** On a single node there is no other node to replicate to, so a hardware failure or corrupted SD card means all persistent data is lost. Treat regular backups of `/var/lib/rancher/k3s/storage/` as mandatory if you are running anything stateful.