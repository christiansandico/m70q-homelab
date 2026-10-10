# Stirling PDF

Self-hosted deployment of Stirling PDF running in Docker Compose on my Ubuntu Server homelab.

## Overview

Stirling PDF is a self-hosted web application for working with PDF files. It is deployed as a Docker container on my Lenovo ThinkCentre M70q server.

The deployment is accessible through:

- Local network (LAN)
- Tailscale for remote access

The service is not intentionally exposed directly to the public Internet.

## Deployment

The container is managed using Docker Compose, with systemd handling automatic startup after server reboots.

Directory structure:

```text
docker/stirling-pdf/
├── compose.yaml
├── systemd/
│   └── stirling-pdf.service
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

## Automatic Startup and Power-Outage Recovery

### Problem

On October 10, 2026, a power outage caused Stirling PDF to fail during automatic startup.

Docker attempted to restore the container before Tailscale had assigned the server's Tailscale IP address.

Docker reported:

```text
failed to bind host port 100.66.59.119:8080/tcp:
cannot assign requested address
```

Because Stirling PDF explicitly binds to both LAN and Tailscale addresses, both must be available before the container starts.

### Root Cause

The original Docker Compose configuration used:

```yaml
restart: unless-stopped
```

This allowed Docker to attempt automatic container restoration when the Docker daemon started.

However, Docker could start before Tailscale had finished initializing its network interface.

Since the required Tailscale IP address was unavailable, Docker could not establish the container's configured port bindings.

The container consequently failed to start correctly.

### Solution

A dedicated systemd service now manages Stirling PDF startup.

The service:

1. Starts after Docker and Tailscale services.
2. Waits until both required IPv4 addresses are assigned.
3. Starts Stirling PDF using Docker Compose.
4. Retries startup if the IP readiness check fails.

The systemd configuration is stored at:

```text
docker/stirling-pdf/systemd/stirling-pdf.service
```

The service checks for both addresses:

```text
192.168.1.253
100.66.59.119
```

The readiness check runs every two seconds for up to approximately 120 seconds.

If the addresses remain unavailable, the startup attempt fails and systemd retries after ten seconds.

This allows Stirling PDF to recover from delayed network initialization without changing the startup behavior of other Docker containers.

### Installation

From the repository root:

```bash
cd ~/m70q-homelab
```

Copy the systemd service:

```bash
sudo cp docker/stirling-pdf/systemd/stirling-pdf.service \
  /etc/systemd/system/stirling-pdf.service
```

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable automatic startup:

```bash
sudo systemctl enable stirling-pdf.service
```

Start the service:

```bash
sudo systemctl start stirling-pdf.service
```

Verify:

```bash
systemctl status stirling-pdf.service --no-pager
docker compose -f docker/stirling-pdf/compose.yaml ps
```

The service uses:

```ini
Type=oneshot
RemainAfterExit=yes
```

Therefore, `active (exited)` is the expected systemd status after successful startup.

The service file contains an absolute `WorkingDirectory` path specific to this server. Adjust the path when deploying to another machine.

### Container Restart Policy

Docker Compose uses:

```yaml
restart: on-failure:5
```

This allows Docker to restart Stirling PDF up to five times after unexpected process failures.

Unlike `unless-stopped`, this policy does not automatically restart the container when the Docker daemon restarts.

Responsibilities are separated:

- **systemd:** Handles automatic startup after server reboots.
- **Docker:** Handles unexpected container process failures.

An unhealthy container that remains running is not automatically restarted by this policy.

### Recovery Verification

After implementing the systemd service, the server was rebooted on October 10, 2026.

Results:

- systemd IP readiness check completed successfully.
- Docker Compose started Stirling PDF automatically.
- The container reached healthy status.
- Both LAN and Tailscale port bindings were restored.
- Other Docker containers started normally.

Automatic startup after reboot was successfully verified.

## Management

### Systemd Management

For routine startup, shutdown, and restarts, use systemd.

**Start:**

```bash
sudo systemctl start stirling-pdf.service
```

**Stop:**

```bash
sudo systemctl stop stirling-pdf.service
```

**Restart:**

```bash
sudo systemctl restart stirling-pdf.service
```

**Check status:**

```bash
systemctl status stirling-pdf.service --no-pager
```

**View service logs:**

```bash
journalctl -u stirling-pdf.service -b
```

**Enable automatic startup:**

```bash
sudo systemctl enable stirling-pdf.service
```

**Disable automatic startup:**

```bash
sudo systemctl disable stirling-pdf.service
```

### Docker Compose Management

Docker Compose can still be used for troubleshooting, maintenance, and configuration updates.

From the Stirling PDF directory:

```bash
cd ~/m70q-homelab/docker/stirling-pdf
```

**Create or start the container:**

```bash
docker compose up -d
```

**Stop and remove the container:**

```bash
docker compose down
```

**Check container status:**

```bash
docker compose ps
```

**View container logs:**

```bash
docker compose logs
```

**Follow container logs:**

```bash
docker compose logs -f
```

**Validate the Compose configuration:**

```bash
docker compose config
```

**Check published ports:**

```bash
docker port stirling-pdf
```

**Check the Docker restart policy:**

```bash
docker inspect stirling-pdf \
  --format '{{.HostConfig.RestartPolicy.Name}}'
```

For routine operation, prefer systemd commands to avoid bypassing the configured startup dependency.

## Verification

The deployment was tested and verified for:

- Successful container startup
- Web UI accessibility over LAN
- Administrator authentication
- Persistent configuration across container recreation
- LAN access from another device
- Remote access through Tailscale over mobile data
- Explicit LAN and Tailscale Docker port bindings
- Uptime Kuma monitoring
- Persistent application data excluded from Git
- Automatic startup after server reboot
- Successful systemd IP readiness checks
- Docker restart policy configured for process failure recovery

The original deployment configuration was also tested in an Ubuntu Server development VM before being deployed to the production server.

## Security Notes

- Administrator credentials are not stored in the Git repository.
- Persistent application data is excluded from Git.
- Docker ports are explicitly bound to the LAN and Tailscale addresses.
- No router port forwarding is configured for Stirling PDF.
- Remote access is provided through Tailscale rather than exposing port 8080 directly to the Internet.
- Docker-published ports are managed through Docker networking and firewall rules.
- The systemd service ensures required host IP addresses are available before container startup.
