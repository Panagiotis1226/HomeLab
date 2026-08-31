# Running the stack on Proxmox VE 9 (i7-12th gen mini-PC)

Target: Intel i7-12xxx (UHD 770 iGPU — excellent Quick Sync), no dedicated
GPU, 32 GB RAM, Proxmox VE 9. The stack runs unchanged; the only decisions
are *where* Docker lives and how Jellyfin gets the iGPU.

## VM or LXC?

**Use an LXC container.** Here's the trade-off:

| | LXC (recommended) | VM |
|---|---|---|
| iGPU / Quick Sync | Trivial — two clicks in the PVE GUI, iGPU stays shared with the host | Full PCIe passthrough of the iGPU; notoriously flaky on Alder Lake (host loses console, BAR errors). SR-IOV vGPU exists but needs a third-party DKMS module |
| Overhead | Near zero; RAM and CPU shared dynamically | Fixed RAM allocation, virtualization overhead (small) |
| Docker support | Works fine (nesting=1), but Proxmox *officially* prefers Docker in a VM; a major PVE upgrade can occasionally need a re-tweak | Officially blessed, most isolated |
| Setup effort | ~15 min | ~15 min *without* transcoding; iGPU passthrough can eat an evening |

The whole reason this box beats the Mac is the UHD 770's Quick Sync
(~10 simultaneous 1080p transcodes). In an LXC that's nearly free; in a VM
it's the hardest part of the build. Single-purpose homelab box → LXC.

**Never install Docker on the PVE host itself.**

> **ZFS caveat:** if you installed PVE 9 with ZFS as the root filesystem,
> Docker's overlay2 storage driver won't work inside an unprivileged LXC on a
> ZFS-backed rootfs. Either give the container a dedicated ext4 volume for
> `/var/lib/docker`, or use a VM instead. On the default LVM-thin/ext4
> install this doesn't apply.

## Part A — Create the LXC (~5 min)

1. PVE GUI → your node → **local** storage → *CT Templates* → **Templates** →
   download **debian-13-standard** (or 12).
2. **Create CT**:
   - *General*: hostname `arrstack`, **Unprivileged container ✓**
   - *Template*: the Debian template
   - *Disks*: 32 GB rootfs is plenty for the apps (config, databases,
     Docker images). The media itself goes on the **external SSD** — see
     **Part D** below; don't allocate internal space for it.
     **ext4 or XFS only** — the hardlink rule from the README applies to
     whatever filesystem ultimately holds `data/`.
   - *CPU*: 6–8 cores. *Memory*: 8192 MB (the stack idles around 2–3 GB;
     8 GB leaves headroom for parallel unpacking + transcoding).
   - *Network*: DHCP or a static LAN IP — you'll want a stable IP either way.
3. Before first boot: **Options → Features → Nesting ✓** (required for
   Docker). Keyctl is not needed for this stack.

## Part B — Give it the iGPU (~2 min)

1. On the PVE **host** shell, confirm the iGPU is there:
   `ls /dev/dri` → you should see `card0` (or `card1`) and `renderD128`.
   PVE 9's kernel already has the i915 driver; nothing to install.
2. Container → **Resources → Add → Device Passthrough**:
   - `/dev/dri/renderD128` — set **Access mode `0666`**
   - `/dev/dri/card0` (or `card1`, whichever exists) — mode `0666`
3. Start the container. Inside it, `ls -l /dev/dri` should show both nodes.

(Mode `0666` sidesteps the unprivileged-container GID-mapping dance. If you'd
rather be strict, set the device's GID to the container's `render` group
instead and put that GID in the compose `group_add`.)

## Part C — Docker + the stack (~10 min)

Inside the container (`pct enter <ctid>` or the console):

```bash
apt update && apt install -y curl rsync
curl -fsSL https://get.docker.com | sh
```

Then migrate, following SETUP.md **Part 10** with these Proxmox-specific
notes layered on top:

1. **Copy the folder** from the Mac (rsync/scp) to e.g. `/opt/arrstack`.
   ⚠️ **`.env` is not in this repo folder** — it lives only on the machine
   where the stack last ran. It holds every API key plus the qBittorrent
   login. Copy it explicitly (dotfiles are easy to miss):
   `rsync -av --include='.env' ...` or `scp .env root@<ct-ip>:/opt/arrstack/`
2. **SABnzbd's config is NOT in `config/`** — it's in the Docker named
   volume `arrstack_sabnzbd_config`. On the Mac, export it:
   ```bash
   docker run --rm -v arrstack_sabnzbd_config:/c -v $PWD:/out alpine tar czf /out/sabnzbd-config.tgz -C /c .
   ```
   On the new box, after the first `docker compose up -d` creates the empty
   volume, restore it:
   ```bash
   docker compose stop sabnzbd
   docker run --rm -v arrstack_sabnzbd_config:/c -v $PWD:/out alpine sh -c "cd /c && tar xzf /out/sabnzbd-config.tgz"
   docker compose start sabnzbd
   ```
   (The named volume was a macOS bind-mount workaround; it works fine as-is
   on Linux, so keep it — no reason to restructure.)
3. **PUID/PGID**: inside the LXC as root, either create a user
   (`useradd -u 1000 media`) and keep `1000`, or set them to `id -u`/`id -g`
   of whoever owns `/opt/arrstack`. Then the chown from SETUP Part 10 step 3.
4. **Jellyfin QSV**: uncomment the `devices:`/`group_add:` block on
   `jellyfin` in the compose file. With the `0666` trick from Part B the
   `group_add` lines are optional; if you used GIDs instead, set them to
   `stat -c '%g' /dev/dri/renderD128` (run *inside* the LXC).
5. **`.env` addresses**: set `JELLYFIN_URL` (and `HOST_LAN_IP` if present) to
   the **container's** LAN IP, not the Proxmox host's.
6. `docker compose up -d`, then in Jellyfin: Dashboard → Playback →
   Transcoding → **Intel QuickSync (QSV)**. Verify with
   `docker compose exec jellyfin ls -l /dev/dri`.
7. **SAB whitelist**: you'll now hit SAB by LAN IP, so add that IP (or the
   hostname you use) in SAB → Settings → Special → `host_whitelist`.
   Same for any browser access — everything that was `localhost:<port>` in
   the docs becomes `<ct-ip>:<port>`.

## Part D — External SSD for `data/` (~10 min)

The plan: format the SSD ext4, mount it on the **PVE host**, then bind-mount
it into the LXC *directly at the stack's `data/` path* — that way the
compose file needs **zero edits** and the hardlink rule survives, because
inside the container `data/` is still one filesystem, one mount.

All of this happens on the **PVE host** shell, container stopped.

### D.1 Format it (once — this wipes the SSD)

1. Plug it in and find it: `lsblk -o NAME,SIZE,MODEL,FSTYPE` — it'll show up
   as `sda` or `sdb` with the SSD's model name. **Triple-check the name**;
   the next commands erase whatever disk you point them at.
2. Partition + format ext4 (not exFAT/NTFS — no hardlinks, and most
   store-bought SSDs come preformatted exFAT):
   ```bash
   sgdisk --zap-all /dev/sdX
   sgdisk -n1:0:0 -t1:8300 /dev/sdX
   mkfs.ext4 -L arrdata /dev/sdX1
   ```

### D.2 Mount it on the host, by UUID, survive-reboot-and-unplug

```bash
mkdir -p /mnt/arrdata
blkid /dev/sdX1        # copy the UUID
```

Add to `/etc/fstab` (one line, with the UUID you just copied):

```
UUID=xxxx-xxxx  /mnt/arrdata  ext4  defaults,nofail,x-systemd.device-timeout=10s  0  2
```

`nofail` matters: without it, booting with the SSD unplugged drops the host
into an emergency shell. Then `systemctl daemon-reload && mount -a` and
confirm with `df -h /mnt/arrdata`.

### D.3 Ownership for the unprivileged LXC

An unprivileged container shifts UIDs by 100000, so container-UID 1000
(your `PUID`) is host-UID **101000**:

```bash
chown -R 101000:101000 /mnt/arrdata
```

(If you ever change `PUID`/`PGID` in `.env`, it's 100000 + that number.)

### D.4 Bind-mount into the container at the stack's data path

Bind mounts can't be added in the GUI — use `pct` on the host (replace
`<ctid>`, and the target path with wherever you put the stack; Part C used
`/opt/arrstack`):

```bash
pct set <ctid> -mp0 /mnt/arrdata,mp=/opt/arrstack/data
```

Start the container. Inside it, `df -h /opt/arrstack/data` should show the
SSD, and `./data` in the compose file now *is* the SSD — no compose changes,
and the folder tree from the README's quick-start `mkdir` lands on it.

### D.5 Caveats

- **Backups**: bind mounts are automatically excluded from vzdump — exactly
  what you want (media is replaceable, and you don't want terabytes in every
  backup). Trade-off: PVE can no longer *snapshot* this container; scheduled
  vzdump backups still work fine.
- **Never unplug while the stack is running** — SAB/qBittorrent write to it
  constantly. `docker compose down` first, then `umount /mnt/arrdata` on the
  host.
- USB enclosure sleep/spin-down can stall imports; if downloads mysteriously
  pause, disable the enclosure's power saving (`usbcore.autosuspend=-1` on
  the host kernel cmdline is the blunt fix).
- If the SSD is missing at container start, the LXC will refuse to start
  (the `mp0` target can't be satisfied) — that's a feature: better than the
  stack writing into an empty directory on the rootfs.

## Backups, the Proxmox way

`vzdump` (Datacenter → Backup) backs up the whole container — config,
volumes, everything — which supersedes hand-rolled config backups. The
external-SSD bind mount from Part D is excluded automatically, so you're
never snapshotting terabytes of replaceable media; a nightly backup of this
container stays small (it's just the app configs and Docker images).

## If you'd rather use a VM anyway

Debian 13 VM, CPU type `host`, 8 GB RAM, same Docker install, same migration
steps — everything works except hardware transcoding. Accept software
transcoding (the i7 handles a couple of 1080p streams, but 4K tone-mapping
will hurt), or attempt iGPU PCIe passthrough / i915 SR-IOV at your own
peril — on 12th-gen those are the two hardest tricks in the Proxmox book,
which is exactly why the LXC route is recommended.
