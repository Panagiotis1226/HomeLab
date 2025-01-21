# Dynamically Scalable Web-Server

### 1. Use the [docker-compose.yaml](docker-compose.yaml) or copy the following:

```yaml
services:
  apache:
    image: httpd:latest
    expose:
      - "80"  # Expose port 80 internally within the Docker network
    volumes:
      - apache_data:/var/www/html  # OPTIONAL: Persistent volume (keep data)
    networks:
      - webnet

  nginx:
    image: nginx:latest
    ports:
      - "8080:80"  # Make NGINX accessible on host port 8080
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro  # Custom NGINX config
    depends_on:
      - apache
    networks:
      - webnet

networks:
  webnet:
```
### 2. Use the [nginx.conf](nginx.conf) file or copy the following:

```nginx
events {}
http {
    upstream apache_backend {
        server apache:80;  # Docker will internally load balance replicas
    }

    server {
        listen 80;

        location / {
            proxy_pass http://apache_backend;
        }
    }
}
```

### 3. Run the `docker compose up -d` command to start the containers.

### 4. Access the web server at `http://localhost:8080` or `http://<your-ip>:8080`.

### 5. To scale the web server, use the `docker compose --scale apache=<number>` command.
`docker compose --scale apache=3`

This will create 3 Apache containers (3 in total) and load balance the requests between them.

### 5.1 To scale down the web server, use the `docker compose --scale apache=<number>` command.
`docker compose --scale apache=1`

This will scale down the web server to 1 Apache container (1 in total).

### 6. Force a reload of the NGINX configuration
`docker kill -s HUP <nginx-container-id>`

This will reload the NGINX configuration without downtime.

### 7. To stop the web server, use the `docker compose stop` command.

### 8. To remove the web server, use the `docker compose down` command.

This will stop and remove all containers and networks.
