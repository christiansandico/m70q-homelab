# Stirling PDF

Self-hosted deployment of Stirling PDF running in Docker Compose on my Ubuntu Server homelab.

## Overview

Stirling PDF is a self-hosted web application for working with PDF files. It is deployed as a Docker container on my Lenovo ThinkCentre M70q server.

The deployment is accessible through:

- Local network (LAN)
- Tailscale for remote access

The service is not intentionally exposed directly to the public Internet.

## Deployment

The container is managed using Docker Compose.

Directory structure:

```text
docker/stirling-pdf/
├── compose.yaml
└── stirling-data/    # Persistent application data (ignored by Git)
```

The Compose configuration publishes Stirling PDF on both the server's LAN and Tailscale addresses:

```yaml
ports:
  - "192.168.1.253:8080:8080"
  - "100.66.59.119:8080:8080"
```

This explicitly limits Docker's port bindings to the intended interfaces instead of using the default wildcard binding (`0.0.0.0:8080`).

## Persistent Storage

Stirling PDF stores persistent configuration data using a bind mount:

```yaml
volumes:
  - ./stirling-data:/configs
```

The local `stirling-data` directory is mounted to `/configs` inside the container.

This allows application configuration and account data to survive container removal and recreation.

Because this directory contains application state, it is excluded from the public Git repository using `.gitignore`.

## Access

### LAN

```text
http://192.168.1.253:8080
```

### Remote Access

Remote access is provided through Tailscale:

```text
http://100.66.59.119:8080
```

No router port forwarding is configured for Stirling PDF.

## Network Exposure

The initial Compose configuration used:

```yaml
ports:
  - "8080:8080"
```

Docker published this as:

```text
0.0.0.0:8080
[::]:8080
```

This meant port 8080 was bound to all IPv4 and IPv6 addresses on the host.

The deployment was later restricted to the two interfaces that require access:

```yaml
ports:
  - "192.168.1.253:8080:8080"
  - "100.66.59.119:8080:8080"
```

Verification with `docker port` confirmed that the container is now published specifically on the LAN and Tailscale addresses rather than wildcard addresses.

This is particularly relevant because Docker-published ports use Docker-managed forwarding/firewall rules and should not be assumed to behave like native services protected by UFW's normal incoming rules.

## Monitoring

Stirling PDF is monitored by Uptime Kuma using an HTTP(s) monitor:

```text
http://192.168.1.253:8080
```

The monitor successfully reports the service as available.

## Verification

The deployment was tested and verified for:

- Successful container startup
- Web UI accessibility over LAN
- Administrator authentication
- Persistent configuration across `docker compose down` and `docker compose up -d`
- LAN access from another device
- Remote access through Tailscale over mobile data
- Explicit LAN and Tailscale Docker port bindings
- Uptime Kuma monitoring
- Persistent application data excluded from Git

The same deployment configuration was also tested in an Ubuntu Server development VM before being deployed to the production server.

## Management

From the Stirling PDF directory:

Start the service:

```bash
docker compose up -d
```

Stop and remove the container:

```bash
docker compose down
```

Check container status:

```bash
docker compose ps
```

View logs:

```bash
docker compose logs
```

Validate the Compose configuration:

```bash
docker compose config
```

Check published ports:

```bash
docker port stirling-pdf
```

## Security Notes

- Administrator credentials are not stored in the Git repository.
- Persistent application data is excluded from Git.
- Docker ports are explicitly bound to the LAN and Tailscale addresses.
- No router port forwarding is configured for Stirling PDF.
- Remote access is provided through Tailscale rather than exposing port 8080 directly to the Internet.
