# Watchtower for one time Container Updates

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
