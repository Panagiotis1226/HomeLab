# Arr — Jellyfin media stack (TRaSH Guides)

Movies + TV. Usenet primary, torrents fallback. One compose file.

## Quick start

```bash
mkdir -p data/media/{movies,tv} \
         data/usenet/{incomplete,complete/{movies,tv}} \
         data/torrents/{movies,tv} \
         config/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,qbittorrent,jellyseerr,recyclarr}

# edit .env first
docker compose up -d
```

Then follow **[SETUP.md](SETUP.md)** from Step 5 — it's numbered end to end.

## What's here

- `docker-compose.yml` — Jellyfin, Sonarr, Radarr, Prowlarr, SABnzbd,
  qBittorrent, Seerr (successor to Jellyseerr), Recyclarr, FlareSolverr
- `.env` — all settings live here (incl. generated qBittorrent WebUI login)
- `config/recyclarr/configs/*.yml` — TRaSH quality profiles (Recyclarr v8
  format; the old single `recyclarr.yml` is kept as `recyclarr.yml.v7.bak`)
- `SETUP.md` — the numbered guide
- `PROXMOX.md` — deploying on the Proxmox VE 9 mini-PC (LXC vs VM, Quick Sync)
- `NEXT-STEPS.md` — what's already configured and the few manual steps left
- `USENET-PROVIDERS.md` — which providers and indexers to buy

> **Note:** SABnzbd's `/config` lives in a Docker named volume
> (`arrstack_sabnzbd_config`), not in `./config/sabnzbd` — its sqlite history
> DB is unreliable on macOS bind mounts. Backup command is in the compose file.

## Ports

| Service | Port |
|---|---|
| Jellyfin | 8096 |
| Seerr (was Jellyseerr) | 5055 |
| Radarr | 7878 |
| Sonarr | 8989 |
| Prowlarr | 9696 |
| SABnzbd | 8080 |
| qBittorrent | 8081 |
| FlareSolverr | 8191 |

## The rule

`./data` is mounted as **one** volume into Radarr and Sonarr. Never split it
into `/downloads` + `/movies` — that breaks hardlinks and every import becomes
a full disk copy.
