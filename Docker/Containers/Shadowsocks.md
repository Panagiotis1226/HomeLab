## Shadowsocks Docker Service

This document explains what **Shadowsocks** is, how the provided Docker service works, and how it can help when your ISP blocks IPTV streams and other types of traffic.

---

### What is Shadowsocks?

Shadowsocks is a **secure, SOCKS5-based proxy** designed to:

- **Encrypt traffic** between your device and a remote server.
- **Obfuscate traffic patterns**, making it harder for ISPs and firewalls to detect and block.
- **Bypass censorship and throttling**, especially for specific services (IPTV, VoIP, streaming, etc.).

Unlike a traditional VPN, Shadowsocks:

- Typically acts as a **per-application proxy** (you point apps or system proxy to it).
- Is lightweight and optimized for **performance and low latency**.
- Is harder to fingerprint by Deep Packet Inspection (DPI) systems used by some ISPs.

---

### How this Docker service works

The following service definition runs a **Shadowsocks server** inside a container:

```yaml
services:
  shadowsocks:
    image: shadowsocks/shadowsocks-libev
    container_name: shadowsocks
    restart: unless-stopped
    ports:
      - "443:8388/tcp"
      - "443:8388/udp"
    command: >
      ss-server
      -s 0.0.0.0
      -p 8388
      -k (PASSWORD)
      -m chacha20-ietf-poly1305
      -u
      --fast-open
```

- **image**: Uses the official `shadowsocks/shadowsocks-libev` image.
- **ports**:
  - Exposes container port `8388` as **TCP/UDP 443** on the host.
  - Port `443` is commonly used for HTTPS, which:
    - Looks like regular encrypted web traffic.
    - Is less likely to be blocked or throttled by ISPs.
- **command**:
  - `ss-server`: runs the Shadowsocks server.
  - `-s 0.0.0.0`: listen on all interfaces.
  - `-p 8388`: server listens on port 8388 inside the container.
  - `-k (PASSWORD)`: the **password** clients must use to connect (replace this with a strong secret).
  - `-m chacha20-ietf-poly1305`: secure, modern AEAD cipher for encryption.
  - `-u`: enables UDP relay (important for some streaming/IPTV protocols).
  - `--fast-open`: enables TCP Fast Open for lower latency (requires OS support).

To start the service (from the directory containing your `docker-compose.yaml`):

```bash
docker compose up -d shadowsocks
```

---

### How Shadowsocks helps when ISP blocks IPTV and other traffic

Many ISPs use techniques like:

- **DNS tampering or blocking**: returning wrong IPs or no response for certain IPTV domains.
- **IP/port blocking**: outright blocking IP addresses or ports used by IPTV providers.
- **Deep Packet Inspection (DPI)** and **traffic shaping**: detecting IPTV/streaming protocols and throttling or dropping them.

By using Shadowsocks:

- **Your traffic is encrypted** between your client and the Shadowsocks server.
  - The ISP sees only encrypted connections to your server on port 443.
  - They cannot easily see that you are accessing IPTV services behind it.
- **IPTV traffic is tunneled** through the Shadowsocks connection:
  - Your IPTV app or player connects to Shadowsocks (SOCKS5 proxy).
  - Shadowsocks then connects to the actual IPTV servers on your behalf.
  - From the ISP’s perspective, it all looks like regular encrypted HTTPS-style traffic.
- **UDP support** (`-u`) allows protocols that rely on UDP (common in streaming and IPTV) to work through the tunnel.

This can:

- **Bypass blocks** on IPTV domains, IPs, or ports.
- **Avoid DPI-based throttling** or shaping that specifically targets streaming/ IPTV.
- Help maintain **stable and higher-quality streams**, assuming your upstream server is in an unfiltered location.

> Note: You still need a **remote server/VPS** in a region where IPTV is not blocked. This Docker service should be running on that remote machine, not on the same network as the ISP doing the blocking.

---

### Client-side usage overview

On your client device (PC, phone, TV box, etc.):

1. **Install a Shadowsocks client** (e.g. Shadowsocks-Android, Shadowsocks-Windows, Shadowsocks for macOS, or router firmware with Shadowsocks support).
2. Configure the client with:
   - **Server address**: Public IP or domain of your VPS/home server.
   - **Port**: `443` (as mapped in the Docker service).
   - **Password**: The same value you set as `(PASSWORD)` in the compose file.
   - **Encryption method**: `chacha20-ietf-poly1305`.
3. Either:
   - Set your system or app to use the **SOCKS5/HTTP proxy** provided by the client, or
   - Use **global mode** (all traffic through Shadowsocks) if your client supports it.

For IPTV:

- Point your IPTV player/app to use the local Shadowsocks proxy (if supported), or
- Run the Shadowsocks client in global/system mode so IPTV traffic is transparently routed through the tunnel.

---

### Security and best practices

- **Use a strong password** for `-k (PASSWORD)`:
  - Long, random, and unique.
- **Limit access**:
  - Consider firewall rules so only you (or your region) can access port 443.
  - Optionally put Shadowsocks behind an additional layer (e.g. Cloudflare Spectrum or other protections) if exposed publicly.
- **Monitor usage**:
  - Watch for unusual bandwidth spikes or unauthorized use.

With this setup, Shadowsocks provides a **lightweight, high-performance encrypted tunnel** that can effectively bypass ISP-level blocking and throttling of IPTV and similar traffic.

