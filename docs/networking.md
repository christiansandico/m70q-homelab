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

