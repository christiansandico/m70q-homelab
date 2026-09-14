# Pi-hole

## Purpose

Pi-hole provides network-wide DNS filtering for the homelab. It runs natively on the M70q and listens for DNS queries on port `53`.

LAN clients can reach Pi-hole through the server's LAN address at `192.168.1.253`, while remote Tailscale clients can reach the same Pi-hole instance through the server's Tailscale address at `100.66.59.119`.

Pi-hole handles filtering but does not rely on a public upstream DNS provider for normal resolution. Allowed queries requiring upstream resolution are forwarded to the local Unbound resolver at `127.0.0.1:5335`.

The DNS path is:

Client → Pi-hole → Unbound → Authoritative DNS servers

## Installation

Pi-hole is installed natively on Ubuntu Server rather than running inside a Docker container.

A native installation was chosen to provide more hands-on experience with Linux services, systemd, configuration files, networking, ports, permissions, and logs. Docker is reserved for application workloads that will be added to the homelab later.

Pi-hole was installed using the official Pi-hole installation script after reviewing the script before execution.

The installation initially used Cloudflare as a temporary upstream DNS provider. After Unbound was installed and verified, Pi-hole's upstream DNS configuration was changed to the local Unbound resolver.

Pi-hole's DNS service runs through the `pihole-FTL` daemon, which runs in the background as a systemd service.

## DNS Configuration

Pi-hole listens for DNS queries on port `53` and uses Unbound as its only upstream DNS resolver.

The configured upstream resolver is:

`127.0.0.1#5335`

`127.0.0.1` is the loopback address of the M70q, so Pi-hole communicates with Unbound locally without exposing Unbound directly to the network. Unbound uses port `5335` to avoid conflicting with Pi-hole, which uses the standard DNS port `53`.

The DNS flow is:

```text
Client
  ↓
Pi-hole :53
  ↓
Unbound 127.0.0.1:5335
  ↓
Authoritative DNS servers
```

## Verification

Pi-hole was tested directly by sending DNS queries to the M70q at `192.168.1.253`.

A normal DNS query for `google.com` returned a valid response, confirming that Pi-hole could receive DNS requests and resolve allowed domains through Unbound.

Blocking was verified by querying a domain present on Pi-hole's blocklist. The blocked domain returned `0.0.0.0`, confirming that Pi-hole was applying its filtering rules instead of forwarding the request upstream.

DNS resolution was also tested remotely through Tailscale using the M70q's Tailscale address at `100.66.59.119`. Both normal DNS resolution and Pi-hole filtering worked through this path.

These tests verified the complete DNS flow:

```text
Client → Pi-hole → Unbound → DNS resolution
              │
              └── Blocked domains are answered locally
```

## Troubleshooting Notes

### Tailscale DNS Queries Were Not Answered

Pi-hole initially accepted DNS queries only from locally permitted interfaces using the `LOCAL` listening mode. DNS queries from Tailscale clients reached the M70q but did not receive a response.

Packet capture testing confirmed that the DNS packets were successfully arriving through the Tailscale interface. This showed that Tailscale connectivity itself was working and narrowed the problem down to Pi-hole's listening policy.

After firewall rules were configured to restrict DNS access to trusted LAN and Tailscale clients, Pi-hole's listening mode was changed from `LOCAL` to `ALL`.

This allowed Pi-hole to answer DNS queries arriving through Tailscale while the firewall continued to control which clients could access port `53`.

### Unbound Availability During Boot

A Pi-hole log entry was observed shortly after a server reboot indicating that a connection to Unbound at `127.0.0.1:5335` had failed.

After boot completed, Unbound was running normally and DNS resolution through Pi-hole worked successfully. This suggests a possible service startup timing issue where Pi-hole attempted to contact Unbound before the resolver was fully ready.

The issue has not caused persistent DNS failures and has not required a configuration change.
