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
| Pi-hole | Native | `192.168.1.253:53` / Tailscale | Network-wide DNS filtering and local DNS |
| Unbound | Native | `127.0.0.1:5335` | Recursive DNS resolver with DNSSEC validation |
| Tailscale | Native | `100.66.59.119` | Secure remote access, remote DNS, and optional exit node |
| UFW | Native | Host firewall | Restricts access to trusted network paths |
| Docker | Native | Host runtime | Runs containerized application workloads |
| Uptime Kuma | Docker | `192.168.1.253:3001` | Service, DNS, connectivity, and Docker container monitoring |
| Stirling PDF | Docker | `192.168.1.253:8080` / `100.66.59.119:8080` | Self-hosted PDF management and processing |
| Palworld | Docker | `192.168.1.253:8211` / `100.66.59.119:8211` | Private dedicated game server |

## DNS Architecture

Pi-hole is the DNS service exposed to clients. It applies filtering rules, provides local DNS resolution, and answers blocked queries locally.

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

## Local DNS

Pi-hole provides private DNS records for services hosted on the M70q.

Separate DNS names are used for LAN and Tailscale access. Local records resolve to the M70q's static LAN address (`192.168.1.253`), while Tailscale records resolve to its Tailscale address (`100.66.59.119`).

### Local

| Address | Service |
| --- | --- |
| `http://pihole.home.arpa/admin/login` | Pi-hole |
| `http://stirling.home.arpa:8080` | Stirling PDF |
| `http://uptime.home.arpa:3001` | Uptime Kuma |
| `http://status.home.arpa:3001/status/homelab` | Homelab Status Page |
| `m70q-srvr.home.arpa` | M70q Server |

### Tailscale

| Address | Service |
| --- | --- |
| `http://pihole-ts.home.arpa/admin/login` | Pi-hole |
| `http://stirling-ts.home.arpa:8080` | Stirling PDF |
| `http://uptime-ts.home.arpa:3001` | Uptime Kuma |
| `http://status-ts.home.arpa:3001/status/homelab` | Homelab Status Page |
| `m70q-srvr-ts.home.arpa` | M70q Server |

Local DNS records point to:

```text
192.168.1.253
```

Tailscale DNS records point to:

```text
100.66.59.119
```

This provides two explicit network paths to the same services. For example:

```text
Local:
stirling.home.arpa
        ↓
192.168.1.253
        ↓
Stirling PDF :8080

Tailscale:
stirling-ts.home.arpa
        ↓
100.66.59.119
        ↓
Stirling PDF :8080
```

DNS itself maps hostnames to IP addresses. Port numbers are not part of DNS and still identify the application service being accessed.

### Why `.home.arpa`?

The `home.arpa` domain is reserved specifically for naming devices and services on residential home networks. It provides a private namespace for the homelab without conflicting with public Internet domains.

Using `.home.arpa` also avoids `.local`, which is reserved for Multicast DNS (mDNS) and is commonly used by technologies such as Bonjour and Avahi.

These records are provided by the homelab's Pi-hole DNS server and are not intended to be publicly resolvable on the Internet.

## Remote Access

Tailscale provides encrypted remote connectivity to the homelab without requiring services to be directly exposed to the public Internet.

Services available through the M70q's Tailscale address can use their corresponding `-ts.home.arpa` DNS names when the client is connected to Tailscale and using Pi-hole for DNS.

For example:

```text
http://stirling-ts.home.arpa:8080
```

resolves to:

```text
100.66.59.119
```

and reaches Stirling PDF through the Tailscale network rather than through the M70q's LAN address.

Palworld also uses Tailscale for private remote access. Authorized remote players connect directly to:

```text
100.66.59.119:8211
```

No router port forwarding is configured for Palworld.

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

Tailscale key expiry is disabled for the M70q because it is an always-on infrastructure server intended to remain remotely accessible without periodic reauthentication.

## Containerized Applications

Docker is used for application workloads while core networking services remain installed natively on Ubuntu Server.

Current containerized applications:

- Uptime Kuma
- Stirling PDF
- Palworld

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
    ├── Uptime Kuma
    │   └── Docker socket proxy
    ├── Stirling PDF
    └── Palworld
```

Docker Compose is used to define application deployments, with persistent application data stored outside the disposable container layer.

### Palworld Dedicated Server

Palworld runs as a private Docker-based dedicated game server.

Players on the LAN connect through:

```text
192.168.1.253:8211
```

Authorized remote players connect through Tailscale:

```text
100.66.59.119:8211
```

The server is configured for two players and includes:

- Password-protected access
- Persistent world and player data
- Cross-platform support
- No item loss on player death
- Player logging
- Automatic idle pausing after five minutes
- Automatic wake when a player reconnects
- Save-before-pause behavior
- Daily automatic backups
- 14-day backup retention
- Uptime Kuma container monitoring
- Discord monitor notifications

The Palworld REST API is enabled internally for server management functionality but is not published through Docker.

Server and administrator passwords are stored in a local `.env` file rather than in the public Compose configuration.

The `.env` file and persistent Palworld runtime data are excluded from Git.

### Docker Port Exposure

Docker-published ports require separate consideration from native services protected by UFW.

During the Stirling PDF deployment, the initial Compose configuration published port 8080 using:

```yaml
ports:
  - "8080:8080"
```

This resulted in Docker binding the service to the host's wildcard addresses:

```text
0.0.0.0:8080
[::]:8080
```

The deployment was restricted to the interfaces that require access:

```yaml
ports:
  - "192.168.1.253:8080:8080"
  - "100.66.59.119:8080:8080"
```

This provides Stirling PDF through the LAN and Tailscale interfaces without publishing the service on every host address.

No router port forwarding is configured for Stirling PDF. Remote access is provided through Tailscale instead of exposing TCP port 8080 directly to the public Internet.

Palworld publishes the game-related UDP ports defined by its Docker Compose configuration:

```text
8211/udp
27015/udp
```

The Palworld REST API uses TCP port `8212` internally but is not published on the host.

No router port forwarding is configured for Palworld. Remote game connections use Tailscale instead of exposing the server directly to the public Internet.

## Monitoring

Uptime Kuma provides basic availability monitoring for the homelab.

The current monitoring setup checks:

- M70q reachability
- Pi-hole web interface
- Pi-hole DNS resolution
- Stirling PDF web interface
- Palworld Docker container
- Internet connectivity
- External DNS resolution

Docker container monitoring is provided through a Docker socket proxy.

The monitoring path is:

```text
Uptime Kuma
     ↓
Docker socket proxy
     ↓
Docker Engine
     ↓
Monitored container
```

Uptime Kuma connects to the proxy internally at:

```text
http://docker-socket-proxy:2375
```

The proxy is available only through Uptime Kuma's Docker network and is not published on a host port.

This allows Uptime Kuma to monitor Docker containers without mounting the host Docker socket directly into the Uptime Kuma container.

Palworld is monitored using a Docker Container monitor targeting:

```text
palworld-server
```

The monitor intentionally tracks the container rather than requiring the game process to continuously respond while idle.

This was verified by allowing Palworld to enter its automatic paused state:

```text
Palworld game process → paused
palworld-server       → running
Uptime Kuma           → Up
```

The monitor therefore remains Up during normal idle pauses while still allowing the Docker container itself to be monitored.

The local status page is available at:

```text
http://status.home.arpa:3001/status/homelab
```

The corresponding Tailscale address is:

```text
http://status-ts.home.arpa:3001/status/homelab
```

Palworld is included on the homelab status page.

Discord is configured for monitor notifications, including the Palworld container monitor.

Because Uptime Kuma runs locally on the M70q, it cannot independently report a complete M70q or home Internet outage if the server cannot reach Discord. Independent external monitoring would be required for that use case.

## Firewall

UFW provides host-level firewall protection using a default-deny policy for incoming traffic.

Access to native services such as DNS and web interfaces is restricted to trusted LAN and Tailscale network paths.

Docker introduces an additional networking boundary. Docker-published ports use Docker-managed forwarding/firewall rules and should not be assumed to follow UFW's normal incoming filtering behavior.

Uptime Kuma's Docker network is permitted to reach only the native host services required for monitoring, including Pi-hole's HTTP and DNS ports.

Docker container monitoring uses an internal Docker socket proxy on the Uptime Kuma Docker network. The proxy's Docker API port is not published on the M70q host.

Where appropriate, Docker services can be explicitly bound to selected host addresses rather than published on wildcard addresses.

Tailscale's networking and Linux IP forwarding are used separately for exit-node traffic.

## Documentation

Detailed documentation for the homelab is available in the `docs/` directory:

- [Networking](docs/networking.md)
- [Pi-hole](docs/pihole.md)
- [Unbound](docs/unbound.md)
- [Tailscale](docs/tailscale.md)
- [Firewall](docs/firewall.md)
- [Uptime Kuma](docs/uptime-kuma.md)
- [Stirling PDF](docs/stirling-pdf.md)
- [Palworld](docs/palworld.md)

Docker application configurations are stored separately under the `docker/` directory. Persistent application data and other sensitive runtime state are excluded from the public repository.

## Project Status

### Completed

- Ubuntu Server installation and static network configuration
- SSH administration
- Git and GitHub repository setup
- Native Pi-hole DNS filtering
- Native Unbound recursive DNS
- DNSSEC validation
- Local DNS using the reserved `.home.arpa` namespace
- Separate LAN and Tailscale DNS names for homelab services
- Native Tailscale remote access
- Pi-hole DNS over Tailscale
- Tailscale exit node
- Tailscale persistent server authentication
- UFW firewall configuration
- Docker Engine and Docker Compose
- Docker persistent storage and bridge networking
- Uptime Kuma monitoring
- Docker container monitoring through a restricted socket proxy
- Homelab status page
- Stirling PDF deployment
- Stirling PDF persistent application storage
- LAN and Tailscale access to Stirling PDF
- Explicit Docker host-interface port bindings for Stirling PDF
- Palworld dedicated server deployment
- Palworld persistent world and player data
- LAN and Tailscale access to Palworld
- Palworld automatic idle pause and wake
- Palworld save-before-pause behavior
- Palworld daily backups with 14-day retention
- Palworld Docker container monitoring
- Palworld status page integration
- Environment-based Palworld secret management
- Discord monitor notifications
- Docker-to-host firewall troubleshooting
- Reboot and service persistence testing

### Planned

- Additional self-hosted applications
- Reverse proxy for hostname-based service access
- Internal HTTPS/TLS
- Expanded infrastructure monitoring and metrics
- Networking labs
