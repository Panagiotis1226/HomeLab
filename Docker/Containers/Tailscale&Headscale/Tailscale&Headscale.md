# Headscale with Caddy & Let's Encrypt

Self-hosted Headscale (Tailscale control server) behind Caddy with TLS via Let's Encrypt.

## Prerequisites

- Docker & Docker Compose
- A domain pointing to your server (e.g. `tail.yourdomain.com`)
- Ports 80/tcp, 443/tcp, 41641/udp, and 3478/udp available and port-forwarded

---

## 1. Get SSL Certificate

Obtain a certificate with Certbot (standalone). **Stop any service using port 80** before running:

```bash
docker run -it --rm -p 80:80 \
  -v /etc/letsencrypt:/etc/letsencrypt \
  certbot/certbot certonly --standalone -d tail.yourdomain.com
```

Replace `tail.yourdomain.com` with your actual domain.

---

## 2. Caddyfile

Create a `Caddyfile` in your project directory:

```caddyfile
tail.yourdomain.com {
    tls /etc/letsencrypt/live/tail.yourdomain.com/fullchain.pem /etc/letsencrypt/live/tail.yourdomain.com/privkey.pem
    reverse_proxy headscale:8080
}
```

> **Note:** Update both the server name and the TLS paths to match your domain and Certbot output.

---

## 3. Docker Compose

Save as `docker-compose.yaml`:

```yaml
services:
  headscale:
    image: headscale/headscale:0.28.0
    container_name: headscale
    restart: unless-stopped
    ports:
      - "3478:3478/udp"
      - "41641:41641/udp"
    expose:
      - "8080"
    volumes:
      - ./config:/etc/headscale
      - ./data:/var/lib/headscale
    command: serve

  caddy:
    image: caddy:latest
    container_name: caddy
    restart: unless-stopped
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - ./caddy_data:/data
      - ./caddy_config:/config
    command: caddy run --config /etc/caddy/Caddyfile
```

---

## 4. Headscale config

Create `config/config.yaml`:

```yaml
server_url: https://tail.yourdomain.com
listen_addr: 0.0.0.0:8080
metrics_listen_addr: 127.0.0.1:9090
grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false

noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48
  allocation: sequential

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "peter"
    region_name: "Peter Home"
    verify_clients: true
    stun_listen_addr: "0.0.0.0:3478"
    private_key_path: /var/lib/headscale/derp_server_private.key
    automatically_add_embedded_derp_region: true
    ipv4: <Public IP of Your DERP Server>
    http_port: -1
    stun_port: 3478

  urls: []
  paths: []
  auto_update_enabled: false
  update_frequency: 24h

ephemeral_node_inactivity_timeout: 30m

database:
  type: sqlite
  debug: false
  gorm:
    prepare_stmt: true
    parameterized_queries: true
    skip_err_record_not_found: true
    slow_threshold: 1000
  sqlite:
    path: /var/lib/headscale/db.sqlite
    write_ahead_log: true
    wal_autocheckpoint: 1000

acme_url: https://acme-v02.api.letsencrypt.org/directory
acme_email: ""
tls_letsencrypt_hostname: ""
tls_letsencrypt_cache_dir: /var/lib/headscale/cache
tls_letsencrypt_challenge_type: HTTP-01
tls_letsencrypt_listen: ":http"

tls_cert_path: ""
tls_key_path: ""

log:
  level: info
  format: text

policy:
  mode: file
  path: ""

dns:
  magic_dns: true
  base_domain: yourdomain.local
  override_local_dns: true
  nameservers:
    global:
      - 1.1.1.1
      - 8.8.8.8
  search_domains: []
  extra_records: []

unix_socket: /var/run/headscale/headscale.sock
unix_socket_permission: "0770"

logtail:
  enabled: false

randomize_client_port: false

taildrop:
  enabled: true
```

Replace `tail.yourdomain.com` and `yourdomain.local` with your domain and preferred MagicDNS base domain.

Replace DERP server IP

---

## 5. Run

```bash
docker compose up -d
```

Headscale will be available at `https://tail.yourdomain.com` (or your configured URL).

---

## 6. Allow Exit Node + Subnet routes

### On the node you want to connect to the network

```bash
tailscale up \
  --login-server https://tail.yourdomain.com \
  --authkey KEY \
  --advertise-exit-node \
  --advertise-routes=192.168.1.0/24,192.168.2.0/24 \
  --accept-routes \
  --accept-dns
```

### On Headscale server

Approve routes:

```bash
docker exec headscale headscale nodes approve-routes -r 192.168.1.0/24,0.0.0.0/0,192.168.2.0/24 -i ID_NUMBER
```

Verify:

```bash
docker exec headscale headscale nodes list-routes
```

Create pre-auth key:

```bash
docker exec headscale headscale preauthkeys create -u ID
```

---

## File layout

```
.
├── README.md
├── Caddyfile
├── docker-compose.yaml
├── config/
│   └── config.yaml
├── data/           # Headscale state (created by container)
├── caddy_data/     # Caddy data (created by container)
└── caddy_config/   # Caddy config (created by container)
```
