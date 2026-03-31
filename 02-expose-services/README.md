# Expose Services

This section decides how traffic gets from the outside world (or your local network) into your cluster. The three options cover almost every home or self-hosted setup, pick the one that matches your situation and follow only that file.

> [!IMPORTANT]
> Read all three options before deciding. The right choice depends on your network, not on what sounds most impressive.

---

## Which option should I use?

<p align="center">
    <img src="..\references\diagrams\expose-services-flow.svg" alt="Expose services decision flow" />
</p>

> [!NOTE]
> The static IPs you set on your nodes in the cluster setup section were internal LAN addresses (e.g. `192.168.1.50`), required so K3S doesn't break when a node's local address changes. The IP being discussed here is different: it's the *public* IP your ISP assigns to your router. Most home internet connections get a dynamic public IP that changes periodically, which is why Option A exists.

| Option | Best for | Requires |
|---|---|---|
| [A - Cloudflare Tunnel](option-a-cloudflare-tunnel.md) | Home networks, ISP-assigned dynamic public IP | A domain delegated to Cloudflare, free Zero Trust account |
| [B - Direct public IP](option-b-public-ip.md) | VPS, cloud server, or static public IP from your ISP | A domain, static public IP or router port forwarding |
| [C - LAN-only](option-c-lan-only.md) | Local access only, fastest to get running | Nothing - no domain, no accounts, no open ports |

> [!TIP]
> If you don't have a domain yet or just want to get something running, start with **Option C**. Your services will be reachable from any device on your home network immediately. You can migrate to Option A later.
