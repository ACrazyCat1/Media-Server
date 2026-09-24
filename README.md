# Media Server Stack

A self-hosted media stack built around Jellyfin, with automated movie/TV acquisition (Radarr/Sonarr/Prowlarr via qBittorrent behind a VPN), a dedicated music server (Navidrome + Feishin), AI-powered music analysis (AudioMuse-AI), and a request manager (Seerr).

## Services

| Service | Purpose | Port | Notes |
|---|---|---|---|
| **jellyfin** | Media server (movies, TV, etc.) | `8096` (+ `7359/udp` for discovery) | GPU-accelerated transcoding via NVIDIA |
| **navidrome** | Music streaming server (Subsonic API) | `4533` | Read-only access to `Music` library |
| **feishin** | Desktop/web music client | `9180` | Configured to talk to Navidrome by default |
| **seerr** | Media request management UI | `5055` | Talks to Jellyfin/Radarr/Sonarr |
| **gluetun** | VPN gateway container | `8080`, `6881` (+udp), `8888` | All torrent traffic is routed through this |
| **qbittorrent** | Torrent client | `8080` (via gluetun's network) | Runs inside gluetun's network namespace — no traffic leaves outside the VPN |
| **prowlarr** | Indexer manager | `9696` | Proxies its own traffic through gluetun's HTTP proxy |
| **flaresolverr** | Cloudflare bypass proxy for indexers | `8191` | Used by Prowlarr for protected indexers |
| **radarr** | Movie collection manager | `7878` | |
| **sonarr** | TV collection manager | `8989` | |
| **postgres** | Database for AudioMuse-AI | — (internal only) | |
| **audiomuse-ai-flask** | AudioMuse-AI web app | `8000` | GPU-accelerated music analysis/clustering |
| **audiomuse-ai-worker** | AudioMuse-AI background job worker | — (internal only) | Shares the Postgres DB and GPU with the Flask app |

## Architecture notes

- **VPN isolation:** `qbittorrent` uses `network_mode: service:gluetun`, so all of its traffic (including the WebUI) is tunneled through `gluetun`. It has no ports of its own — the `QBITTORRENT_PORT` mapping is defined on the `gluetun` service instead.
- **Indexer traffic:** `prowlarr` is *not* routed through gluetun's network namespace, but it is configured to use gluetun's HTTP proxy (`8888`) for outbound requests, with local services (`radarr`, `sonarr`, `flaresolverr`, `jellyfin`, `qbittorrent`) excluded via `NO_PROXY`.
- **GPU access:** `jellyfin`, `audiomuse-ai-flask`, and `audiomuse-ai-worker` all reserve NVIDIA GPU resources for transcoding / clustering respectively. This requires the [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) on the host.
- **Network:** Most services share the `jellyfin-network` bridge network.
- **AudioMuse-AI:** Uses a Postgres backend and separate Flask (web) / worker (background jobs) containers that share persistent volumes for downloaded/installed plugins and temp audio files.

## Prerequisites

- Docker and Docker Compose
- NVIDIA Container Toolkit (for GPU-accelerated services)
- A VPN account with a provider supported by [gluetun](https://github.com/qdm12/gluetun-wiki) (this stack is preconfigured for ProtonVPN)

## Setup

1. Copy `.env.example` to `.env` (or edit the provided `.env`) and fill in the values below.
2. Create the external network if it doesn't exist:
   ```bash
   docker network create webui-network
   ```
3. Make sure the paths referenced in `.env` (`DATA_LOCATION`, `MEDIA_STORAGE`, `TORRENT_DOWNLOADS`) exist and are writable by the `PUID`/`PGID` you configure.
4. Start the stack:
   ```bash
   docker compose up -d
   ```
5. On first run, visit each service's WebUI to complete setup:
   - Jellyfin: `http://localhost:8096`
   - Navidrome: `http://localhost:4533`
   - Prowlarr: `http://localhost:9696`
   - Radarr: `http://localhost:7878`
   - Sonarr: `http://localhost:8989`
   - qBittorrent: `http://localhost:8080`
   - Seerr: `http://localhost:5055`
   - AudioMuse-AI: `http://localhost:8000`

## Configuration (`.env`)

### General
- `TZ` — timezone for all containers (e.g. `America/New_York`).

### Jellyfin
- `JELLYFIN_SERVER_NAME` — display name.
- `JELLYFIN_SERVER_LOCAL_URL` — internal URL other containers (e.g. Feishin) use to reach Jellyfin.
- `JELLYFIN_SERVER_ADDRESS` — public/published URL, used for `JELLYFIN_PublishedServerUrl`.
- `JELLYFIN_SERVER_PORT` — host port (default `8096`).

### Navidrome
- `NAVIDROME_PORT` — host port (default `4533`).
- Agents (last.fm, Deezer, AudioMuse-AI) are enabled by default in the compose file via `ND_AGENTS`.

### Feishin
- `FEISHIN_SERVER_LOCK` — set `true` to prevent changing the server from the Feishin UI.
- `FEISHIN_SERVER_TYPE` — `jellyfin` or `navidrome`.
- `FEISHIN_SERVER_URL` — URL of whichever server type you chose.
- `FEISHIN_PORT` — host port (default `9180`).

### VPN (gluetun)
- `VPN_SERVICE_PROVIDER` — e.g. `protonvpn`. See [gluetun's supported providers](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers).
- `VPN_TYPE` — `openvpn` or `wireguard`.
- `OPENVPN_USER` / `OPENVPN_PASSWORD` — required if `VPN_TYPE=openvpn`.
- `WIREGUARD_PRIVATE_KEY` / `WIREGUARD_ADDRESSES` — required if `VPN_TYPE=wireguard`.
- `VPN_SERVER_COUNTRIES` — which VPN exit country/countries to use.
- `LAN_SUBNET` — your local subnet, whitelisted through gluetun's firewall so LAN devices can still reach qBittorrent's WebUI.

### Storage
- `MEDIA_STORAGE` — root path containing your `Movies`, `TV Shows`, and `Music` folders.
- `TORRENT_DOWNLOADS` — shared download directory for qBittorrent/Radarr/Sonarr.
- `DATA_LOCATION` — root path for all container config/data (`.jellyfin`, `.navidrome`, etc.).

### Passwords
- `PW` — general-purpose password placeholder (check which service consumes this before relying on it).
- `POSTGRES_PASSWORD` — Postgres password for AudioMuse-AI.

### Performance
- `USE_GPU_CLUSTERING` — enable GPU-accelerated clustering in AudioMuse-AI.

### Prowlarr Proxy
- `PROWLARR_HTTP_PROXY` / `PROWLARR_HTTPS_PROXY` — point at gluetun's proxy (`http://gluetun:8888`) so Prowlarr's indexer requests go through the VPN.

### Ports
All service ports are overridable and default to each service's standard port (see the table above). Adjust these in `.env` if you have conflicts on the host.

## Data layout

By default, all persistent config lives under `${DATA_LOCATION}` (e.g. `/srv/docker`), one subfolder per service (`.jellyfin`, `.navidrome`, `.seerr`, `.gluetun`, `.qbittorrent`, `.prowlarr`, `.flaresolverr`, `.radarr`, `.sonarr`). Media lives under `${MEDIA_STORAGE}`, expected to contain `Movies`, `TV Shows`, and `Music` subdirectories. Postgres data and AudioMuse-AI temp/plugin data are stored in named Docker volumes (`postgres-data`, `temp-audio-flask`, `temp-audio-worker`, `plugins-flask`, `plugins-worker`).

## Notes / TODO

- Lidarr and Readarr ports are stubbed out in `.env` but the services themselves aren't yet defined in `docker-compose.yml`.
- A `watchtower` service is stubbed but commented out — add it if you want automatic image updates.
- The old `jellyseerr` service definition is commented out in favor of `seerr`; remove it once you've confirmed the migration works.
- Double-check that `PW` is actually referenced somewhere you expect — it doesn't appear to be consumed by any service in the current compose file.
- Write a comprehensive Documentation in `docs/`.

## Security notes

- qBittorrent, Radarr, Sonarr, and Prowlarr WebUIs are unauthenticated by default on first run — set a password in each service immediately after first launch.
- `LAN_SUBNET` should be scoped as tightly as possible; it's whitelisted through gluetun's firewall.
- Don't commit your populated `.env` file (with real VPN credentials and passwords) to version control — keep a `.env.example` with placeholders instead.