# M70q Homelab Context

Current project state for use with a local LLM. Update this file as the homelab changes.

> **Security note:** This file is intended for a public Git repository. Do not add passwords, authentication keys, API tokens, private keys, recovery codes, `.env` secrets, account identifiers, public IP addresses, or sensitive log output. Configuration containing secrets must be sanitized before being added here.

## Current Status

- Ubuntu Server / Networking ✓
- SSH ✓
- Git / GitHub ✓
- Pi-hole (native) ✓
- Unbound (native) ✓
- DNSSEC ✓
- Tailscale (native) ✓
- Remote Pi-hole DNS ✓
- Tailscale Exit Node ✓
- UFW ✓
- Client Naming ✓
- Reboot / Persistence Tests ✓
- Documentation ✓
- README ✓
- **Docker ← CURRENT NEXT PHASE**
- Uptime Kuma — planned
- Additional Applications — planned
- Palworld / Other Workloads — planned

Docker has not yet been started in this rebuild. The next objective is to determine whether Docker is already installed on the M70q. Do not immediately provide the command; let me determine how to check first.

## Server

| Field | Value |
|---|---|
| Hardware | Lenovo M70q |
| Hostname | `m70q-srvr` |
| OS | Ubuntu Server 26.04.1 LTS (Resolute Raccoon) |
| LAN interface | `eno1` |
| LAN IPv4 | `192.168.1.253/24` |
| LAN subnet | `192.168.1.0/24` |
| Default gateway | `192.168.1.1` |
| Wi-Fi interface | `wlp1s0` (unused/down) |
| Tailscale IPv4 | `<M70Q_TAILSCALE_IP>` |

The M70q is primarily administered headlessly through SSH.

## Architecture

The intentional deployment model is:

```text
Ubuntu host
├── Native core infrastructure
│   ├── Pi-hole
│   ├── Unbound
│   ├── Tailscale
│   └── UFW
└── Docker
    └── Application workloads
```

Native deployment of core networking services was chosen to provide hands-on experience with systemd, Linux services, `/etc` configuration, ports, logs, permissions, and networking.

Docker is reserved primarily for application workloads.

## Networking

The M70q uses a static IPv4 configuration through Netplan.

Important concepts already covered:

- Same-subnet traffic uses ARP and Layer 2 forwarding rather than the Layer 3 default gateway.
- For off-subnet traffic, the IP packet retains the remote destination IP while the first-hop Ethernet frame uses the gateway's MAC address.
- Default route behavior.
- DNS versus routing.
- Linux interfaces and routing tables.

IPv6 exists on the network, but usable native IPv6 Internet connectivity was not established. IPv6 troubleshooting is intentionally deprioritized unless specifically requested.

## Pi-hole

Pi-hole is installed natively on Ubuntu.

Client-facing DNS:

```text
LAN:       192.168.1.253:53
Tailscale: <M70Q_TAILSCALE_IP>:53
```

Pi-hole forwards allowed upstream queries to:

```text
127.0.0.1#5335
```

This is the local Unbound resolver.

Pi-hole's main daemon is:

```text
pihole-FTL.service
```

Pi-hole v6 configuration is primarily stored in:

```text
/etc/pihole/pihole.toml
```

Verified behavior:

- Normal DNS resolution works.
- Pi-hole blocking works.
- Tested blocked queries returned `0.0.0.0`.
- Remote DNS through Tailscale works.

### Previous Tailscale DNS Issue

Pi-hole originally used the `LOCAL` listening mode.

Tailscale DNS queries reached the M70q but received no response. Packet capture proved that the requests successfully traversed Tailscale and arrived at the server, narrowing the problem to Pi-hole's listening policy rather than Tailscale transport.

After UFW rules were configured to restrict DNS access to trusted LAN and Tailscale paths, Pi-hole's listening mode was changed to `ALL`.

This allowed Pi-hole to answer Tailscale DNS queries while UFW continued to restrict access to port 53.

## Unbound

Unbound is installed natively and acts as Pi-hole's recursive upstream resolver.

It listens only on:

```text
127.0.0.1:5335
```

Normal resolution path:

```text
Client
  ↓
Pi-hole :53
  ↓
Unbound 127.0.0.1:5335
  ↓
Root DNS servers
  ↓
TLD DNS servers
  ↓
Authoritative DNS servers
```

Unbound performs recursive resolution rather than normally forwarding queries to a public resolver such as Cloudflare or Google.

DNSSEC validation was verified:

```text
dnssec.works        → successful response with AD flag
fail01.dnssec.works → SERVFAIL
```

### Previous Port Conflict

After installation, Unbound initially attempted to bind to port 53, which was already occupied by Pi-hole.

Service logs identified the bind failure.

Unbound was then configured to use:

```text
interface: 127.0.0.1
port: 5335
```

Afterward, Unbound started successfully and integrated with Pi-hole.

### IPv6

Unbound currently has IPv6 transport disabled:

```text
do-ip6: no
```

This was chosen because usable native IPv6 Internet connectivity is not currently available.

Unbound can still return AAAA records; this setting concerns the transport Unbound itself uses for upstream DNS communication.

### Boot Observation

One Pi-hole log message was observed shortly after a reboot indicating a temporary failure connecting to Unbound at `127.0.0.1:5335`.

After boot completed, Unbound and Pi-hole worked normally.

A startup/readiness timing issue is a hypothesis, but no configuration change was made because there was not enough evidence of a persistent failure.

## DNS Architecture

### LAN

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

### Tailscale

```text
Remote client
    ↓
Tailscale
    ↓
<M70Q_TAILSCALE_IP>:53
    ↓
Pi-hole
    ↓
127.0.0.1:5335
    ↓
Unbound
    ↓
DNS hierarchy
```

### Allowed Query

```text
Client → Pi-hole :53 → Unbound :5335 → DNS hierarchy
                                      ↓
Client ← Pi-hole ← DNS answer ← Unbound
```

### Blocked Query

```text
Client → Pi-hole :53
             ↓
        Gravity match
             ↓
Client ← 0.0.0.0
```

No Unbound query is needed for a blocked domain.

## Tailscale

Tailscale is installed natively.

M70q Tailscale IPv4:

```text
<M70Q_TAILSCALE_IP>
```

Tailscale provides:

- Secure remote homelab access.
- Remote Pi-hole/Unbound DNS.
- An optional exit node.

The M70q is configured as the tailnet's global DNS destination at `<M70Q_TAILSCALE_IP>`, with Override DNS enabled.

Remote DNS has been tested successfully from Windows and iPhone clients.

### Tailscale Exit Node

The M70q is configured as an optional Tailscale exit node.

Persistent Linux forwarding configuration is stored in:

```text
/etc/sysctl.d/99-tailscale.conf
```

with:

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

The exit node was tested from an iPhone over cellular. Internet access continued to work and the observed public IPv4 changed to the home's public IPv4.

### Without Exit Node

```text
Tailnet/homelab traffic → Tailscale

Configured DNS → Tailscale → M70q → Pi-hole → Unbound

Normal Internet → Client's normal Internet connection
```

Do not simplify this to "only DNS uses Tailscale." Other tailnet/homelab traffic also uses Tailscale.

### With M70q Exit Node

```text
Remote client
    ↓
Tailscale
    ↓
M70q
    ↓
Home router
    ↓
Internet
```

DNS remains separately configured through Tailscale DNS and continues to use Pi-hole/Unbound.

## Windows Client

Main Windows PC:

| Field | Value |
|---|---|
| LAN IPv4 | `192.168.1.252` |
| Tailscale IPv4 | `<WINDOWS_TAILSCALE_IP>` |
| Pi-hole client name | `<WINDOWS_HOSTNAME>` |
| Underlying IPv4 DNS | `1.1.1.1` |

The LAN address is intentionally static because this PC hosts application/Docker services and a router-side DHCP reservation is not currently available.

Verified DNS behavior:

```text
Tailscale ON + Override DNS
    ↓
Tailscale DNS integration
    ↓
M70q
    ↓
Pi-hole
    ↓
Unbound
```

With Tailscale disconnected, Windows returned to `1.1.1.1`.

Do not claim that Windows automatically falls back to `1.1.1.1` if the M70q fails while Tailscale remains connected. That scenario was not verified.

IPv6 is currently disabled on the Windows Ethernet adapter because advertised IPv6 DNS servers were unreachable and caused DNS timeouts when Tailscale was disconnected.

## iPhone Client

| Field | Value |
|---|---|
| Pi-hole client name | `<IPHONE_HOSTNAME>` |
| Tailscale IPv4 | `<IPHONE_TAILSCALE_IP>` |

Remote Pi-hole DNS and M70q exit-node functionality were tested successfully from this device over cellular.

## UFW Firewall

UFW is enabled.

Default policy:

```text
Incoming:  DENY
Outgoing:  ALLOW
Forwarded: DENY
```

Explicitly permitted service traffic includes:

- TCP/UDP port 53 from `192.168.1.0/24` through `eno1`.
- TCP/UDP port 53 through `tailscale0`.
- TCP ports 80 and 443 from the trusted LAN.
- TCP ports 80 and 443 through `tailscale0`.
- OpenSSH is currently allowed more broadly than the service-specific LAN/Tailscale rules.

Restricting SSH further is a future hardening opportunity.

UFW's forwarded default is deny, but the tested Tailscale exit node still functions because Tailscale manages its own networking/netfilter behavior together with Linux IP forwarding.

Do not add unnecessary UFW routed rules without troubleshooting evidence that they are required.

## Git and GitHub

Repository location:

```text
~/m70q-homelab
```

Repository name:

```text
m70q-homelab
```

GitHub SSH authentication is configured using Ed25519.

Normal workflow:

```text
edit
 ↓
git diff
 ↓
git add
 ↓
git diff --staged
 ↓
git commit
 ↓
git push
```

The repository is public. Sensitive credentials and unnecessary identifying/network data should not be committed.

Real configurations containing secrets should remain ignored locally, with sanitized example files committed when useful.

## Documentation

Current repository documentation:

```text
m70q-homelab/
├── README.md
└── docs/
    ├── networking.md
    ├── pihole.md
    ├── unbound.md
    ├── tailscale.md
    └── firewall.md
```

These documents and the updated README have been completed and pushed.

The README serves as the concise front page. Detailed explanations belong in `docs/`.

## Persistence Testing

The M70q was rebooted and the following were verified to survive reboot and function afterward:

- Static LAN networking.
- SSH.
- Unbound.
- Pi-hole FTL.
- Tailscale.
- UFW.
- DNS resolution.

## Next Phase: Docker

The native infrastructure phase is complete.

The next phase is Docker for application workloads.

Planned progression:

```text
Docker
  ↓
Uptime Kuma
  ↓
Additional self-hosted applications
  ↓
Palworld / other workloads
```

Docker should be treated as another learning exercise rather than simply copying an installation guide.

## Immediate Next Objective

Determine whether Docker is currently installed on the M70q.

Give only this objective first. Let me decide how to check before providing a command.
