## Prepare Your Nodes

**Required** - do this for every node in your cluster before installing K3S.

> **Mixed hardware clusters:** The steps in this section are the same for every node, but the specifics vary by OS and hardware. Run each check on each node individually and apply the fix that matches that node, the cgroups path, firewall requirements, and static IP method all differ between Raspberry Pi nodes and x86 nodes. Do not apply Raspberry Pi-specific steps to x86 nodes or vice versa.

### Set a static IP or DHCP reservation

K3S nodes communicate by IP. If a node's IP changes after install, the cluster breaks because all the TLS certificates and kubeconfig entries reference the old address. Either:

- Configure a static IP in your OS network settings, or
- Create a DHCP reservation in your router so the node always gets the same IP

DHCP reservation is usually easier. Most home routers support it under a section called "Address Reservation" or "Static Leases".

> **No control over your network?** If you are on a shared or public network (university housing, public WiFi, a workplace network) where you cannot configure DHCP reservations or assign static IPs, a cheap unmanaged network switch gives you a private subnet that your nodes can communicate on. Connect all your Pi nodes to the switch, then connect the switch to the shared network uplink. The nodes will see each other at stable IPs on the switch's subnet, isolated from the upstream network's DHCP churn.

> **VPS and cloud VM users:** Skip this section. Cloud providers assign your instance a fixed public IP at the infrastructure level, it will not change unless you explicitly release it or destroy the instance. Do not attempt to configure a static IP through the OS on a cloud VM; this can break cloud provider networking.

> **OS note:** How you configure a static IP depends on your distro. Ubuntu uses Netplan (`/etc/netplan/`, then `sudo netplan apply`). Raspberry Pi OS uses dhcpcd (`/etc/dhcpcd.conf`). Fedora and RHEL-based systems use NetworkManager (`nmcli` or `nmtui`). Many other distros also use NetworkManager or systemd-networkd. If you are unsure, `ip route` and `ip addr` will show your current network config. DHCP reservation on the router is the easiest option regardless of OS.

### Enable cgroups memory (required on some hardware)

K3S uses Linux control groups (cgroups) to enforce container resource limits like CPU and memory caps. On most modern distros this is already enabled. On Raspberry Pi OS and some minimal distros it is not.

Check whether it is enabled:

```bash
cat /proc/cgroups | grep memory
```

> **Ubuntu 22.04 and 24.04 users:** These distros default to cgroups v2 with memory accounting already enabled. You will almost certainly see `1` here and can skip the rest of this section.

If the last column is `1`, you are fine and can skip the rest of this section. If it is `0`, cgroups memory is disabled and K3S will either fail to start or fail silently when resource limits are applied.

To enable it, edit the kernel boot parameters. The file location depends on your OS **and your hardware**:

- **Any OS on Raspberry Pi hardware (Pi OS, Ubuntu, Debian):** `/boot/firmware/cmdline.txt`. Raspberry Pi does not use GRUB regardless of which OS is installed. The Pi firmware bootloader always reads from this file
- **Ubuntu / Debian on x86 hardware (GRUB):** `/etc/default/grub` - add to `GRUB_CMDLINE_LINUX`, then run `sudo update-grub`

**For Raspberry Pi (any OS):** append to the end of the existing single line in `/boot/firmware/cmdline.txt` (do not add a new line; this file must remain a single line):

```
cgroup_memory=1 cgroup_enable=memory
```

The K3S documentation specifically requires these two parameters on Raspberry Pi OS. `cgroup_enable=cpuset` is often seen in older guides and is harmless to include, but is not required by K3S.

> **Reference:** [K3S Raspberry Pi requirements](https://docs.k3s.io/installation/requirements?os=pi)

**For Ubuntu / Debian on x86 (GRUB):** add to `GRUB_CMDLINE_LINUX` in `/etc/default/grub`:

```
cgroup_memory=1 cgroup_enable=memory
```

Then run `sudo update-grub` to regenerate the GRUB boot configuration and apply the new parameters on next boot. This is required because GRUB reads from a generated config file, not directly from `/etc/default/grub`.

> **Note:** Ubuntu 23.10 and newer, and some other modern distros (Fedora, etc.), use **systemd-boot** instead of GRUB on x86. If `/etc/default/grub` does not exist on your system, check whether you are using systemd-boot (`bootctl status`). For systemd-boot, kernel parameters are set in the boot entry file under `/boot/loader/entries/`. However, on these distros cgroups memory is almost always already enabled; run the check first and only proceed here if the output shows `0`.

Reboot after making this change, then re-run the check above to confirm the column now shows `1`.

> **Reference:** [K3S known issues](https://docs.k3s.io/known-issues)

### Ensure SSH access

Confirm you can SSH into each node from your local machine. If you configured key-based auth during OS setup, you are ready. If not:

```bash
ssh-copy-id <YOUR_USERNAME>@<NODE_IP>
```

Key-based auth is worth setting up properly, you will be SSHing into these nodes frequently, and password prompts get old quickly.

### Open required firewall ports

**Required if your distro has an active firewall. Can be skipped if no firewall is running.**

In a mixed cluster, run this check on each node individually, you may need to open ports on your x86 Ubuntu nodes but not on your Raspberry Pi Debian nodes.

If you are unsure what firewall your system uses, first identify it:

```bash
# Check for UFW (Ubuntu and Ubuntu-based distros)
sudo ufw status

# Check for firewalld (Fedora, RHEL, Rocky, Alma, CentOS)
sudo firewall-cmd --state

# Check for active nftables or iptables rules (other distros)
sudo nft list ruleset
```

If none of these are active, your distro has no firewall rules and you can skip the rest of this section. If UFW is active, follow the UFW commands below. If firewalld is active, the K3S docs have a [firewalld-specific section](https://docs.k3s.io/installation/requirements#networking) with equivalent commands.

Ubuntu 22.04 and 24.04 ship with UFW installed and sometimes pre-enabled. If UFW is active and you do not open the ports K3S needs, the cluster will appear to install correctly but inter-node networking and remote kubectl access will silently fail in ways that are hard to diagnose.

Check whether UFW is active:

```bash
sudo ufw status
```

If it says `inactive`, you can skip the rest of this section. If it says `active`, open the required ports before installing K3S.

> **VPS and cloud VM users:** Your provider has a separate network-level firewall (called Security Groups on AWS/DigitalOcean/Hetzner, Firewall Rules on GCP, Network Security Groups on Azure) that sits in front of your VM. UFW alone is not enough, you must also open the required ports in your provider's control panel. Both layers must allow the traffic or it will be silently dropped before it reaches your node. Check your provider's documentation for how to configure inbound rules.

> **Reference:** [K3S networking requirements](https://docs.k3s.io/installation/requirements#networking)

**On the server node only:**

```bash
# Allow kubectl access from your local machine and agent nodes
sudo ufw allow 6443/tcp
```

**On every node (server and agents):**

```bash
# Allow flannel VXLAN for pod-to-pod networking across nodes
sudo ufw allow 8472/udp

# Allow kubelet for kubectl logs, exec, and port-forward
sudo ufw allow 10250/tcp
```

**If you are running a multi-server HA setup (multiple control plane nodes), also open:**

```bash
# etcd peer communication
sudo ufw allow 2379:2380/tcp
```

**If you want to reach services via NodePort from other machines on your LAN:**

```bash
# NodePort range
sudo ufw allow 30000:32767/tcp
```

Reload UFW after making changes:

```bash
sudo ufw reload
```

If you are using `firewalld` (common on Fedora and RHEL-based distros) instead of UFW, the K3S docs have a [firewalld-specific section](https://docs.k3s.io/installation/requirements#networking) with equivalent commands.

**If you plan to use Option B (direct public IP or port forwarding):** also open ports 80 and 443, which are required for inbound web traffic and for Let's Encrypt certificate issuance (the HTTP-01 challenge requires port 80 to be reachable from the internet):

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
```

### Disable swap (recommended)

Kubernetes schedulers make resource decisions assuming swap is off. K3S will run with swap enabled on most configurations, but it can cause unpredictable behavior when memory pressure is high, particularly around pod eviction. The [K3S known issues page](https://docs.k3s.io/known-issues) calls out swap as a source of instability and recommends disabling it.

**Skip this if:** you are on a low-RAM machine (1-2GB) where swap provides a meaningful safety net, or if you are comfortable managing swap behavior via Kubernetes `swapBehavior` config.

```bash
sudo swapoff -a
```

To make it permanent across reboots, comment out the swap line in `/etc/fstab`:

```bash
sudo nano /etc/fstab
# Add a # at the start of any line that contains "swap"
```

**Next:** choose your K3S install path in [install-k3s/](install-k3s/)