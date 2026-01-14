## WireGuard Easy Docker Service

This guide explains what **WireGuard** is, how the provided **wg-easy** Docker Compose service works, and how it helps when your ISP blocks or interferes with traffic.

---

### What is WireGuard?

WireGuard is a **modern, lightweight VPN protocol** focused on:

- **Strong encryption** with a small codebase.
- **High performance and low latency** compared to many legacy VPNs.
- **Simple configuration** using key pairs instead of certificates.

In practice, WireGuard creates a secure tunnel between clients and the server, giving clients private IPs inside a VPN subnet. Traffic through the tunnel is encrypted, making it difficult for ISPs or intermediaries to inspect or block specific services.

---

### How this Docker service works

Service definition:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy-51820
    environment:
      - WG_HOST=(PUBLIC IP)
      - WG_PORT=51820
      - WG_UI_PORT=51821
      - WG_DEFAULT_ADDRESS=10.10.0.x
      - WG_DEFAULT_DNS=1.1.1.1
      - INSECURE=true			 # To Allow Access to the WebUI using HTTP
      - DISABLE_IPV6=true                 # To disable IPV6
    volumes:
      - ./wg-data:/etc/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    network_mode: host
    restart: unless-stopped
```

- **image**: Uses `wg-easy`, a Web UI wrapper around WireGuard for easy peer management.
- **WG_HOST**: Public IP or DNS name clients will use to reach your server.
- **WG_PORT**: UDP port for WireGuard (default 51820). Ensure it’s open on your firewall/router.
- **WG_UI_PORT**: HTTP port for the wg-easy admin UI (default 51821). Access via `http://server:51821`.
- **WG_DEFAULT_ADDRESS**: Base address for clients, e.g., `10.10.0.2` etc. (replace `x` with a number range).
- **WG_DEFAULT_DNS**: DNS given to clients (e.g., `1.1.1.1`).
- **INSECURE=true**: Allows HTTP (no TLS) for the UI; use only on trusted networks or put behind a reverse proxy with HTTPS.
- **DISABLE_IPV6=true**: Disables IPv6 inside WireGuard; set to `false` if you need IPv6.
- **volumes**: `./wg-data` persists server keys, peer configs, and QR codes.
- **cap_add**: `NET_ADMIN` and `SYS_MODULE` are required for WireGuard kernel module and networking.
- **network_mode: host**: Exposes WireGuard UDP port and UI directly on host network.
- **restart: unless-stopped**: Keeps the service running across reboots/crashes.

Bring it up from the directory with your `docker-compose.yaml`:

```bash
docker compose up -d
```

> Ensure UDP port `51820` is reachable from the internet (router port forward + firewall allow). If you change `WG_PORT`, update the forwarding rule.

---

### Why this helps against ISP blocking/throttling

ISPs may block or throttle specific destinations (streaming/IPTV), ports, or protocols, or use Deep Packet Inspection (DPI). With WireGuard:

- **All traffic is encrypted** between client and server; ISP sees only UDP packets to your server.
- **DPI evasion**: Encrypted tunnel hides which services you access (IPTV, streaming, gaming, etc.).
- **Consistent port**: Using a single UDP port (e.g., 51820) simplifies firewall rules and can avoid random port blocks. If your ISP heavily filters common VPN ports, you can change `WG_PORT` to something more generic.
- **Stable routing**: Traffic exits from the VPN server’s network location, bypassing local ISP restrictions on content.

You still need a **server in a location without those restrictions** (VPS, remote home, or cloud). Run this container on that server.

---

### Client setup (high level)

1. Install a WireGuard client (mobile, desktop, or router).
2. Access the wg-easy UI at `http://<WG_HOST>:<WG_UI_PORT>` to create peers.
3. Download the generated `.conf` or scan the QR code with your client.
4. Import the config and connect. The client receives an IP (e.g., `10.10.0.2`) and routes traffic through the encrypted tunnel.
5. For full-tunnel use, ensure the peer config has `AllowedIPs = 0.0.0.0/0`; for split tunnel, narrow it to specific subnets/services.

---

### Security and best practices

- **Secure the UI**: Because `INSECURE=true` serves over HTTP, limit UI access (VPN, LAN-only, or reverse proxy with HTTPS+auth). Set `WG_UI_PASSWORD` if desired.
- **Firewall**: Allow only necessary ports (UDP `WG_PORT`, TCP `WG_UI_PORT`), optionally restrict source IPs.
- **Back up** `./wg-data` to preserve keys/configs.
- **Change defaults**: Use strong randomness for keys; change `WG_PORT` if your ISP filters common VPN ports.
- **Monitor**: Keep an eye on bandwidth and peer lists to spot unauthorized use.

With this setup, WireGuard provides a fast, secure tunnel that can bypass ISP restrictions or throttling on IPTV, streaming, and other traffic. Clients connect via the VPN and their traffic exits from your server’s network, keeping local ISP filters out of the path.
