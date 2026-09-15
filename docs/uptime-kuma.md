# Uptime Kuma

## Purpose

Uptime Kuma provides service monitoring for the homelab. It runs as a Docker container on the M70q and monitors the availability of local services, DNS resolution, and Internet connectivity.

The initial monitoring setup includes:

- M70q host reachability
- Pi-hole web interface
- Pi-hole DNS resolution
- Internet connectivity
- External DNS resolution

Discord is used for monitor notifications.

## Docker Deployment

Uptime Kuma runs as a Docker workload rather than as a native Ubuntu service.

The Docker Compose configuration is stored at:

`docker/uptime-kuma/compose.yaml`

The container uses the Uptime Kuma 2 image:

`louislam/uptime-kuma:2`

The web interface is published on host port `3001`:

```text
M70q :3001 → Uptime Kuma container :3001
```

This allows the dashboard to be accessed through the M70q.

A custom Docker bridge network named `kuma_network` is defined through Docker Compose. Docker created the Compose network as:

`uptime-kuma_kuma_network`

During deployment, the network was assigned:

```text
Subnet:  172.18.0.0/16
Gateway: 172.18.0.1
```

The Uptime Kuma container was assigned `172.18.0.2` during testing.

Container IP addresses are treated as dynamic and are not relied upon as permanent service addresses.

## Persistent Storage

Uptime Kuma stores its application data in the external Docker volume:

`uptime-kuma-data`

The volume is mounted inside the container at:

`/app/data`

Using persistent storage separates the application data from the lifecycle of the container.

This was verified by removing and recreating the Uptime Kuma container with Docker Compose. The container was recreated while the existing Uptime Kuma configuration remained available.

Persistence was also verified after rebooting the M70q. Docker automatically restarted Uptime Kuma, and the existing monitors and application configuration remained intact.

## Monitoring

### M70q

A Ping monitor checks reachability of:

`192.168.1.253`

This verifies that the M70q is reachable from the Uptime Kuma container.

Because Uptime Kuma itself runs on the M70q, this is not an independent external availability check. If the entire M70q or home Internet connection becomes unavailable, the local Uptime Kuma instance may also be unable to send notifications.

### Pi-hole Web Interface

An HTTP monitor checks the Pi-hole web service through:

`http://192.168.1.253/admin/`

This verifies that the Pi-hole HTTP service is responding rather than only checking whether the host itself responds to ICMP.

### Pi-hole DNS

A DNS monitor sends a DNS query through Pi-hole at:

```text
Resolver: 192.168.1.253
Port:     53
Record:   A
```

The monitor resolves a public hostname through Pi-hole.

This provides an application-level DNS check and verifies that Pi-hole can answer DNS queries.

### Internet Connectivity

A Ping monitor checks:

`1.1.1.1`

This provides a simple indication that the M70q and Uptime Kuma container can reach an external Internet address.

Because notifications are also delivered through the Internet, a complete ISP outage may prevent the local Uptime Kuma instance from sending a Discord notification about the outage.

An external monitoring service would be required for independent detection and notification of a complete home Internet outage.

### External DNS

A DNS monitor queries Cloudflare's resolver at:

`1.1.1.1:53`

This provides a DNS test independent of the local Pi-hole resolver.

Comparing the Pi-hole DNS and External DNS monitors can help distinguish between a local DNS problem and a broader Internet or external DNS connectivity problem.

## Notifications

Discord is configured as the notification channel for Uptime Kuma.

Uptime Kuma sends monitor status notifications to Discord through a webhook.

The Discord webhook URL is treated as a secret and is not stored in this public repository.

## Firewall Configuration

UFW uses a default-deny incoming policy on the M70q.

During testing, Uptime Kuma could ping the M70q at `192.168.1.253`, but its HTTP monitor for Pi-hole timed out.

Testing from inside the container confirmed that ICMP connectivity worked:

```text
Uptime Kuma → 192.168.1.253 → ICMP reachable
```

However, an HTTP connection to TCP port `80` timed out.

Pi-hole was confirmed to be listening on TCP port `80` on all IPv4 interfaces. UFW logs then showed packets from the Uptime Kuma container being blocked:

```text
SRC=172.18.0.2
DST=192.168.1.253
PROTO=TCP
DPT=80
```

The Docker network was identified as `172.18.0.0/16`, and UFW was configured to permit that subnet to access TCP port `80` on the M70q.

The Pi-hole DNS monitor encountered the same firewall boundary. UFW logs showed DNS queries from the container being blocked on UDP port `53`.

Access was therefore permitted from the Docker subnet to DNS on both UDP and TCP port `53`.

The resulting access requirement is:

```text
172.18.0.0/16 → TCP/80 → Pi-hole web interface
172.18.0.0/16 → UDP/53 → Pi-hole DNS
172.18.0.0/16 → TCP/53 → Pi-hole DNS
```

These rules permit only the required service ports rather than unrestricted access from the Docker subnet to the host.

The Docker subnet is currently assigned dynamically by Docker. If the Compose network is recreated with a different subnet in the future, these UFW rules may need to be updated or the Docker network may need to be assigned a fixed subnet.

## Verification

Uptime Kuma was verified through several tests.

The container reported a healthy status through Docker:

```text
uptime-kuma → healthy
```

The Pi-hole HTTP monitor successfully reached the native Pi-hole web service after the required UFW rule was added.

The Pi-hole DNS monitor successfully resolved a public hostname through `192.168.1.253:53` after DNS access from the Docker network was permitted.

The Internet Ping and External DNS monitors successfully reached external services.

A full M70q reboot was performed to verify service persistence. After reboot:

```text
Pi-hole      → active
Unbound      → active
Tailscale    → connected
Uptime Kuma  → running and healthy
```

Uptime Kuma restarted automatically, and its existing monitors and configuration remained available through the persistent Docker volume.

## Troubleshooting Notes

### Pi-hole HTTP Monitor Timed Out

The Pi-hole HTTP monitor initially reported a timeout even though the Pi-hole web interface was reachable from LAN clients.

A ping from inside the Uptime Kuma container to `192.168.1.253` succeeded, proving basic IP connectivity.

A direct HTTP request from inside the container then timed out.

Pi-hole was confirmed to be listening on `0.0.0.0:80`, eliminating the web server's listening address as the cause.

UFW logs provided the decisive evidence:

```text
[UFW BLOCK]
SRC=172.18.0.2
DST=192.168.1.253
PROTO=TCP
DPT=80
```

After TCP port `80` was allowed from the Docker subnet, the HTTP request succeeded and the Uptime Kuma monitor changed to Up.

### Pi-hole DNS Monitor Was Blocked

The Pi-hole DNS monitor initially reported Down.

UFW logs showed DNS packets from the Uptime Kuma container being blocked:

```text
[UFW BLOCK]
SRC=172.18.0.2
DST=192.168.1.253
PROTO=UDP
DPT=53
```

After DNS access on port `53` was permitted from the Docker subnet, the monitor changed to Up.

### Direct Unbound Monitoring

Unbound is intentionally bound to:

`127.0.0.1:5335`

This keeps the recursive resolver accessible only locally to services such as Pi-hole.

Because `127.0.0.1` inside the Uptime Kuma container refers to the container itself rather than the M70q host, Uptime Kuma does not currently monitor Unbound directly.

The localhost-only Unbound design was left unchanged rather than exposing the resolver solely for monitoring purposes.
