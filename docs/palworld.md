# Palworld Dedicated Server

## Purpose

Palworld runs as a private dedicated game server on the M70q.

The server is deployed with Docker Compose and is intended for a small number of trusted players. Local players can connect through the home network, while authorized remote players can connect through Tailscale without exposing the game server through router port forwarding.

The deployment includes persistent world data, automatic backups, automatic idle pausing, password-protected access, and Uptime Kuma monitoring.

## Docker Deployment

The Docker Compose configuration is stored at:

`docker/palworld/compose.yaml`

The server uses the image:

`thijsvanloef/palworld-server-docker:latest`

The container is named:

`palworld-server`

The restart policy is:

`unless-stopped`

This allows the container to restart automatically with Docker unless it has been explicitly stopped.

The server publishes:

```text
8211/udp
27015/udp
```

Port `8211/udp` is used for game connections.

## Persistent Storage

The Compose deployment mounts:

```text
./palworld → /palworld/
```

Persistent server data is therefore stored on the host under:

`docker/palworld/palworld/`

This keeps world data and other server state outside the disposable container layer.

The persistent Palworld directory is excluded from Git because it contains generated runtime state, world data, player data, and backups.

## Network Access

### LAN

Players on the local network connect to:

```text
192.168.1.253:8211
```

LAN connectivity was tested successfully.

### Tailscale

Authorized remote players can connect through:

```text
100.66.59.119:8211
```

Remote connectivity through Tailscale was tested successfully.

The resulting path is:

```text
Remote player
     ↓
Tailscale
     ↓
100.66.59.119:8211
     ↓
M70q
     ↓
Docker
     ↓
Palworld
```

No router port forwarding is configured for Palworld.

Tailscale therefore provides remote access without directly exposing the game server to the public Internet.

## Server Configuration

The server is configured for two players.

Relevant configuration includes:

```yaml
PLAYERS: 2
DEATH_PENALTY: None
REST_API_ENABLED: true
ENABLE_PLAYER_LOGGING: true
AUTO_PAUSE_ENABLED: true
AUTO_PAUSE_TIMEOUT_EST: 300
DELETE_OLD_BACKUPS: true
OLD_BACKUP_DAYS: 14
TZ: "Asia/Manila"
COMMUNITY: false
CROSSPLAY_PLATFORMS: "(Steam,Xbox,PS5,Mac)"
```

`DEATH_PENALTY: None` prevents players from dropping items when they die.

Community mode is disabled because the server is intended to remain private.

## Secrets

The server password and administrator password are not stored directly in the public Compose configuration.

The Compose file references:

```yaml
SERVER_PASSWORD: ${SERVER_PASSWORD}
ADMIN_PASSWORD: ${ADMIN_PASSWORD}
```

The actual values are stored locally in:

`docker/palworld/.env`

The `.env` file is excluded from Git.

The relevant repository exclusions are:

```gitignore
docker/palworld/.env
docker/palworld/palworld/
```

This prevents server passwords, world data, generated server state, and backups from being committed to the public repository.

## REST API

The Palworld REST API is enabled internally.

It listens inside the container on:

```text
8212/tcp
```

The REST API is used by management functionality including player logging, save operations, and automatic pause.

Port `8212` is not published through Docker and is not intended for external access.

## Automatic Pause

Automatic pause is enabled to reduce server resource usage while nobody is playing.

The configured idle timeout is:

```text
300 seconds
```

or five minutes.

When no players remain connected, the server waits for the idle timeout, requests a world save through the REST API, and pauses the Palworld server process.

Testing confirmed the save and pause sequence:

```text
REST accessed endpoint /v1/api/save OK
[AUTO PAUSE] Paused.
```

The Docker container itself remains running while the Palworld process is paused.

When a player reconnects, the server automatically resumes.

Automatic wake was tested successfully through Tailscale.

### Resource Behavior

Automatic pause substantially reduces active CPU usage because the game world is no longer being actively simulated.

The Palworld process remains loaded in memory while paused, allowing it to resume without requiring a complete server startup.

The feature therefore primarily reduces active CPU usage rather than releasing all allocated memory.

## Saves and Backups

Palworld maintains its normal world and player saves during operation.

Automatic pause explicitly requests a save before suspending the server process.

Automatic backups are also enabled.

The generated backup schedule is:

```cron
0 0 * * *
```

With the server configured for the `Asia/Manila` time zone, backups run once per day at midnight.

Before creating a backup, the backup script requests a server save through the REST API.

Backups are stored inside:

```text
/palworld/backups/
```

which corresponds to the host directory:

`docker/palworld/palworld/backups/`

Backup files use names similar to:

```text
palworld-save-YYYY-MM-DD_HH-MM-SS.tar.gz
```

Backups older than 14 days are automatically removed.

The retention configuration is:

```yaml
DELETE_OLD_BACKUPS: true
OLD_BACKUP_DAYS: 14
```

A manual backup was successfully created during deployment testing, verifying the backup process end-to-end.

## Uptime Kuma Monitoring

Palworld is monitored by Uptime Kuma using its Docker Container monitor.

The monitoring path is:

```text
Uptime Kuma
     ↓
Docker socket proxy
     ↓
Docker Engine
     ↓
palworld-server
```

The monitor specifically targets the `palworld-server` container.

A restricted Docker socket proxy provides Uptime Kuma with the Docker API access required to inspect containers without mounting the Docker socket directly into the Uptime Kuma container.

Both Uptime Kuma and the proxy share the `kuma_network` Docker network.

Uptime Kuma connects to the proxy internally at:

```text
http://docker-socket-proxy:2375
```

The proxy's port is available only within the Docker network and is not published on the M70q host.

The proxy is configured to expose only the Docker API functionality required for container discovery and monitoring.

### Auto-Pause Monitoring Behavior

The Docker Container monitor was deliberately tested while Palworld entered its automatic paused state.

The Palworld game process paused successfully after five minutes without connected players while:

```text
palworld-server → running
Uptime Kuma     → Up
```

This is the intended behavior.

Uptime Kuma monitors whether the Palworld container remains available rather than treating the intentionally paused game process as an outage.

The monitor therefore remains green during normal automatic idle pauses but can report a problem if the Palworld container itself stops or becomes unavailable.

Discord notifications are enabled for the Palworld monitor.

Palworld is also included on the homelab Uptime Kuma status page.

## Management

Commands in this section are run from:

`~/m70q-homelab/docker/palworld`

### Check Container Status

```bash
docker compose ps
```

### Start or Apply Configuration

```bash
docker compose up -d
```

### Stop

```bash
docker compose down
```

Persistent data remains on the host when the container is removed.

### Validate Compose Configuration

```bash
docker compose config --quiet
```

No output indicates that Docker Compose accepted the configuration.

### Recent Logs

```bash
docker compose logs --tail=30
```

### Follow Logs

```bash
docker compose logs -f
```

`Ctrl+C` exits the log viewer without stopping the server.

### Watch Automatic Pause Events

```bash
docker compose logs -f | grep -i "AUTO PAUSE"
```

### Resource Usage

```bash
docker stats --no-stream palworld-server
```

### Manual Backup

```bash
docker compose exec palworld bash /usr/local/bin/backup
```

### List Backups

```bash
ls -lh palworld/backups/
```

## Security

The server is intended to remain private.

The current design includes:

- Password-protected server access
- Community server mode disabled
- No router port forwarding
- Remote access through Tailscale
- REST API not published through Docker
- Passwords stored outside the public Compose file
- Persistent world data excluded from Git
- Backups excluded from Git
- Docker monitoring performed through a restricted socket proxy

Docker-published ports are treated separately from native host services protected by UFW because Docker manages its own forwarding and firewall rules.

## Verification

The following were tested during deployment:

- Container installation and startup
- Docker health status
- LAN game connection through `192.168.1.253:8211`
- Tailscale game connection through `100.66.59.119:8211`
- Environment-based password loading
- Persistent server data
- No item loss on player death configuration
- Five-minute automatic pause
- Save operation before automatic pause
- Automatic wake after a connection attempt
- Daily automatic backup configuration
- Manual backup creation
- 14-day backup retention configuration
- Git exclusion of `.env`
- Git exclusion of persistent Palworld data
- Uptime Kuma Docker API connectivity through the socket proxy
- Palworld Docker Container monitor
- Uptime monitor remaining Up during automatic pause
- Discord monitor notification configuration
- Homelab status page integration
