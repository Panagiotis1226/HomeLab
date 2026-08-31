# Setup status — what's done, what's left

*Automated setup ran 2026-08-19. Everything below was verified working.*

## Already configured ✅

| Step | Status |
|---|---|
| Folder structure (`data/`, hardlink-safe single mount) | ✅ |
| All 9 containers running | ✅ |
| **SABnzbd** — folders (`/data/usenet/incomplete` + `complete`), categories `movies`/`tv` (+Delete), Direct Unpack on, NewsDemon server entry created (SSL, port 563, 50 connections) | ✅ except credentials |
| **qBittorrent** — WebUI login set (see `.env` → `QBIT_USERNAME`/`QBIT_PASSWORD`), save path `/data/torrents`, no incomplete dir, categories `movies`/`tv` | ✅ |
| **Radarr + Sonarr** — SABnzbd (priority 1) + qBittorrent (priority 50) clients, delay profile usenet 0 min / torrent 30 min, TRaSH naming formats, hardlinks on, root folders | ✅ |
| **Prowlarr** — FlareSolverr proxy, Radarr + Sonarr apps (full sync), altHUB indexer added | ✅ except API key |
| **Recyclarr** — TRaSH sync done: Radarr profile **HD Bluray + WEB** (40 custom formats), Sonarr profile **WEB-1080p** (37 custom formats), nightly re-sync at 04:00 | ✅ |

API keys for all services are in `.env`. SABnzbd's own API key lives inside its
config volume (visible in SAB → Settings → General).

## You do these (5–10 minutes) 🔧

1. **NewsDemon credentials** → <http://localhost:8080> → ⚙ → Servers →
   `news.newsdemon.com`: replace `YOUR_NEWSDEMON_USERNAME` / `YOUR_NEWSDEMON_PASSWORD`
   with your real login, tick **Enable**, hit **Test Server** (want green), Save.

2. **altHUB API key** → get it from your altHUB profile page (althub.co.za) →
   <http://localhost:9696> → Indexers → altHUB → paste the key, toggle
   **Enable**, hit **Test**, Save. Prowlarr then pushes it into Radarr and
   Sonarr automatically.
   *Note: the free altHUB tier has a very small daily API limit — Radarr/Sonarr
   RSS polling will burn through it fast. Upgrading to premium (as planned)
   fixes that.*

3. **First visit to Radarr / Sonarr / Prowlarr** — each will ask you to pick an
   authentication method and create a login. Choose *Forms* and set a
   username/password of your choice.

4. **Jellyfin wizard** → <http://localhost:8096> — create your admin user, add
   libraries: Movies → `/data/media/movies`, Shows → `/data/media/tv`
   (SETUP.md Step 13). Leave hardware acceleration at **None** on the Mac.

5. **Seerr wizard** (the app formerly deployed as Jellyseerr — done ✅) → <http://localhost:5055> — sign in with Jellyfin
   (URL `http://jellyfin:8096`), then add Radarr/Sonarr per SETUP.md Step 15.
   Use quality profile **HD Bluray + WEB** (Radarr) and **WEB-1080p** (Sonarr).

6. **Optional:** add 2–3 torrent indexers in Prowlarr as fallback
   (SETUP.md Step 7.3), and when you're happy, buy the NewsDemon 1 TB block
   and a second indexer per `USENET-PROVIDERS.md`.

Then run the end-to-end test in SETUP.md **Step 16** (add a movie, watch it
land in SABnzbd, verify the hardlink count is 2).

## Deviations from the original files 📝

- **SABnzbd config** is in the named volume `arrstack_sabnzbd_config`, not
  `./config/sabnzbd` — its sqlite history DB kept failing ("readonly database")
  on the macOS bind mount. Backup one-liner is commented in `docker-compose.yml`.
- **Recyclarr pinned to v8** and its config regenerated in the v8 format at
  `config/recyclarr/configs/` (ghcr has no `:latest` tag, and v8 removed the
  old `include:` templates the original `recyclarr.yml` used).
- **SAB host whitelist** is set inside SAB's config (the compose env var the
  file used previously wasn't a real SABnzbd setting).
