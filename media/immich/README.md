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
