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

### 7. Access Immich from Anywhere
To access Immich from anywhere outside your network, you'll need to set up a VPN.

See the [WireGuard Setup Guide](../../WireGuard/README.md) for detailed installation instructions.

### 8. Optional: Enable HTTPS, Self-Signed SSL, and Reverse Proxy

#### 8.1. Create Self-Signed SSL
```bash 
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/immich-selfsigned.key -out /etc/ssl/certs/immich-selfsigned.crt
```
#### 8.2. Install & Configure Caddy
```bash
apt update && apt install caddy -y
nano /etc/caddy/Caddyfile
```
Replace the following to the Caddyfile:
```Caddyfile
# Default site configuration (commented out)
#{
    # Set this path to your site's directory.
    #root * /usr/share/caddy

    # Enable the static file server.
    #file_server

    # Another common task is to set up a reverse proxy:
    # reverse_proxy localhost:8080

    # Or serve a PHP site through php-fpm:
    # php_fastcgi localhost:9000
#}

# Reverse proxy with custom TLS for server IP (change this IP to the server ip)
https://{server_ip} {
    tls /etc/ssl/certs/immich-selfsigned.crt /etc/ssl/private/immich-selfsigned.key
    reverse_proxy 127.0.0.1:2283
}

# Redirect all HTTP traffic on port 80 to HTTPS 
http://{server_ip} {
    redir https://{server_ip}{uri} permanent
}
```

#### 8.3. Fix Permission Issues with SSL
If you have permission issues, you can run the following command to fix them:
```bash
sudo chown -R caddy:caddy /etc/ssl/private /etc/ssl/certs
sudo chmod 755 /etc/ssl/certs
sudo chmod 700 /etc/ssl/private
sudo chmod 644 /etc/ssl/certs/immich-selfsigned.crt
sudo chmod 600 /etc/ssl/private/immich-selfsigned.key
```
Then restart the Caddy service:
```bash
systemctl restart caddy
```
#### 8.4. New Immich Access
New Immich URL:
```
https://{server_ip}
```

For Mobile App:
Go to advanced settings and turn on allow self-signed SSL certificates.
Then use the new URL to login.
