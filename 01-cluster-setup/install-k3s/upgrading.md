### Upgrading K3S

Re-run the install script with `INSTALL_K3S_VERSION` set to your target version. Pass the same flags used during the original install. In a multi-node cluster, upgrade the server first, verify it shows `Ready`, then upgrade agents one at a time.

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=<TARGET_VERSION> sh -s - --tls-san <NODE_IP>
```

Version strings look like `v1.32.3+k3s1`; find them on the [K3S releases page](https://github.com/k3s-io/k3s/releases). For automated rolling upgrades across multiple nodes, see the [K3S system-upgrade-controller](https://docs.k3s.io/upgrades/automated).

> **Reference:** [K3S Manual Upgrades](https://docs.k3s.io/upgrades/manual)
