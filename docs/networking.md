# Networking

## Interface
| Field | Value |
| --- | --- |
| Interface | `eno1` |
| MAC address | `38:f3:ab:1c:58:cc` |
| IPv4 address | `192.168.1.253/24` |
| Network | `192.168.1.0/24` |
| Default gateway | `192.168.1.1` |

## Persistent Configuration

The server uses Netplan for persistent network configuration.

Configuration file:

`/etc/netplan/00-installer-config.yaml`

```yaml
network:
  ethernets:
    eno1:
      addresses:
        - 192.168.1.253/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1]
      dhcp4: false
      dhcp6: true
      match:
        macaddress: 38:f3:ab:1c:58:cc
      set-name: eno1
  version: 2
  wifis: {}
```

IPv4 is configured statically, while IPv6 remains enabled with DHCPv6.

## Routing

The local subnet `192.168.1.0/24` is directly connected through `eno1`.

All other IPv4 traffic uses the default route through `192.168.1.1`.

## DNS

LAN clients:

    Client
      ↓
    192.168.1.253:53
      ↓
    Pi-hole
      ↓
    127.0.0.1:5335
      ↓
    Unbound

Tailscale clients:

    Client
      ↓
    100.66.59.119:53
      ↓
    Pi-hole
      ↓
    127.0.0.1:5335
      ↓
    Unbound

Clients on the local network can reach Pi-hole through the M70q's LAN address at `192.168.1.253:53`. Tailscale clients can reach the same Pi-hole instance through the M70q's Tailscale address at `100.66.59.119:53`. These are two different IP addresses assigned to the same server, providing different paths to the same DNS service.

When Pi-hole receives a DNS query, it applies its filtering rules. If the query is allowed and requires upstream resolution, Pi-hole forwards it to Unbound at `127.0.0.1:5335`. Unbound then performs recursive DNS resolution and returns the result to Pi-hole, which returns the answer to the client.

The M70q itself normally uses Tailscale's DNS integration. During testing, stopping the `tailscaled` service caused the M70q to fall back to its interface-configured DNS resolver at `192.168.1.1`.

Remote clients depend on Tailscale to reach Pi-hole through `100.66.59.119`. If Tailscale becomes unavailable, they can no longer use that path to the M70q and will depend on the DNS configuration of their underlying network. Likewise, a LAN client that is manually configured to use only `192.168.1.253` would lose DNS resolution if the M70q or Pi-hole becomes unavailable unless another DNS server is configured.


```text
192.168.1.253 ──┐
                ├──► M70q ──► Pi-hole
100.66.59.119 ──┘
```

### Client DNS and Fallback

My Windows PC uses a static IPv4 address because it hosts Docker services that need a consistent LAN address. The underlying DNS server on the Windows network adapter is manually configured to use Cloudflare DNS at `1.1.1.1`.

When Tailscale is connected and **Override DNS** is enabled, Tailscale takes precedence over the DNS server configured on the Windows adapter. DNS queries are sent through Tailscale to the M70q at `100.66.59.119:53`, where Pi-hole handles filtering and forwards allowed queries requiring upstream resolution to Unbound at `127.0.0.1:5335`.

If Tailscale is disconnected, the Tailscale DNS override is removed and Windows returns to its underlying DNS configuration at `1.1.1.1`. This provides an independent DNS path when Tailscale is not being used, although Pi-hole filtering and Unbound resolution are bypassed.

This behavior was verified using `nslookup`:

- **Tailscale connected:** Windows used `magicdns.localhost-tailscale-daemon` (`fd7a:115c:a1e0::53`) as its local DNS resolver.
- **Tailscale disconnected:** Windows used Cloudflare DNS at `1.1.1.1`.

### IPv6

IPv6 configuration is advertised on the local network. The Windows PC receives a global IPv6 address, an IPv6 default gateway, and IPv6 DNS servers.

Testing showed that the PC could successfully reach its local IPv6 gateway at `fe80::1`, but could not reach public IPv6 destinations or the advertised IPv6 DNS servers.

The following connectivity was observed:

```text
PC → IPv6 gateway (fe80::1)       ✓
PC → Public IPv6 Internet         ✕
PC → Advertised IPv6 DNS servers  ✕
