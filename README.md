# Media Automation Stack for Ugreen NAS

A complete, production-ready Docker Compose stack for automated media management with VPN routing, SSL certificates, and remote access.

**Tested on Ugreen NAS DXP4800+** but should work on any Docker host (Synology, QNAP, Linux server, etc.).

## Legal Notice

This project provides configuration files for **legal, open-source software** designed for managing personal media libraries. All included tools have legitimate purposes - see **[LEGAL.md](docs/LEGAL.md)** for details on intended use, user responsibilities, and disclaimer.

---

## Getting Started

**[Setup Guide](docs/SETUP.md)** - Complete step-by-step instructions for deployment.

<details>
<summary>Using Claude Code for guided setup (optional)</summary>

[Claude Code](https://docs.anthropic.com/en/docs/claude-code) can walk you through deployment step-by-step, executing commands and troubleshooting as you go.

**VS Code / Cursor:** Install the Claude extension, open this folder, and start a chat.

**Terminal:**
```bash
npm install -g @anthropic-ai/claude-code
cd arr-stack-ugreennas && claude
```

Ask Claude to help deploy the stack - it reads the [`.claude/instructions.md`](.claude/instructions.md) file automatically.

</details>

---

## Features

- **VPN-protected networking** via Gluetun + Surfshark for privacy
- **Automated SSL/TLS** certificates via Traefik + Cloudflare
- **Media library management** with Sonarr, Radarr, Prowlarr, Bazarr
- **Media streaming** with Jellyfin (or Plex - see below)
- **Request management** with Jellyseerr (or Overseerr for Plex)
- **Remote access** via WireGuard VPN
- **Ad-blocking DNS** with Pi-hole

## Stack Architecture

| Compose File | Layer | Services |
|--------------|-------|----------|
| `docker-compose.traefik.yml` | Infrastructure | Traefik (reverse proxy) |
| `docker-compose.cloudflared.yml` | External Access | Cloudflared (tunnel) - *Option A only* |
| `docker-compose.arr-stack.yml` | Media | Gluetun, qBittorrent, Sonarr, Radarr, Prowlarr, Jellyfin, Jellyseerr, Bazarr, FlareSolverr, Pi-hole, WireGuard |
| `docker-compose.utilities.yml` | Optional | deunhealth, Uptime Kuma, duc |

## Services Included

| Service | Description | Local Port | Domain URL |
|---------|-------------|------------|------------|
| **Traefik** | Reverse proxy with automatic SSL | 8080 | traefik.yourdomain.com |
| **Gluetun** | VPN gateway for network privacy | - | Internal |
| **qBittorrent** | BitTorrent client (VueTorrent UI included) | 8085 | qbit.yourdomain.com |
| **Sonarr** | TV show library management | 8989 | sonarr.yourdomain.com |
| **Radarr** | Movie library management | 7878 | radarr.yourdomain.com |
| **Prowlarr** | Search aggregator | 9696 | prowlarr.yourdomain.com |
| **Bazarr** | Subtitle management | 6767 | bazarr.yourdomain.com |
| **Jellyfin** | Media streaming server | 8096 | jellyfin.yourdomain.com |
| **Jellyseerr** | Media request system | 5055 | jellyseerr.yourdomain.com |
| **Pi-hole** | DNS + Ad-blocking | 53, 80 | pihole.yourdomain.com |
| **WireGuard** | VPN server for remote access | 51820/udp | wg.yourdomain.com |
| **FlareSolverr** | CAPTCHA solver | 8191 | Internal |

> **Don't need all these?** Remove any service by deleting its section from `docker-compose.arr-stack.yml`. Core dependencies: Gluetun (VPN gateway), Traefik (if using external access).
>
> **Prefer Plex?** See `docker-compose.plex-arr-stack.yml` for an untested Plex/Overseerr variant.

## Optional Utilities

Deploy with: `docker compose -f docker-compose.utilities.yml up -d`

| Service | Description | Local Port | Domain URL |
|---------|-------------|------------|------------|
| **deunhealth** | Auto-restart services when VPN recovers | - | Internal |
| **Uptime Kuma** | Service monitoring dashboard | 3001 | uptime.yourdomain.com |
| **duc** | Disk usage analyzer (treemap UI) | 8838 | duc.yourdomain.com |

## Deployment Options

### Option A: Remote Access (Recommended)

Access your services from anywhere - phone on mobile data, travelling, etc. Requires a cheap domain (~$10/year):
- **Remote access** from anywhere via Cloudflare Tunnel
- **SSL/HTTPS** with automatic certificates
- **Pretty URLs** like `jellyfin.yourdomain.com`
- **WireGuard VPN** for secure access to your home network

**Requirements:** Domain name, Cloudflare account (free), VPN subscription

> **Cloudflare:** This stack is configured for Cloudflare (DNS + Tunnel). Other DNS providers work but you'll need to modify `docker-compose.traefik.yml` and `traefik/traefik.yml`. See [Traefik ACME docs](https://doc.traefik.io/traefik/https/acme/).
>
> **VPN:** Configured for Surfshark but Gluetun supports 30+ providers (NordVPN, PIA, Mullvad, etc.). See [Gluetun providers](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers).

### Option B: Local Network Only (No Domain)

Skip the domain and access services directly via IP:port. All services work out of the box:
- `http://NAS_IP:8096` → Jellyfin
- `http://NAS_IP:5055` → Jellyseerr
- `http://NAS_IP:8989` → Sonarr
- `http://NAS_IP:7878` → Radarr
- `http://NAS_IP:9696` → Prowlarr
- `http://NAS_IP:8085` → qBittorrent
- `http://NAS_IP:6767` → Bazarr
- `http://NAS_IP:53` → Pi-hole DNS

**What works:** All media automation, VPN-protected downloads, Pi-hole DNS, local streaming

**What you lose:** Remote access, HTTPS, subdomain routing, WireGuard remote VPN

**To deploy local-only:**
1. Skip `docker-compose.traefik.yml` and `docker-compose.cloudflared.yml`
2. Deploy: `docker compose -f docker-compose.arr-stack.yml up -d`
3. Access via `http://NAS_IP:PORT`


## Security

**Many services default to NO authentication!** After deployment, enable authentication on:

| Service | Default Auth | Action Required |
|---------|--------------|-----------------|
| Bazarr | Disabled | Enable Forms auth, regenerate API key |
| Sonarr/Radarr/Prowlarr | Disabled for Local | Set to Forms + Enabled |
| qBittorrent | Bypass localhost | Disable bypass, change default password |

**Cloudflare Tunnel warning**: Tunnel traffic appears as localhost, bypassing "Disabled for Local Addresses" auth.

See [Security section](docs/SETUP.md#59-security-enable-authentication) in Setup Guide for details.

## License

Documentation, configuration files, and examples in this repository are licensed under [CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) (Attribution-NonCommercial). Individual software components (Sonarr, Radarr, Jellyfin, etc.) retain their own licenses.

## Acknowledgments

Forked from [TheRealCodeVoyage/arr-stack-setup-with-pihole](https://github.com/TheRealCodeVoyage/arr-stack-setup-with-pihole). Thanks to [@benjamin-awd](https://github.com/benjamin-awd) for VPN config improvements.
