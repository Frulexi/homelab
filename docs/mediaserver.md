# Media Server Stack

The media server stack (`dev_services/media_server/`) provides automated media acquisition, subtitle management, and streaming. It is currently deployed to the **staging** environment only.

## Services

| Service | Image | Port | Role |
| --- | --- | --- | --- |
| Jellyfin | `jellyfin/jellyfin:latest` | 8096 | Media server — streams movies and TV to any device |
| Jellyseerr | `fallenbagel/jellyseerr:latest` | 5055 | Request portal — users browse and request content here |
| Sonarr | `lscr.io/linuxserver/sonarr:latest` | 8989 | TV series manager — monitors, grabs, renames, and organises episodes |
| Radarr | `lscr.io/linuxserver/radarr:latest` | 7878 | Movie manager — same as Sonarr but for films |
| Prowlarr | `lscr.io/linuxserver/prowlarr:latest` | 9696 | Indexer hub — centralises indexer config and syncs it to Sonarr/Radarr |
| Bazarr | `lscr.io/linuxserver/bazarr:latest` | 6767 | Subtitle manager — watches libraries and downloads matching subtitles |
| qBittorrent | `lscr.io/linuxserver/qbittorrent:latest` | 8080 | Torrent downloader |
| SABnzbd | `lscr.io/linuxserver/sabnzbd:latest` | 8080 | Usenet downloader |

## How the pipeline works

```
User request → Jellyseerr → Sonarr / Radarr → Prowlarr (search indexers)
                                   ↓
                          qBittorrent / SABnzbd (download)
                                   ↓
                          Sonarr / Radarr (import & organise)
                                   ↓
                             Jellyfin (stream)
                                   ↓
                             Bazarr (subtitles)
```

1. A user submits a request through **Jellyseerr**.
2. Jellyseerr forwards the request to **Sonarr** (TV) or **Radarr** (movies).
3. Sonarr/Radarr query **Prowlarr** to search configured indexers for releases matching quality profiles.
4. The chosen release is sent to **qBittorrent** (torrent) or **SABnzbd** (Usenet) for download.
5. Once complete, Sonarr/Radarr import the file — renaming it and moving it into the appropriate library folder.
6. **Jellyfin** detects the new file and makes it available for streaming.
7. **Bazarr** monitors the libraries and automatically downloads subtitles for new content.

## Shared storage

All services share three host directories under `/opt/homelab/media/`. Using a single host path for each media type means file moves between services are instant (same filesystem, no copy).

| Host path | Container mount | Services |
| --- | --- | --- |
| `/opt/homelab/media/downloads` | `/data/downloads` | Sonarr, Radarr, qBittorrent, SABnzbd |
| `/opt/homelab/media/movies` | `/data/movies` | Jellyfin, Radarr, Bazarr |
| `/opt/homelab/media/tv` | `/data/tv` | Jellyfin, Sonarr, Bazarr |

Each service also has a named Docker volume (`<service>_config`) for its persistent configuration data.

## Networking

| Network | Purpose |
| --- | --- |
| `media_server_network` | Internal communication between all eight services |
| `traefik_network` | Exposes each service's web UI through Traefik with HTTPS |

Both networks are declared as `external: true` in the compose file and pre-created by Ansible via `docker_service_networks` in `ansible/inventory/staging/group_vars/homelab.yml`.

## Traefik routing

Every service is exposed over HTTPS with automatic Cloudflare DNS-challenge certificates:

| Service | URL |
| --- | --- |
| Jellyfin | `https://jellyfin.dev.landingsnet.com` |
| Jellyseerr | `https://jellyseerr.dev.landingsnet.com` |
| Sonarr | `https://sonarr.dev.landingsnet.com` |
| Radarr | `https://radarr.dev.landingsnet.com` |
| Prowlarr | `https://prowlarr.dev.landingsnet.com` |
| Bazarr | `https://bazarr.dev.landingsnet.com` |
| qBittorrent | `https://qbittorrent.dev.landingsnet.com` |
| SABnzbd | `https://sabnzbd.dev.landingsnet.com` |

## Environment

All services run as the same non-root user (`PUID=1000` / `PGID=1000`) to avoid file permission conflicts on the shared volumes. Timezone is set to `Europe/Madrid`. These values are delivered via Ansible Vault as a single `.env` file rendered from `docker_service_env_files.media_server`.

## Deployment

```bash
# Deploy only the media server to staging
ansible-playbook -i ansible/inventory/staging ansible/playbooks/deploy-services.yml \
  -e '{"docker_service_targets": ["media_server"]}'
```

The `docker_service` role copies `dev_services/media_server/` to `/opt/homelab/services/media_server/` on the staging host, renders the `.env` from vault, ensures the shared networks exist, and runs `docker compose up -d`.
