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
| Host DNS Resolver | 192.168.1.1 |

## Services
| Service | Address | Purpose |
| --- | --- | --- |
| Pi-hole | 192.168.1.253:53 | Network-wide DNS filtering |
| Unbound | 127.0.0.1:5335 | Local recursive DNS resolver with DNSSEC validation |

## DNS Architecture
When a client sends a DNS query to 192.168.1.253:53, the query reaches my home server where Pi-hole is listening on port 53. Pi-hole first checks whether it can answer the query locally, such as from its cache, and applies its filtering rules. If the requested domain is blocked, Pi-hole responds locally with 0.0.0.0 instead of forwarding the query.

If the domain is allowed and Pi-hole needs an upstream answer, it forwards the query to Unbound at 127.0.0.1:5335. Unbound recursively resolves the domain using the DNS hierarchy and validates DNSSEC where applicable. It returns the DNS answer to Pi-hole, which returns the result to the requesting client and can cache the response according to its TTL.

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
