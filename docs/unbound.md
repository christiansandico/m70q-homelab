# Unbound

## Purpose

Unbound provides recursive DNS resolution for the homelab and acts as Pi-hole's upstream DNS resolver.

Instead of forwarding allowed DNS queries from Pi-hole to a public resolver such as Cloudflare or Google, Unbound performs recursive resolution by contacting the DNS hierarchy directly.

The normal DNS path is:

```text
Client
  ↓
Pi-hole :53
  ↓
Unbound 127.0.0.1:5335
  ↓
DNS hierarchy
  ├── Root servers
  ├── TLD servers
  └── Authoritative DNS servers
```

Unbound listens only on the M70q's loopback address at `127.0.0.1:5335`. This means it is not directly exposed to LAN or Tailscale clients; Pi-hole is the DNS service clients interact with.

## Installation

Unbound is installed natively on Ubuntu Server using the system package manager.

Like Pi-hole, Unbound was installed directly on the host rather than inside a Docker container. This keeps the core DNS infrastructure independent of Docker and provides direct experience with Linux services, configuration files, ports, and logs.

Unbound runs as a systemd service and starts automatically when the M70q boots.

After installation, a custom configuration was created for the Pi-hole and Unbound integration. Unbound was configured to listen only on `127.0.0.1` using port `5335`.

## Configuration

The custom Unbound configuration is stored at:

`/etc/unbound/unbound.conf.d/pi-hole.conf`

Unbound is configured to listen only on the M70q's loopback interface:

```text
interface: 127.0.0.1
port: 5335
```

IPv4, UDP, and TCP DNS queries are enabled. Native IPv6 resolution is currently disabled because working IPv6 Internet connectivity is not available on the network.

Several security and reliability options are also enabled, including DNSSEC hardening, prefetching, and an EDNS buffer size of `1232` bytes.

Private IPv4 and IPv6 address ranges are defined using `private-address` directives. This helps prevent public DNS responses from unexpectedly returning addresses belonging to private network ranges.

## DNSSEC

Unbound performs DNSSEC validation to verify the authenticity and integrity of signed DNS responses.

DNSSEC validation was tested using `dnssec.works`.

A correctly signed domain returned a successful response with the `AD` (Authenticated Data) flag, confirming that Unbound successfully validated the DNSSEC signatures.

A deliberately broken DNSSEC domain, `fail01.dnssec.works`, returned `SERVFAIL`. This confirmed that Unbound rejected a response that could not be successfully validated.

Together, these tests confirmed that DNSSEC validation is functioning correctly.

## Verification

Unbound was tested directly on the M70q by sending DNS queries to `127.0.0.1` on port `5335`.

Successful responses confirmed that Unbound was listening on the expected address and port and could perform recursive DNS resolution independently of Pi-hole.

The complete Pi-hole and Unbound integration was then verified by sending DNS queries through Pi-hole. Allowed queries were successfully forwarded to Unbound and returned valid DNS responses.

This verified both layers of the DNS stack:

```text
Direct test:
dig → Unbound :5335 → DNS resolution

Integrated test:
Client → Pi-hole :53 → Unbound :5335 → DNS resolution
```

## Troubleshooting Notes

### Port 53 Conflict with Pi-hole

After Unbound was initially installed, the service failed to start because its default configuration attempted to listen on port `53`.

Pi-hole was already using port `53` for DNS, so both services could not bind to the same address and port.

Service status and logs showed that Unbound was attempting to bind to `::1:53`, which identified the port conflict.

Unbound was then configured to listen only on the IPv4 loopback address `127.0.0.1` using port `5335`:

```text
interface: 127.0.0.1
port: 5335
```

This separated the roles of the two DNS services:

```text
Clients
   ↓
Pi-hole :53
   ↓
Unbound 127.0.0.1:5335
```

After the configuration change, Unbound started successfully and DNS resolution through Pi-hole was verified.
