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

### 2. Configure Backend Settings

Before building the Docker images, you need to configure the backend settings:

1. Navigate to the backend repository and edit the `Conflux.API/appsettings.json` file:

```bash
cd ../Conflux
# Edit the file with your preferred editor
nano Conflux.API/appsettings.json
```

2. Update the following important settings:

```json
{
  "Authentication": {
    "SRAM": {
      "ClientId": "YOUR_SRAM_CLIENT_ID",
      "RedirectUri": "https://<API_DOMAIN>/login/redirect",
      "AllowedRedirectUris": [
        "https://<WEB_UI_DOMAIN>/dashboard"
      ]
    },
    "Orcid": {
      "ClientId": "YOUR_ORCID_CLIENT_ID",
      "RedirectUri": "https://<API_DOMAIN>/orcid/redirect", 
      "AllowedRedirectUris": [
        "https://<WEB_UI_DOMAIN>/profile"
      ]
    }
  },
  "Cors": {
    "AllowedOrigins": [
      "https://<WEB_UI_DOMAIN>"
    ]
  },
  "FeatureFlags": {
    "Swagger": false,
    "SeedDatabase": false,
    "DatabaseConnection": true,
    "SRAMAuthentication": true,
    "OrcidAuthentication": true,
    "ReverseProxy": true,
    "HttpsRedirect": true
  }
}
```

Replace all `<WEB_UI_DOMAIN>` and `<API_DOMAIN>` placeholders with your actual domain names. The client IDs should match your registered applications with SRAM and ORCID.

### 3. Build Docker Images

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

### 4. Configure Frontend Environment Variables

Before building the frontend image, you must update the environment variables in `.env.production` file:

```
# Frontend environment variables (.env.production)
VITE_API_URL=https://<API_DOMAIN>
VITE_WEBUI_URL=https://<WEBUI_DOMAIN>
```

Replace all placeholder values with your actual configuration.

### 5. Configure Deployment Environment

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

### 6. Start the Services

```bash
docker-compose up -d
```

This will start all services in detached mode.

### 7. Verify Deployment

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

## Backend Configuration Details

The backend requires several important configuration settings:

- **SRAM Authentication**: Configure the SRAM OAuth provider settings
  - `ClientId`: Your application's ID in SRAM
  - `RedirectUri`: Must match your API domain with the correct callback path
  - `AllowedRedirectUris`: Where users are redirected after login (frontend domain)

- **ORCID Authentication**: Configure the ORCID integration
  - `ClientId`: Your registered ORCID application ID
  - `RedirectUri`: API domain with correct callback path
  - `AllowedRedirectUris`: Frontend redirect endpoint

- **CORS Configuration**: Must include your frontend domain for proper cross-origin requests

- **Feature Flags**: Enable/disable specific features
  - `Swagger`: Enable for API documentation (development only)
  - `SeedDatabase`: Enable only for initial database setup
  - `DatabaseConnection`: Must be true for production
  - `SRAMAuthentication`: Enable SRAM login
  - `OrcidAuthentication`: Enable ORCID integration
  - `ReverseProxy`: Should be true when behind nginx
  - `HttpsRedirect`: Should be true for production

## Troubleshooting

- If SSL certificates aren't being issued, check that your domains point to the server's IP address
- For database connection issues, verify the database credentials are consistent
- Check container logs with `docker-compose logs [service-name]`
- If authentication fails, verify the client IDs and secrets match your registered applications
- For CORS issues, ensure your frontend domain is properly listed in the backend configuration

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