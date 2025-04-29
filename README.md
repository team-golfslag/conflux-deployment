# Conflux Deployment Guide

This README explains how to set up and deploy the Conflux application stack (frontend and backend) using Docker Compose.

## Prerequisites

- Docker and Docker Compose installed
- Domain names configured for your web interface and API
- Git installed
- Basic understanding of Docker and networking

## Setup Instructions

### 1. Clone the Repositories

First, clone both the frontend and backend repositories:

```bash
# Clone frontend repository
git clone https://github.com/team-golfslag/Conflux-web.git

# Clone backend repository
git clone https://github.com/team-golfslag/Conflux.git

# Clone this deployment repository
git clone https://github.com/your-organization/conflux-deployment.git
cd conflux-deployment
```

### 2. Build Docker Images

You have two options for building the Docker images:

#### Option 1: Build locally

```bash
# Build frontend image
cd ../Conflux-web
# Update environment variables in .env.production first! (see details below)
docker build -t conflux-web:latest .

# Build backend image
cd ../Conflux
docker build -t conflux-backend:latest .
```

#### Option 2: Fork repositories and set up CI/CD

If you prefer automated builds:
1. Fork both repositories to your own GitHub organization/account
2. Configure GitHub Actions or other CI/CD pipeline to build and publish the images
3. Update the docker-compose.yml to use your published image names and tags

### 3. Configure Frontend Environment Variables

Before building the frontend image, you must update the environment variables in `.env.production` file:

```
# Frontend environment variables (.env.production)
VITE_API_URL=https://<API_DOMAIN>
VITE_WEBUI_URL=https://<WEBUI_DOMAIN>
```

Replace all placeholder values with your actual configuration.

### 4. Configure Deployment Environment

Edit the docker-compose.yml file to replace all placeholder values:

- `<EMAIL_ADDRESS>`: Your email for Let's Encrypt notifications
- `<DB_USER>`: PostgreSQL database username
- `<DB_PASSWORD>`: Strong password for PostgreSQL database
- `<DB_NAME>`: Name for the PostgreSQL database
- `<DOMAIN>`: Domain name for the web frontend (e.g., app.example.com)
- `<API_DOMAIN>`: Domain name for the backend API (e.g., api.example.com)
- `<SRAM_CLIENT_SECRET>`: Authentication secret for SRAM client
- `<SRAM_SCIM_SECRET>`: Authentication secret for SRAM SCIM client
- `<ORCID_CLIENT_SECRET>`: Authentication secret for ORCID client

### 5. Start the Services

```bash
docker-compose up -d
```

This will start all services in detached mode.

### 6. Verify Deployment

Check that all containers are running:

```bash
docker-compose ps
```

## Architecture Overview

This deployment consists of:

1. **Nginx Proxy**: Handles incoming HTTP/HTTPS requests and routes them to appropriate services
2. **ACME Companion**: Automatically obtains and renews Let's Encrypt SSL certificates
3. **PostgreSQL Database**: Stores application data
4. **Conflux Web**: Frontend application
5. **Conflux Backend**: API and backend application

## Network Configuration

The setup uses two Docker networks:
- `proxy_network`: For external communications between Nginx and web services
- `db_network`: For internal communications between the backend and database

## Troubleshooting

- If SSL certificates aren't being issued, check that your domains point to the server's IP address
- For database connection issues, verify the database credentials are consistent
- Check container logs with `docker-compose logs [service-name]`

## Notes and Clarifications Needed

- Are there any specific configuration requirements for the SRAM authentication?
- What roles/permissions are needed for the SRAM integration?
- Are there any backend environment variables that should be configured beyond those in the docker-compose file?
- What is the expected database schema migration process?

## Maintenance

- SSL certificates will renew automatically via the ACME companion
- Database data persists between container restarts
- To update container images after code changes:
  ```bash
  # If using locally built images:
  # 1. Rebuild images
  # 2. Restart containers
  docker-compose up -d --force-recreate

  # If using CI/CD pipelines:
  docker-compose pull
  docker-compose up -d
  ```

## Data Backup

Consider setting up a backup strategy for the PostgreSQL database:

```bash
# Example backup command
docker-compose exec postgres pg_dump -U <DB_USER> <DB_NAME> > backup.sql
```