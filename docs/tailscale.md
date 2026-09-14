## Purpose

Tailscale provides secure remote access to the homelab without exposing services directly to the public Internet.

The M70q is connected to the Tailscale network and can be reached remotely through its Tailscale address at `100.66.59.119`.

Tailscale serves three main purposes in the homelab:

- Providing remote access to services hosted on the M70q.
- Allowing remote devices to use the Pi-hole and Unbound DNS stack.
- Allowing the M70q to operate as an optional exit node for Internet traffic.

This allows devices such as my PC and phone to securely access homelab services and use Pi-hole DNS filtering while away from the local network.

## Installation

Tailscale is installed natively on the Ubuntu Server host rather than running inside a Docker container.

The official Tailscale installation script was reviewed before being used to install the required packages and repository configuration.

After installation, the M70q was authenticated with the Tailscale network and assigned a stable Tailscale IPv4 address of `100.66.59.119`.

Tailscale runs through the `tailscaled` daemon, which is managed by systemd and enabled to start automatically at boot.

This keeps remote connectivity available independently of Docker and other application workloads running on the server.

## Remote DNS

Tailscale allows remote devices to use the Pi-hole and Unbound DNS stack even when they are outside the local network.

The M70q's Tailscale IPv4 address, `100.66.59.119`, is configured as the global nameserver in the Tailscale DNS settings with **Override DNS** enabled.

When a Tailscale client performs a DNS lookup, Tailscale's local DNS integration handles the request and follows the tailnet DNS configuration. Queries are ultimately sent to Pi-hole on the M70q, which filters them before forwarding allowed queries to Unbound.

The remote DNS path is:

```text
Remote client
    ↓
Tailscale DNS
    ↓
100.66.59.119:53
    ↓
Pi-hole
    ↓
127.0.0.1:5335
    ↓
Unbound
```

This allows the same Pi-hole filtering and Unbound recursive resolution used on the local network to be available to devices connected through Tailscale.

## Exit Node

The M70q is configured to operate as an optional Tailscale exit node. This allows a remote Tailscale device to route its Internet traffic through the M70q and the home Internet connection.

IP forwarding is enabled on the M70q for both IPv4 and IPv6 so that the server can forward traffic between Tailscale and other network interfaces.

The persistent forwarding configuration is stored in:

`/etc/sysctl.d/99-tailscale.conf`

```text
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
```

The M70q advertises itself as an exit node to the tailnet. Remote devices can choose whether to use it; simply connecting to Tailscale does not require all Internet traffic to pass through the M70q.

When the exit node is selected, the general traffic path is:

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

DNS remains configured separately through Tailscale's DNS settings, allowing remote clients to continue using Pi-hole and Unbound while the M70q is selected as the exit node.

## Verification

Tailscale connectivity was tested from both Windows and iPhone clients.

The M70q was reachable through its Tailscale address at `100.66.59.119`, confirming connectivity between devices on the tailnet.

Remote DNS was verified by performing DNS lookups while connected to Tailscale. Queries were successfully handled through the M70q's Pi-hole and Unbound DNS stack, and Pi-hole filtering continued to work remotely.

The exit node was tested from an iPhone using cellular data. After selecting the M70q as the exit node, Internet access continued to work and the device's observed public IP changed to the public IP of the home Internet connection.

These tests verified the two main Tailscale traffic paths:

```text
Without exit node:
Remote device → Tailscale → Homelab services / DNS
Internet traffic → Device's normal Internet connection

With M70q exit node:
Remote device → Tailscale → M70q → Home router → Internet
```

## Troubleshooting Notes

### systemd Unit Warning

During testing, stopping and starting `tailscaled` produced a warning that the service unit file or its configuration had changed on disk.

The service definition was inspected and no unexpected drop-in configuration was found. `systemctl daemon-reload` was used to make systemd reread its unit definitions.

This cleared the warning without requiring changes to the Tailscale configuration.
