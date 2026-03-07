## Uptime Kuma – Docker Setup

This directory contains a minimal Docker Compose setup for running **Uptime Kuma**.

### 1. Prerequisites

- **Docker** and **Docker Compose** installed
- A host machine with port **3001/tcp** available

### 2. Files

- **`docker-compose.yaml`** – defines the Uptime Kuma container and volume

### 3. How to start Uptime Kuma

From this directory, run:

```bash
docker compose up -d
```

Then open Uptime Kuma in your browser:

- `http://<your-server-ip>:3001`

On first run, you will be asked to create an admin account and can then start adding monitors.

### 4. Data persistence

All application data is stored in the `./uptime-kuma-data` directory (mapped to `/app/data` in the container).  
This ensures that your configuration and history are preserved across container restarts.

### 5. Full `docker-compose.yaml`

Below is an exact copy of the `docker-compose.yaml` in this directory:

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:2
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./uptime-kuma-data:/app/data
    environment:
      - UPTIME_KUMA_PORT=3001
    stop_grace_period: 30s
```

