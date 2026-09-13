# m70q-homelab
My documented Ubuntu Server homelab featuring Pi-hole, Unbound, Tailscale, Docker, monitoring, and networking labs.

## Network Configuration
| Field | Value |
| --- | --- |
| Hostname | m70q-srvr |
| Operating System | Ubuntu 26.04.1 LTS (Resolute Raccoon) |
| Server LAN IP | 192.168.1.253 |
| Subnet | /24 |
| Default Gateway | 192.168.1.1 |
| DNS Server | 192.168.1.1 |

## Services
| Service | Address | Purpose |
| --- | --- | --- |
| Pi-hole | 192.168.1.253:53 | Network-wide DNS filtering |
| Unbound | 127.0.0.1:5335 | Local recursive DNS resolver with DNSSEC validation |

