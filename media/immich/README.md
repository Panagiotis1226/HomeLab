# media/immich/installation.md

# Immich Installation Guide

## Prerequisites
- Docker and Docker Compose installed
- Terminal access to your server
- Sufficient storage space for your media

## Installation Steps

### 1. Create Application Directory
```bash
mkdir {path}/immich-app
cd {path}/immich-app
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

### 2. Configure .env File
Edit the .env file to set your desired configuration.
Example: 
```yaml 
TZ= {your_timezone}
UPLOAD_LOCATION= {your_media_path}
DB_DATA_LOCATION= {your_db_path}
DB_PASSWORD= {your_db_password}
```
### 3. Run Docker Compose
```bash
docker compose up -d
```
### 4. Access the Web Interface
Open your browser and navigate to `http://{your_server_ip}:2282`.
Next, you will need to create an admin user.

### 5. Access Mobile App
Insert the following into the mobile app:
```
http://{your_server_ip}:2282
```
Then, you will need to login with the admin user you created in the web interface.

### 6. Updating Immich
To update Immich, simply run the following command:
```bash
docker compose pull
docker compose up -d
```
### 7. Optional: Enable HTTPS, and Reverse Proxy
```bash
apt update && apt install caddy -y
nano /etc/caddy/Caddyfile
```
Replace the following to the Caddyfile:
```yaml
#:80 {
	# Set this path to your site's directory.
	#root * /usr/share/caddy

	# Enable the static file server.
	#file_server

	# Another common task is to set up a reverse proxy:
	# reverse_proxy localhost:8080

	# Or serve a PHP site through php-fpm:
	# php_fastcgi localhost:9000
#}

# Adding the new site with a custom TLS certificate and reverse proxy:
# Reverse proxy with custom TLS for to server IP (change this IP to the server ip)
https://{server_ip} {
	tls /etc/ssl/certs/immich-selfsigned.crt /etc/ssl/private/immich-selfsigned.key
	reverse_proxy 127.0.0.1:2283
}

```
