# Watchtower for one time Container Updates

### 1. Create a `docker-compose.yaml` file for Watchtower.

Run `nano docker-compose.yaml` to create the file.

Add the following to the file:
```yaml
version: "3.8"
services:
  watchtower:
    container_name: watchtower
    image: containrrr/watchtower
    command: --cleanup --run-once
    restart: "no"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

## Run Watchtower once to update all containers

```bash
docker compose up -d
```
