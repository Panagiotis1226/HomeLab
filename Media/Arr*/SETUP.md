# Jellyfin + *Arr, TRaSH-Guides style — step by step

**Scope:** movies and TV shows only. Usenet first, torrents as fallback.
**Everything** (configs, downloads, library) lives inside this folder.
**Source of truth:** <https://trash-guides.info> — this doc is that guide, condensed
and pinned to your exact setup.

Files here:

| File | What it is |
|---|---|
| `docker-compose.yml` | The entire stack. One file, nine containers. |
| `.env` | Your settings. Edit this, not the compose file. |
| `config/recyclarr/recyclarr.yml` | TRaSH quality profiles → Radarr/Sonarr |
| `USENET-PROVIDERS.md` | What to actually buy |
| `data/` | Downloads + library (created in Step 2) |

---

## The one rule that makes all of this work

TRaSH's central point is **hardlinks and atomic moves**. When Radarr "moves" a
finished download into your library, you want that to be instant and to cost
**zero extra disk space** — the file exists once on disk with two names.

That only works if the download folder and the library folder are on the
**same filesystem, reached through the same mount**. So:

```
data/                          <- ONE mount. Radarr & Sonarr see all of it as /data
├── usenet/
│   ├── incomplete/            <- SAB working dir
│   └── complete/
│       ├── movies/            <- SAB category "movies"
│       └── tv/                <- SAB category "tv"
├── torrents/
│   ├── movies/                <- qBit category "movies"
│   └── tv/                    <- qBit category "tv"
└── media/
    ├── movies/                <- Radarr root folder + Jellyfin library
    └── tv/                    <- Sonarr root folder + Jellyfin library
```

The classic mistake is mounting `/downloads` and `/movies` as two separate
volumes. Then every import is a slow full copy, and seeding a torrent means
storing the file twice. Don't do that. The compose file here already does it
correctly — Radarr and Sonarr get `./data:/data` and nothing else.

---

# Part 1 — Get it running

### 1. Install Docker

- **Mac (now):** Docker Desktop for Apple Silicon. Give it at least 4 CPUs and
  8 GB RAM in Settings → Resources.
- **Linux (later):** Docker Engine + the compose plugin.

Verify:

```bash
docker --version
docker compose version
```

### 2. Create the folder structure

From inside this folder:

```bash
mkdir -p data/media/{movies,tv} \
         data/usenet/{incomplete,complete/{movies,tv}} \
         data/torrents/{movies,tv} \
         config/{jellyfin,sonarr,radarr,prowlarr,sabnzbd,qbittorrent,jellyseerr,recyclarr}
```

On the Linux box, also fix ownership (TRaSH's recommendation):

```bash
sudo chown -R $USER:$USER data
sudo chmod -R a=,a+rX,u+w,g+w data
```

On macOS, skip that — Docker Desktop handles ownership for you.

### 3. Edit `.env`

Open `.env` and set:

- `TZ` — already `America/Toronto`.
- `PUID` / `PGID` — leave at `1000` on the Mac. On Linux, run `id -u` and
  `id -g` and use the real values.
- Leave `RADARR_API_KEY` and `SONARR_API_KEY` blank for now — Step 12.

### 4. Start the stack

```bash
docker compose up -d
docker compose ps
```

First pull is a few GB. Once it settles you have:

| Service | URL | Job |
|---|---|---|
| Jellyfin | <http://localhost:8096> | Watch things |
| Seerr (was Jellyseerr) | <http://localhost:5055> | Request things |
| Radarr | <http://localhost:7878> | Movies |
| Sonarr | <http://localhost:8989> | TV |
| Prowlarr | <http://localhost:9696> | Indexers |
| SABnzbd | <http://localhost:8080> | Usenet downloads |
| qBittorrent | <http://localhost:8081> | Torrent downloads |
| FlareSolverr | <http://localhost:8191> | Cloudflare bypass helper |

> **Buy your Usenet access before Step 5.** See `USENET-PROVIDERS.md`. You need
> one provider (for the actual files) and at least two indexers (to find them).

---

# Part 2 — Downloaders

### 5. SABnzbd (Usenet)

1. Open <http://localhost:8080>. The wizard runs on first launch.
2. **Server:** enter your provider's host, port `563`, tick **SSL**, your
   username/password, and set connections to whatever your plan allows
   (usually 20–50). Hit **Test Server** — you want a green result.
3. Finish the wizard. Then **⚙ → Folders**:
   - Temporary Download Folder: `/data/usenet/incomplete`
   - Completed Download Folder: `/data/usenet/complete`
4. **⚙ → Categories** — create exactly these two, because Radarr and Sonarr
   will reference them by name:

   | Category | Folder/Path |
   |---|---|
   | `movies` | `movies` |
   | `tv` | `tv` |

   Leave Priority `Default` and Processing `+Delete`.
5. **⚙ → Switches:** turn **off** "Post-Process Only Verified Jobs" is *not*
   needed — instead make sure **"Direct Unpack"** is on (faster) and
   **Pause Downloading During Post-Processing** is off.
6. **⚙ → General:** copy your **API Key**. You need it in Step 8.

> If you buy block accounts later (recommended), add them as additional servers
> in SABnzbd with **Priority 1 / 2**, leaving your unlimited provider at
> **Priority 0**. SAB only touches the blocks when the primary is missing
> articles, so a 1 TB block lasts years.

### 6. qBittorrent (torrent fallback)

1. Get the temporary admin password:

   ```bash
   docker compose logs qbittorrent | grep -i "temporary password"
   ```

2. Open <http://localhost:8081>, log in as `admin` with that password.
3. **Tools → Options → Web UI:** set a real username and password.
4. **Downloads:**
   - Default Save Path: `/data/torrents`
   - Uncheck "Keep incomplete torrents in" (keeps hardlinks working)
   - **Uncheck** "Delete .torrent files afterwards"
5. **Categories** (right-click the Categories panel → Add category):

   | Category | Save path |
   |---|---|
   | `movies` | `/data/torrents/movies` |
   | `tv` | `/data/torrents/tv` |

6. **BitTorrent tab:** leave seeding limits alone for now. If you're using
   private trackers, respect their ratio rules; on public trackers, setting
   "Seeding time limit" to ~10080 minutes (7 days) is polite.

---

# Part 3 — Indexers

### 7. Prowlarr

1. Open <http://localhost:9696>. Set **Settings → General → Authentication** to
   *Forms* and create a login.
2. **Settings → Indexers → Add Indexer Proxy → FlareSolverr**
   - Tags: `flaresolverr`
   - Host: `http://flaresolverr:8191/`
   (Only needed for a couple of public torrent sites. Harmless otherwise.)
3. **Indexers → Add Indexer.** Add each Usenet indexer you bought:
   search its name, paste the **API key** from that site's profile page.
   Then add 2–3 public torrent indexers as fallback, e.g. **TorrentLeech**,
   **1337x**, **TheRARBG**, **Torrentio**-style trackers, or whatever private
   trackers you have. Tag the Cloudflare-protected ones `flaresolverr`.
4. **Settings → Apps → Add → Radarr**
   - Prowlarr Server: `http://prowlarr:9696`
   - Radarr Server: `http://radarr:7878`
   - API Key: from Radarr → Settings → General
5. Repeat for **Sonarr** with `http://sonarr:8989`.

Prowlarr now pushes every indexer into both apps automatically. You never add
an indexer inside Radarr or Sonarr again.

---

# Part 4 — Radarr (movies)

### 8. Download clients

**Settings → Download Clients → + → SABnzbd**

| Field | Value |
|---|---|
| Name | SABnzbd |
| Host | `sabnzbd` |
| Port | `8080` |
| API Key | from Step 5.6 |
| Category | `movies` |
| Client Priority | **1** |

**+ → qBittorrent**

| Field | Value |
|---|---|
| Name | qBittorrent |
| Host | `qbittorrent` |
| Port | `8081` |
| Username / Password | from Step 6.3 |
| Category | `movies` |
| Client Priority | **50** |

Then **Settings → Download Clients → Completed Download Handling**: Enable is on,
Remove is on.

That priority gap is what "Usenet first, torrents only if needed" actually
means. Also go to **Settings → Profiles → Delay Profiles**, edit the default,
and set **Usenet delay: 0 min**, **Torrent delay: 30 min** — Radarr will wait
half an hour for a Usenet release before it settles for a torrent.

> **Remote path mappings: not needed.** Every container sees the same `/data`
> paths, which is the whole point of the layout. If you ever see
> "file not found" on import, that's the thing that's broken.

### 9. Media management

**Settings → Media Management**, tick *Show Advanced* (top right).

- **Rename Movies:** ✅
- **Replace Illegal Characters:** ✅
- **Standard Movie Format:**

  ```
  {Movie CleanTitle} {(Release Year)} {tmdb-{TmdbId}} - {edition-{Edition Tags}} {[MediaInfo 3D]}{[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
  ```

- **Movie Folder Format** (the Jellyfin variant — the tmdbid makes Jellyfin
  match every title correctly, including remakes and foreign titles):

  ```
  {Movie CleanTitle} ({Release Year}) [tmdbid-{TmdbId}]
  ```

- **Use Hardlinks instead of Copy:** ✅ ← the important one
- **Import Extra Files:** ✅, extensions `srt,sub,idx`
- **Unmonitor Deleted Movies:** ✅
- **Propers and Repacks:** *Do not Prefer* (Recyclarr's custom formats handle
  this properly — this setting fights them)
- **Analyse video files:** ✅

**Root Folder:** add `/data/media/movies`.

---

# Part 5 — Sonarr (TV)

### 10. Download clients

Identical to Step 8, except **Category = `tv`** on both clients.
Same priorities (SABnzbd 1, qBittorrent 50), same delay profile.

### 11. Media management

**Settings → Media Management**, *Show Advanced* on.

- **Rename Episodes:** ✅
- **Standard Episode Format:**

  ```
  {Series CleanTitleWithoutYear} {(Series Year)} - S{season:00}E{episode:00} - {Episode CleanTitle:90} {[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
  ```

- **Daily Episode Format:**

  ```
  {Series CleanTitleWithoutYear} {(Series Year)} - {Air-Date} - {Episode CleanTitle:90} {[Custom Formats]}{[Quality Full]}{[Mediainfo AudioCodec}{ Mediainfo AudioChannels]}{[MediaInfo VideoDynamicRangeType]}{[Mediainfo VideoCodec]}{-Release Group}
  ```

- **Series Folder Format** (Jellyfin variant):

  ```
  {Series CleanTitleWithoutYear} {(Series Year)} [tvdbid-{TvdbId}]
  ```

- **Season Folder Format:** `Season {season:00}`
- **Multi Episode Style:** `Prefixed Range`
- **Use Hardlinks instead of Copy:** ✅
- **Import Extra Files:** ✅ — `srt,sub,idx`
- **Propers and Repacks:** *Do not Prefer*

**Root Folder:** add `/data/media/tv`.

---

# Part 6 — Recyclarr (this is the actual TRaSH part)

Everything above is plumbing. Recyclarr is what imports TRaSH's opinion about
**which release is better than which** — hundreds of custom formats scoring
release groups, encoders, audio formats, upscale detection, and known-bad rips.
Doing it by hand is a full afternoon and it goes stale.

### 12. Wire it up

1. Radarr → **Settings → General → API Key**. Copy it.
2. Sonarr → **Settings → General → API Key**. Copy it.
3. Paste both into `.env`:

   ```
   RADARR_API_KEY=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   SONARR_API_KEY=yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
   ```

4. Recreate the container and run a sync immediately:

   ```bash
   docker compose up -d recyclarr
   docker compose exec recyclarr recyclarr sync
   ```

   You should see it create quality definitions, a quality profile, and a long
   list of custom formats in each app. If you get `401`, the API key is wrong.

5. Radarr → **Settings → Profiles**: a profile named **HD Bluray + WEB** now
   exists. Set it as the default in **Settings → Media Management → Root
   Folders**, or pick it when adding a movie.
6. Sonarr → same, profile **WEB-1080p**.

From now on Recyclarr re-syncs nightly at 04:00 (`RECYCLARR_CRON` in `.env`),
so you inherit TRaSH's updates automatically.

> **Recyclarr v8 note:** the config now lives in
> `config/recyclarr/configs/hd-bluray-web.yml` and `configs/web-1080p.yml`
> (v8 replaced the old `include:` template system; the previous
> `recyclarr.yml` is kept as `recyclarr.yml.v7.bak`). To go 4K later,
> generate a new template: `docker compose exec recyclarr recyclarr config
> create -t uhd-bluray-web` (Radarr) / `-t web-2160p` (Sonarr).

**Why 1080p:** your target host is an i7-12xxx with only the integrated UHD
graphics. It will chew through 1080p, including transcodes. 4K remuxes are
40–80 GB each and any client that can't direct-play HDR forces a tone-mapping
transcode that iGPU will struggle with. To switch later, edit
`config/recyclarr/recyclarr.yml` — the 4K template names are in the comments.

---

# Part 7 — Jellyfin

### 13. First run

1. Open <http://localhost:8096>, pick a language, create your admin user.
2. **Add Media Library → Movies**
   - Folder: `/data/media/movies`
   - Metadata downloaders: TheMovieDb ✅
   - *Preferred download language*: your choice
3. **Add Media Library → Shows**
   - Folder: `/data/media/tv`
   - Metadata downloaders: TheTVDB ✅, TheMovieDb ✅
4. Finish the wizard. Set remote access as you prefer.

### 14. Transcoding

**Dashboard → Playback → Transcoding.**

- **On the Mac, right now:** leave Hardware Acceleration set to **None**.
  Docker Desktop does not pass the Apple GPU into the Linux VM. There is no
  workaround. Software transcoding on an M4 is still quite fast.
- **On the i7-12xxx later:** set Hardware Acceleration to **Intel QuickSync
  (QSV)**, enable **Low-Power** encoding, and tick H264/HEVC/VP9/AV1 decoding.
  You must first uncomment the `devices:` and `group_add:` blocks under
  `jellyfin` in `docker-compose.yml` and put your host's real `render`/`video`
  group ids there:

  ```bash
  getent group render | cut -d: -f3
  getent group video  | cut -d: -f3
  ```

  Then `docker compose up -d jellyfin` and confirm `/dev/dri/renderD128` shows
  up inside the container:

  ```bash
  docker compose exec jellyfin ls -l /dev/dri
  ```

  Also enable **Allow encoding in HEVC format** and set **Enable Tone mapping**
  only if you go 4K HDR later.

---

# Part 8 — Seerr, formerly Jellyseerr (requests)

### 15. Connect everything

1. Open <http://localhost:5055>. Choose **Sign in with Jellyfin**.
   - Jellyfin URL: `http://jellyfin:8096`
   - Log in with the admin account from Step 13.
2. Sync libraries: select **Movies** and **Shows**, run **Start Scan**.
3. **Settings → Services → Add Radarr Server**

   | Field | Value |
   |---|---|
   | Server Name | Radarr |
   | Hostname | `radarr` |
   | Port | `7878` |
   | API Key | Radarr's |
   | Quality Profile | **HD Bluray + WEB** |
   | Root Folder | `/data/media/movies` |
   | Default Server | ✅ |
   | Enable Automatic Search | ✅ |

4. **Add Sonarr Server**

   | Field | Value |
   |---|---|
   | Hostname | `sonarr` |
   | Port | `8989` |
   | API Key | Sonarr's |
   | Quality Profile | **WEB-1080p** |
   | Root Folder | `/data/media/tv` |
   | Default Server | ✅ |
   | Enable Automatic Search | ✅ |

5. **Settings → Users:** import your Jellyfin users and set their request
   quotas. Requests from non-admins land in an approval queue by default.

**How a request flows:** user searches in Jellyseerr → Jellyseerr creates the
movie/series in Radarr/Sonarr and triggers a search → Prowlarr's indexers get
queried → best release by TRaSH score wins → SABnzbd grabs it (or qBittorrent
after the 30-minute delay if there's no Usenet release) → Radarr/Sonarr hardlink
it into `/data/media` with the correct name → Jellyfin picks it up.

---

# Part 9 — Verify it works

### 16. End-to-end test

1. In Radarr, add a well-known older movie and hit **Search**.
2. **Activity → Queue** should show it in SABnzbd.
3. When it finishes, check the import worked *and used a hardlink*:

   ```bash
   docker compose exec radarr sh -c 'ls -li /data/media/movies/*/*'
   ```

   Look at the **second column** — that's the link count. It should be **2**
   (or more) while the download is still in `/data/usenet/complete`. If it's
   `1`, hardlinking silently failed and you're burning double the disk.

4. Confirm the same for a TV episode via Sonarr.
5. In Jellyfin, **Dashboard → Scheduled Tasks → Scan Media Library**. The title
   should appear with full artwork.
6. Request something through Jellyseerr and watch it complete without you
   touching Radarr.

### 17. Routine checks

```bash
docker compose ps                 # everything "running"
docker compose logs -f radarr     # follow one app
docker compose pull && docker compose up -d   # update everything
docker compose exec recyclarr recyclarr sync  # force a TRaSH re-sync
```

Back up `config/` — that's your entire setup. `data/` is replaceable.

---

# Part 10 — Moving to the Linux box

> The i7 box runs **Proxmox VE 9** — see **[PROXMOX.md](PROXMOX.md)** first
> for the VM-vs-LXC decision, iGPU passthrough, and the two migration traps
> (the missing `.env`, and SABnzbd's named volume). Then come back here.

When you migrate to the i7-12xxx:

1. Copy this whole folder over (`config/`, `docker-compose.yml`, `.env`).
   Keep `data/` on the big drive; if it lives elsewhere, change **only** the
   `./data` prefix in the compose file — never split it into multiple mounts.
2. Set real `PUID`/`PGID` in `.env` from `id -u` / `id -g`.
3. `sudo chown -R $USER:$USER data && sudo chmod -R a=,a+rX,u+w,g+w data`
4. Uncomment the `devices:` / `group_add:` block on `jellyfin` and set the
   right group ids, then enable QSV in Jellyfin (Step 14).
5. Set `JELLYFIN_URL` and `HOST_LAN_IP` in `.env` to the server's IP.
6. Use **ext4, XFS, or Btrfs** for `data`. **Not exFAT, not NTFS** — hardlinks
   don't work there and the whole design collapses.
7. `docker compose up -d`.

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| "Host not whitelisted" from Radarr → SAB | SAB's `host_whitelist` setting (Settings → Special) must include `sabnzbd`. It's already set; add your LAN IP there too if you access SAB from another machine. |
| Imports are slow / disk fills twice | Hardlinks failed. Check the link count (Step 16.3), check "Use Hardlinks" is on, check `data` isn't exFAT/NTFS or split across mounts. |
| Radarr: "file not found" on import | Paths differ between containers. Radarr/Sonarr must mount `./data:/data`, not `/downloads` + `/movies`. |
| Recyclarr `401 Unauthorized` | Wrong API key in `.env`. Re-copy, then `docker compose up -d recyclarr`. |
| Permission denied writing to library | `PUID`/`PGID` don't match the owner of `data/`. |
| Jellyfin can't see new files | Library folder is `/data/media/movies`, not `/movies`. Run Scan Media Library. |
| Indexer returns nothing | Test it in Prowlarr first (Settings → Indexers → Test All). Cloudflare-blocked ones need the `flaresolverr` tag. |
| qBittorrent WebUI won't accept `admin` | Password rotates each restart until you set one. Grep the logs again. |

---

## Reference

- TRaSH Guides — <https://trash-guides.info>
- Folder structure — <https://trash-guides.info/File-and-Folder-Structure/>
- Hardlinks explained — <https://trash-guides.info/File-and-Folder-Structure/Hardlinks-and-Instant-Moves/>
- Recyclarr — <https://recyclarr.dev>
- Jellyfin HW acceleration — <https://jellyfin.org/docs/general/administration/hardware-acceleration/>
