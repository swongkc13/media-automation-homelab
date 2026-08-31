# Docker Media Automation Homelab

A general-purpose, Docker-based media automation stack running on a Linux virtual machine hosted by Proxmox. It provides a clean request-to-library workflow while keeping the download client behind a VPN network namespace.


## Architecture

```text
Users
  |
  v
Seerr ────────────────> Jellyfin (authentication and library discovery)
  |                         ^
  |                         |
  +--> Radarr (movies) -----+----> /data/media/movies
  |
  +--> Sonarr (TV) ---------+----> /data/media/tv
            ^
            |
       Prowlarr (indexer management)
            |
            v
 qBittorrent (download client, shares Gluetun network)
            |
            v
       Gluetun (WireGuard/OpenVPN VPN tunnel and kill switch)
            |
            v
     /data/downloads/{incomplete,complete}

FlareSolverr is available only to Docker-internal services when an indexer requires it.
```

### Services

| Service | Role |
| --- | --- |
| Proxmox VE | Hosts the Linux VM and provides virtualized compute, storage, and networking. |
| Docker Compose | Defines and operates the application stack declaratively. |
| Gluetun | Provides the VPN tunnel, firewall/kill switch, and network namespace for qBittorrent. |
| qBittorrent | Downloads content to the shared downloads volume. |
| Prowlarr | Centrally manages indexers and synchronizes them with Radarr and Sonarr. |
| Radarr | Searches, monitors, imports, and organizes movies. |
| Sonarr | Searches, monitors, imports, and organizes TV series and episodes. |
| Seerr | Lets users request movies and TV; passes approved requests to Radarr/Sonarr. |
| Jellyfin | Serves the completed media libraries. |
| FlareSolverr | Optional helper for compatible Cloudflare-protected indexers; not exposed publicly. |

## Suggested infrastructure

The exact sizing depends on concurrent transcodes, library size, and download activity. A practical starting point is:

| Layer | Suggested starting point |
| --- | --- |
| Proxmox host | 4+ CPU cores, 16 GB RAM, SSD for VM disks; separate high-capacity storage for media where possible. |
| Linux VM | 2–4 vCPU, 4–8 GB RAM, 40–80 GB system disk, and access to persistent media/download storage. |
| Operating system | A current, supported minimal Debian or Ubuntu Server release. |
| Network | Static DHCP reservation or another stable private address; only publish ports intentionally needed on the LAN. |
| Media storage | Persistent filesystem mounted at `/data`; avoid storing libraries in the VM's ephemeral root filesystem. |

For hardware-accelerated Jellyfin transcoding, pass through or expose the appropriate GPU/iGPU device to the VM and container, following the documentation for the hardware and operating system.

## Proxmox and Linux VM setup

1. Create a Linux VM in Proxmox using the suggested resources above.
2. Install a supported server OS and apply operating-system updates.
3. Install Docker Engine with the Docker Compose plugin using the official Docker documentation for the chosen distribution.
4. Create a dedicated, non-root account for administration and add it to the `docker` group if appropriate for the environment.
5. Attach or mount persistent storage at `/data` and ensure it mounts automatically after reboot.
6. Create the directory layout below and grant the container user/group IDs access to it.

```bash
sudo mkdir -p /data/media/movies /data/media/tv
sudo mkdir -p /data/downloads/complete/movies /data/downloads/complete/tv
sudo mkdir -p /data/downloads/incomplete
sudo mkdir -p /opt/media-stack
```

Use consistent `PUID` and `PGID` values in the Compose environment and set ownership/permissions to match. Confirm the mount is available before starting Docker; otherwise Docker can create an empty directory on the VM's root disk and applications may write media to the wrong place.

## Storage and path mappings

All services that exchange files should see the same host paths and the same container paths. This prevents import failures caused by different views of a completed download.

| Host path | Container path | Used by |
| --- | --- | --- |
| `/data/media/movies` | `/movies` | Radarr, Jellyfin |
| `/data/media/tv` | `/tv` | Sonarr, Jellyfin |
| `/data/downloads` | `/downloads` | qBittorrent, Radarr, Sonarr |
| `/opt/media-stack/config/<service>` | `/config` | Respective service |

Recommended qBittorrent directories:

```text
/downloads/incomplete
/downloads/complete/movies
/downloads/complete/tv
```

Set download categories to `movies` and `tv`, with matching category save paths. In Radarr and Sonarr, configure the download client and root folders using the container paths shown above.

## Docker Compose setup

Keep the Compose file, an example environment file, and per-service configuration together, for example:

```text
/opt/media-stack/
├── compose.yml
├── .env.example
└── config/
    ├── gluetun/
    ├── qbittorrent/
    ├── prowlarr/
    ├── radarr/
    ├── sonarr/
    ├── seerr/
    ├── jellyfin/
    └── flaresolverr/
```

Use a `.env` file excluded by `.gitignore` for values such as timezone, IDs, VPN provider settings, VPN credentials, and service secrets. Commit only `.env.example` with empty or clearly fake placeholders.

Typical lifecycle commands:

```bash
cd /opt/media-stack
docker compose config
docker compose up -d
docker compose ps
docker compose logs -f --tail=100
docker compose pull
docker compose up -d
```

Before upgrades, back up the `config` directory and any Compose/environment files. Review release notes and update one logical group at a time where practical.

## VPN and qBittorrent networking

The important isolation pattern is for qBittorrent to share Gluetun's network stack. In Compose, this is normally expressed as:

```yaml
qbittorrent:
  network_mode: "service:gluetun"
  depends_on:
    gluetun:
      condition: service_healthy
```

Publish the qBittorrent web UI port on the `gluetun` service, not on qBittorrent itself. Gluetun should be configured with a provider-supported WireGuard or OpenVPN connection and its firewall/kill-switch enabled. The download client should have no direct network configuration that bypasses Gluetun.

Keep port forwarding optional and provider-dependent. If it is enabled, configure it only through documented Gluetun/provider mechanisms and do not commit the resulting settings or credentials.

## Application configuration order

1. Start the stack and confirm all containers are healthy.
2. Configure qBittorrent's incomplete directory, completed base directory, and `movies`/`tv` categories.
3. Add indexers to Prowlarr and test each one.
4. In Prowlarr, add Radarr and Sonarr as applications. Supply their internal Docker service URLs and API keys through the application UI; never commit those keys.
5. In Radarr, add qBittorrent as the movie download client. Add `/movies` as a root folder and confirm importing is enabled.
6. In Sonarr, add qBittorrent as the TV download client. Add `/tv` as a root folder and confirm importing is enabled.
7. Configure Jellyfin libraries to use `/movies` and `/tv`.
8. Connect Seerr to Jellyfin for user/library data, then to Radarr and Sonarr for requests. Test a request through to the relevant *arr application.
9. Add FlareSolverr only when an explicitly supported indexer needs it. Keep it on an internal Docker network with no host port published.

Prefer Docker service names (for example `http://radarr:7878`) for communication between containers. This avoids hard-coded host addresses and keeps traffic on the Docker network.

## Security considerations

- Never commit `.env`, Compose files containing secrets, application backups, database files, or API keys. Add them to `.gitignore`.
- Use strong, unique passwords for every web interface. Disable default credentials immediately.
- Do not expose service dashboards directly to the public internet. Prefer LAN-only access, a trusted reverse proxy with authentication, or a private overlay network.
- Restrict published ports to those that are genuinely required. FlareSolverr should remain internal.
- Keep the OS, Docker images, and application images current, while reviewing changes before upgrades.
- Back up configuration and document how to restore it. Treat configuration directories as operational state.
- Apply least privilege to filesystem permissions. Avoid world-writable media or configuration directories.
- Verify the VPN kill switch after configuration changes and upgrades.

## Validation and testing

Run these checks after deployment and after meaningful changes:

```bash
# Validate rendered Compose configuration and current containers
docker compose config
docker compose ps
docker ps --format 'table {{.Names}}\t{{.Ports}}\t{{.Status}}'

# Confirm download directories are visible in the download client
docker exec qbittorrent sh -c 'ls -ld /downloads/complete/movies /downloads/complete/tv /downloads/incomplete'

# Confirm the VPN egress address from both network-sharing containers
docker exec gluetun sh -c 'wget -qO- --timeout=10 https://api.ipify.org && echo'
docker exec qbittorrent sh -c 'wget -qO- --timeout=10 https://api.ipify.org && echo'
```

For a controlled kill-switch test, temporarily stop Gluetun, then confirm qBittorrent cannot reach the internet. Restart Gluetun and qBittorrent afterward:

```bash
docker stop gluetun
docker exec qbittorrent sh -c 'wget -qO- --timeout=5 https://api.ipify.org && echo'
docker start gluetun
docker restart qbittorrent
```

The connectivity command should fail while Gluetun is stopped. Do not run this during active downloads. Also validate one movie and one TV workflow end-to-end: request/search, queue, download, import, and Jellyfin library scan.

## Troubleshooting

| Symptom | Checks and likely resolution |
| --- | --- |
| Radarr or Sonarr cannot import a completed download | Confirm both the download client and the relevant *arr app mount `/data/downloads` as `/downloads`. Check category paths and filesystem ownership. |
| Completed download appears in the wrong app | Verify qBittorrent categories are exactly `movies` and `tv`, and that each category's save path matches the intended completed directory. |
| qBittorrent UI is unreachable | With `network_mode: service:gluetun`, publish the UI port through Gluetun. Check Gluetun logs and allowed inbound ports. |
| qBittorrent has internet access when Gluetun is down | Stop and inspect the stack immediately. Confirm qBittorrent shares Gluetun's network namespace and has no separate ports/network configuration. Re-run the kill-switch test. |
| Prowlarr sync fails | Check the application URL from Prowlarr, the API key, DNS/service-name resolution on the Docker network, and Prowlarr logs. |
| Jellyfin sees an empty library | Verify host mounts are present, the `/movies` and `/tv` paths are mapped into Jellyfin, and the library paths use container paths. Trigger a library scan. |
| Permission denied errors | Compare ownership and mode bits on `/data` with the `PUID`/`PGID` used by containers. Avoid fixing this with overly permissive `777` permissions. |
| Containers restart repeatedly | Inspect service logs, validate the rendered Compose configuration, and check disk space/memory on the VM. |

Useful diagnostics:

```bash
docker compose logs --tail=200 gluetun qbittorrent radarr sonarr prowlarr
docker inspect qbittorrent
find /data -maxdepth 3 -type d | sort
ls -ld /data/media /data/media/movies /data/media/tv
ls -ld /data/downloads /data/downloads/complete /data/downloads/incomplete
df -h
```

## Repository hygiene

Recommended tracked files:

```text
README.md
compose.yml
.env.example
.gitignore
docs/
```

Recommended `.gitignore` entries:

```gitignore
.env
config/
*.db
*.log
backups/
```

Adjust this list if selected services require non-secret, curated configuration files to be version controlled. Keep secrets and live application data out of the repository.

## Disclaimer

Use this stack only with media and sources you are authorized to access. Follow applicable laws, service terms, and network policies.
