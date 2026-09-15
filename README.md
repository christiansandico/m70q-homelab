# m70q-homelab

A self-hosted homelab running on Ubuntu Server for learning Linux administration, networking, DNS, security, containerization, monitoring, and infrastructure management.

Core network services run natively on the host, while Docker is used for application workloads.

## Network Configuration

| Field | Value |
| --- | --- |
| Hostname | `m70q-srvr` |
| Operating System | Ubuntu 26.04.1 LTS (Resolute Raccoon) |
| Server LAN IP | `192.168.1.253` |
| Subnet | `192.168.1.0/24` |
| Default Gateway | `192.168.1.1` |
| Tailscale IP | `100.66.59.119` |
| LAN Interface | `eno1` |

## Services

| Service | Deployment | Address | Purpose |
| --- | --- | --- | --- |
| Pi-hole | Native | `192.168.1.253:53` / Tailscale | Network-wide DNS filtering |
| Unbound | Native | `127.0.0.1:5335` | Recursive DNS resolver with DNSSEC validation |
| Tailscale | Native | `100.66.59.119` | Secure remote access, remote DNS, and optional exit node |
| UFW | Native | Host firewall | Restricts access to trusted network paths |
| Docker | Native | Host runtime | Runs containerized application workloads |
| Uptime Kuma | Docker | `192.168.1.253:3001` / Tailscale | Service, DNS, and connectivity monitoring |

## DNS Architecture

Pi-hole is the DNS service exposed to clients. It applies filtering rules and answers blocked queries locally.

Allowed queries requiring upstream resolution are forwarded to Unbound at `127.0.0.1:5335`.

Unbound performs recursive DNS resolution using the DNS hierarchy rather than forwarding normal queries to a public resolver such as Cloudflare or Google. DNSSEC validation is enabled.

### LAN DNS

```text
LAN client
    ↓
192.168.1.253:53
    ↓
Pi-hole
    ↓
127.0.0.1:5335
    ↓
Unbound
    ↓
DNS hierarchy
```

### Remote DNS

Tailscale clients can use the same DNS stack while outside the local network.

```text
Remote client
    ↓
Tailscale
    ↓
100.66.59.119:53
    ↓
Pi-hole
    ↓
127.0.0.1:5335
    ↓
Unbound
    ↓
DNS hierarchy
```

### Query Flow

**Allowed:**

```text
Client → Pi-hole :53 → Unbound :5335 → DNS hierarchy
                                      ↓
Client ← Pi-hole ← DNS answer ← Unbound
```

**Blocked:**

```text
Client → Pi-hole :53
             ↓
        Gravity match
             ↓
Client ← 0.0.0.0

No Unbound query needed
```

Blocked domains are answered locally by Pi-hole and therefore do not need to be forwarded to Unbound.

## Remote Access

Tailscale provides encrypted remote connectivity to the homelab without requiring services to be directly exposed to the public Internet.

The M70q also operates as an optional Tailscale exit node. When selected by a remote device, Internet traffic can be routed through the M70q and the home Internet connection.

```text
Remote device
    ↓
Tailscale
    ↓
M70q exit node
    ↓
Home router
    ↓
Internet
```

Using the exit node is optional. Without it, normal Internet traffic continues through the client's existing Internet connection while Tailscale is used for tailnet resources and configured DNS.

## Containerized Applications

Docker is used for application workloads while core networking services remain installed natively on Ubuntu Server.

Uptime Kuma is the first containerized application deployed in the homelab. Docker Compose defines the deployment, while a Docker named volume provides persistent application data.

```text
Ubuntu Server
│
├── Native services
│   ├── Pi-hole
│   ├── Unbound
│   ├── Tailscale
│   └── UFW
│
└── Docker
    └── Uptime Kuma
```

The Uptime Kuma container runs on a Docker bridge network and communicates with selected native services through firewall-controlled paths.

## Monitoring

Uptime Kuma provides basic availability monitoring for the homelab.

The current monitoring setup checks:

- M70q reachability
- Pi-hole web interface
- Pi-hole DNS resolution
- Internet connectivity
- External DNS resolution

Discord is configured for monitor notifications.

Because Uptime Kuma runs locally on the M70q, it cannot independently report a complete M70q or home Internet outage if the server cannot reach Discord. Independent external monitoring would be required for that use case.

## Firewall

UFW provides host-level firewall protection using a default-deny policy for incoming traffic.

Access to services such as DNS and web interfaces is restricted to trusted LAN and Tailscale network paths.

Docker introduces an additional networking boundary. Uptime Kuma's Docker network is permitted to reach only the native host services required for monitoring, including Pi-hole's HTTP and DNS ports.

Tailscale's networking and Linux IP forwarding are used separately for exit-node traffic.

## Documentation

Detailed documentation for the homelab is available in the `docs/` directory:

- [Networking](docs/networking.md)
- [Pi-hole](docs/pihole.md)
- [Unbound](docs/unbound.md)
- [Tailscale](docs/tailscale.md)
- [Firewall](docs/firewall.md)
- [Uptime Kuma](docs/uptime-kuma.md)

Docker application configurations are stored separately under the `docker/` directory.

## Project Status

### Completed

- Ubuntu Server installation and static network configuration
- SSH administration
- Git and GitHub repository setup
- Native Pi-hole DNS filtering
- Native Unbound recursive DNS
- DNSSEC validation
- Native Tailscale remote access
- Pi-hole DNS over Tailscale
- Tailscale exit node
- UFW firewall configuration
- Docker Engine and Docker Compose
- Docker persistent storage and bridge networking
- Uptime Kuma monitoring
- Discord monitor notifications
- Docker-to-host firewall troubleshooting
- Reboot and service persistence testing

### Planned

- Additional self-hosted applications
- Expanded infrastructure monitoring and metrics
- Networking labs
