## Purpose

UFW (Uncomplicated Firewall) is used as the host firewall on the M70q to control which network traffic is allowed to reach services running on the server.

The firewall follows a default-deny approach for incoming traffic. Services are only made accessible when a firewall rule explicitly permits the required traffic.

Rules are designed around the two trusted network paths used by the homelab:

- The local network through `eno1`.
- The Tailscale network through `tailscale0`.

This allows services such as Pi-hole and its web interface to be reachable from trusted LAN and Tailscale clients without unnecessarily exposing them to other networks.

## Default Policy

UFW is enabled on the M70q with a default-deny policy for incoming traffic.

The firewall uses the following default behavior:

```text
Incoming traffic:  DENY
Outgoing traffic:  ALLOW
Forwarded traffic: DENY
```

## Allowed Traffic

Firewall rules explicitly allow access to the services required by the homelab.

### SSH

SSH is allowed so that the M70q can be administered remotely. The current OpenSSH rule permits SSH traffic more broadly than the service-specific LAN and Tailscale rules.

This rule can be restricted further in the future so that SSH is only accessible through trusted network paths.

### DNS

Pi-hole requires both UDP and TCP port `53`.

DNS access is permitted from:

- The local `192.168.1.0/24` network through `eno1`.
- Tailscale clients through `tailscale0`.

This allows Pi-hole to use the `ALL` listening mode while UFW restricts which networks can actually reach the DNS service.

### Web Access

TCP ports `80` and `443` are permitted from the local network and through Tailscale.

These rules allow trusted clients to access web interfaces and services hosted on the M70q without making those ports generally accessible from untrusted networks.

## Tailscale

The `tailscale0` interface is treated as a trusted path for remote access to selected homelab services.

UFW explicitly allows DNS traffic on TCP and UDP port `53` through `tailscale0`, allowing remote Tailscale clients to reach Pi-hole.

TCP ports `80` and `443` are also allowed through `tailscale0` for access to web interfaces and services hosted on the M70q.

Tailscale exit-node traffic is separate from normal connections to services hosted directly on the M70q. Linux IP forwarding is enabled so the M70q can operate as an exit node, while Tailscale manages the networking required to route exit-node traffic.

The firewall therefore serves two different purposes:

```text
Traffic TO the M70q
    ↓
UFW controls access to services
(DNS, web interfaces, SSH)

Traffic THROUGH the M70q as an exit node
    ↓
Linux IP forwarding + Tailscale networking
    ↓
Home router → Internet
```

## Verification

Firewall behavior was verified while testing services from both the local network and Tailscale.

DNS queries from trusted LAN clients successfully reached Pi-hole through `192.168.1.253` on port `53`. DNS queries from remote Tailscale clients also successfully reached Pi-hole through the `tailscale0` interface.

The Pi-hole web interface was reachable from permitted network paths, confirming that the required web traffic was allowed through the firewall.

Tailscale exit-node functionality was also tested while UFW remained enabled. Internet traffic successfully passed through the M70q when it was selected as an exit node, confirming that the host firewall configuration did not prevent Tailscale's exit-node functionality.

These tests verified the intended behavior:

```text
Trusted LAN → permitted services       ✓
Tailscale   → permitted services       ✓
Unsolicited incoming traffic by default → denied
Tailscale exit-node forwarding         ✓
```

## Security Notes

The firewall configuration follows the principle of exposing only the services required by the homelab.

Pi-hole is configured to listen for DNS queries beyond only local interfaces so that it can serve Tailscale clients. Access to DNS is therefore restricted by UFW to the trusted LAN and Tailscale network paths.

The current OpenSSH firewall rule is broader than the other service-specific rules. A future hardening improvement is to restrict SSH access to the local network and Tailscale rather than allowing it generally.

Future services should not automatically be exposed when installed. Before adding a firewall rule, the required port, protocol, source network, and intended clients should be identified.

Firewall changes should follow the general process:

```text
Install or configure service
        ↓
Determine required port/protocol
        ↓
Determine which clients need access
        ↓
Add the minimum required firewall rule
        ↓
Verify connectivity
```

