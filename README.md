# Conflux Deployment Guide

This README explains how to set up and deploy the Conflux application stack (frontend, backend, and database) using Docker Compose. This guide assumes you will be running the application behind your own reverse proxy.

## Prerequisites

*   Docker and Docker Compose installed
*   A reverse proxy (like Nginx, Caddy, or Traefik) set up, or willingness to follow the included Nginx setup guide
*   At least one domain name configured to point to your server. You can choose either:
    *   **Option 1 (Simpler)**: One domain for both services using subpaths (e.g., `example.com` for frontend, `example.com/api/` for API)
    *   **Option 2**: Two separate domains/subdomains (e.g., `conflux.example.com` for frontend, `api.conflux.example.com` for API)
*   Git installed
*   Basic understanding of Docker and networking

## Setup Instructions

### 1. Clone the Repositories

First, clone the required repositories.

```bash
# Clone frontend repository
git clone https://github.com/team-golfslag/Conflux-web.git

# Clone backend repository
git clone https://github.com/team-golfslag/Conflux.git

# Clone this deployment repository (or create your own)
# git clone https://github.com/your-organization/conflux-deployment.git
# cd conflux-deployment
```

### 2. Build Docker Images

You can build the Docker images locally.

1.  **Configure Frontend Environment Variables**

    Before building the frontend image, you must create a `.env.production` file in the `Conflux-web` directory with the correct URLs for your API and web UI.

    ```bash
    # Navigate to the frontend repository
    cd Conflux-web

    # Create and edit the production environment file
    nano .env.production
    ```

    Add the following content, replacing the placeholder values:

    ```
    # File: Conflux-web/.env.production
    VITE_API_URL=https://<API_DOMAIN>
    VITE_WEBUI_URL=https://<WEB_UI_DOMAIN>
    ```

2.  **Build the Images**

    Now, build the images for both the frontend and backend.

    ```bash
    # Build frontend image from the Conflux-web directory
    docker build -t conflux-web:latest .

    # Navigate to the backend repository
    cd ../Conflux

    # Build backend image from the Conflux directory
    docker build -t conflux-backend:latest .
    ```

### 3. Configure the Deployment

The backend service is configured using environment variables set in the `docker-compose.yml` file. This method overrides the default values found in `Conflux.API/appsettings.json`.

1.  **Create an Environment File**

    For security, it's best to store your secrets and configuration in a `.env` file in the same directory as your `docker-compose.yml`. Docker Compose will automatically load it.

    ```bash
    # Create the .env file
    nano .env
    ```

    Add the following key-value pairs, replacing all placeholders with your actual production values.

    ```dotenv
    # .env file

    # PostgreSQL Credentials
    DB_HOST=postgres
    DB_NAME=conflux-prod
    DB_USER=conflux-user
    DB_PASSWORD=a_very_strong_and_secret_password

    # Application Secrets
    SRAM_CLIENT_SECRET=your_sram_client_secret_here
    SRAM_SCIM_SECRET=your_sram_scim_secret_here
    ORCID_CLIENT_SECRET=your_orcid_client_secret_here
    RAID_USERNAME=your_raid_service_account_username
    RAID_PASSWORD=your_raid_service_account_password
    WEBARCHIVE_ACCESS_KEY=<WEBARCHIVE_ACCESS_KEY>
    WEBARCHIVE_SECRET_KEY=<WEBARCHIVE_SECRET_KEY>

    # Backend Configuration Overrides
    # See the "Backend Configuration" section for more details.
    # --- Connection String ---
    ConnectionStrings__Database="Host=postgres;Database=${DB_NAME};Username=${DB_USER};Password=${DB_PASSWORD}"

    # --- CORS ---
    Cors__AllowedOrigins__0="https://<WEB_UI_DOMAIN>"

    # --- SRAM Authentication ---
    Authentication__SRAM__ClientId="YOUR_SRAM_CLIENT_ID"
    Authentication__SRAM__RedirectUri="https://<API_DOMAIN>/login/redirect"
    Authentication__SRAM__AllowedRedirectUris__0="https://<WEB_UI_DOMAIN>/dashboard"

    # --- ORCID Authentication ---
    Authentication__Orcid__ClientId="YOUR_ORCID_CLIENT_ID"
    Authentication__Orcid__RedirectUri="https://<API_DOMAIN>/orcid/redirect"
    Authentication__Orcid__AllowedRedirectUris__0="https://<WEB_UI_DOMAIN>/profile"

    # --- Feature Flags for Production ---
    FeatureFlags__Swagger="false"
    FeatureFlags__SeedDatabase="false"
    FeatureFlags__DatabaseConnection="true"
    FeatureFlags__SRAMAuthentication="true"
    FeatureFlags__OrcidAuthentication="true"
    FeatureFlags__ReverseProxy="true"
    FeatureFlags__HttpsRedirect="true"
    ```

2.  **Review `docker-compose.yml`**

    Ensure your `docker-compose.yml` file is ready. The provided `docker-compose.yml` is already set up to use the variables from your `.env` file.

### 4. Start the Services

With your configuration in place, start all services in detached mode.

```bash
docker-compose up -d
```

### 5. Verify Deployment

Check that all containers are running and healthy.

```bash
docker-compose ps
```

You should see `postgres`, `conflux-web`, and `conflux-backend` services with a status of `Up` or `running`.

## Reverse Proxy Configuration (Nginx)

The Conflux application stack requires a reverse proxy to handle incoming internet traffic, provide SSL termination, and route requests to the appropriate services. An example Nginx configuration file (`nginx.conf.example`) is provided in this repository.

### Setting Up Nginx

1. **Install Nginx and Certbot**

   ```bash
   # On Ubuntu/Debian
   sudo apt update
   sudo apt install nginx certbot python3-certbot-nginx

   # On CentOS/RHEL
   sudo yum install nginx certbot python3-certbot-nginx
   ```

2. **Choose Your Domain Configuration**

   The `nginx.conf.example` file provides two configuration options:

   **Option 1: Single Domain with Subpaths (Recommended for simpler setups)**
   - Replace `<DOMAIN>` with your domain name (e.g., `example.com`)
   - Frontend will be accessible at: `https://example.com/`
   - API will be accessible at: `https://example.com/api/`
   - Requires only one SSL certificate
   - Simpler DNS setup

   **Option 2: Separate Domains/Subdomains**
   - Replace `<WEB_UI_DOMAIN>` with your frontend domain (e.g., `conflux.example.com`)
   - Replace `<API_DOMAIN>` with your API domain (e.g., `api.conflux.example.com`)
   - More separation between services
   - May require separate SSL certificates

   Choose one configuration and remove or comment out the other in the nginx.conf.example file.

3. **Install the Configuration**

   ```bash
   # Copy the example configuration to Nginx sites-available
   sudo cp nginx.conf.example /etc/nginx/sites-available/conflux

   # Enable the site
   sudo ln -s /etc/nginx/sites-available/conflux /etc/nginx/sites-enabled/

   # Test the configuration
   sudo nginx -t

   # Reload Nginx
   sudo systemctl reload nginx
   ```

4. **Obtain SSL Certificates**

   Use Certbot to obtain SSL certificates:

   **For Option 1 (Single Domain):**
   ```bash
   sudo certbot --nginx -d <DOMAIN>
   ```

   **For Option 2 (Separate Domains):**
   ```bash
   sudo certbot --nginx -d <WEB_UI_DOMAIN> -d <API_DOMAIN>
   ```

   Certbot will automatically update your Nginx configuration with the SSL certificate paths.

### Understanding the Nginx Configuration

The provided Nginx configuration includes several important sections:

- **HTTP to HTTPS Redirect**: All HTTP traffic is automatically redirected to HTTPS for security
- **SSL Configuration**: Uses Let's Encrypt certificates managed by Certbot
- **Proxy Headers**: Properly forwards client information to the backend services
- **Separate Logging**: Each service has its own access and error logs for easier troubleshooting
- **WebSocket Support**: Includes headers for WebSocket connections if needed
- **ACME Challenge**: Allows Certbot to renew certificates automatically

#### Port Mapping

The Nginx configuration routes traffic as follows:

**Option 1 (Single Domain):**
- `https://<DOMAIN>/` → `127.0.0.1:8081` (conflux-web service)
- `https://<DOMAIN>/api/` → `127.0.0.1:8082` (conflux-backend service)

**Option 2 (Separate Domains):**
- `https://<WEB_UI_DOMAIN>/` → `127.0.0.1:8081` (conflux-web service)
- `https://<API_DOMAIN>/` → `127.0.0.1:8082` (conflux-backend service)

These ports correspond to the `ports` mapping in your `docker-compose.yml` file.

#### Environment Variable Alignment

Ensure your environment variables align with your chosen Nginx configuration:

**For Option 1 (Single Domain):**

```dotenv
# Frontend Environment Variables (Conflux-web/.env.production)
VITE_API_URL=https://<DOMAIN>/api
VITE_WEBUI_URL=https://<DOMAIN>

# Backend Environment Variables (.env)
Cors__AllowedOrigins__0="https://<DOMAIN>"
Authentication__SRAM__RedirectUri="https://<DOMAIN>/api/login/redirect"
Authentication__SRAM__AllowedRedirectUris__0="https://<DOMAIN>/dashboard"
Authentication__Orcid__RedirectUri="https://<DOMAIN>/api/orcid/redirect"
Authentication__Orcid__AllowedRedirectUris__0="https://<DOMAIN>/profile"
```

**For Option 2 (Separate Domains):**

```dotenv
# Frontend Environment Variables (Conflux-web/.env.production)
VITE_API_URL=https://<API_DOMAIN>
VITE_WEBUI_URL=https://<WEB_UI_DOMAIN>

# Backend Environment Variables (.env)
Cors__AllowedOrigins__0="https://<WEB_UI_DOMAIN>"
Authentication__SRAM__RedirectUri="https://<API_DOMAIN>/login/redirect"
Authentication__SRAM__AllowedRedirectUris__0="https://<WEB_UI_DOMAIN>/dashboard"
Authentication__Orcid__RedirectUri="https://<API_DOMAIN>/orcid/redirect"
Authentication__Orcid__AllowedRedirectUris__0="https://<WEB_UI_DOMAIN>/profile"
```

### Nginx Security Considerations

- **Firewall**: Ensure only ports 80 and 443 are open to the internet
- **SSL Configuration**: The configuration uses modern SSL settings provided by Certbot
- **Proxy Buffers**: Configured to handle larger responses efficiently
- **Log Rotation**: Consider setting up log rotation for `/var/log/nginx/` files

## Backend Configuration

The .NET backend uses a flexible configuration system. While default settings are present in `Conflux.API/appsettings.json`, for deployment you **must** override them using environment variables. This avoids committing secrets and allows for environment-specific setups.

As per the [official Microsoft documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration/?view=aspnetcore-9.0), you can override any setting by creating an environment variable that matches the JSON path, using a double underscore (`__`) to separate keys.

**Example:**

To override the `ClientId` within the `SRAM` section of the `Authentication` block:

*   **JSON Structure:**
    ```json
    "Authentication": {
      "SRAM": {
        "ClientId": "some-default-id"
      }
    }
    ```
*   **Environment Variable:**
    ```
    Authentication__SRAM__ClientId="YOUR_PRODUCTION_CLIENT_ID"
    ```

To override an item in a JSON array, use a zero-based index:

*   **JSON Structure:**
    ```json
    "Cors": {
      "AllowedOrigins": [
        "http://localhost:5173"
      ]
    }
    ```
*   **Environment Variable:**
    ```
    Cors__AllowedOrigins__0="https://your-frontend-domain.com"
    ```

The `.env` file created in Step 3 already includes the most critical overrides needed for a production environment.

## Architecture Overview

This deployment consists of three core services orchestrated by Docker Compose:

1.  **PostgreSQL Database (`postgres`)**: A dedicated, persistent database service for storing all application data.
2.  **Frontend (`conflux-web`)**: The user-facing web application built with Vue.js.
3.  **Backend (`conflux-backend`)**: The .NET API that handles business logic, authentication, and database interactions.

This stack is designed to be placed behind a **reverse proxy** that handles incoming internet traffic, provides SSL termination, and routes requests to the appropriate service. See the [Reverse Proxy Configuration (Nginx)](#reverse-proxy-configuration-nginx) section for a complete example.

*   Traffic for `https://<WEB_UI_DOMAIN>` should be routed to `conflux-web` (port `8081` on the host).
*   Traffic for `https://<API_DOMAIN>` should be routed to `conflux-backend` (port `8082` on the host).

## Network Configuration

The setup uses two isolated Docker networks for security and organization:

*   `app_network`: Connects the `conflux-backend` and `conflux-web` services.
*   `db_network`: Connects the `conflux-backend` service to the `postgres` database, isolating the database from the frontend.

## Troubleshooting

*   **Container not starting:** Check the logs for a specific service with `docker-compose logs <service-name>` (e.g., `docker-compose logs conflux-backend`).
*   **Database Connection Issues:** Ensure the `DB_` variables in your `.env` file are correct and that the `ConnectionStrings__Database` override is properly formatted.
*   **Authentication Fails (400/500 errors):** Double-check that your `RedirectUri` and `AllowedRedirectUris` in the environment variables exactly match the URLs registered with your SRAM and ORCID applications. Also, verify your client IDs and secrets are correct.
*   **CORS Issues (Frontend can't reach API):** Make sure the `Cors__AllowedOrigins__0` variable is set to your exact frontend domain (`https://<WEB_UI_DOMAIN>`).
*   **502 Bad Gateway:** This usually indicates an issue with your reverse proxy configuration. Ensure your Nginx configuration is correctly routing traffic to the host ports (`127.0.0.1:8081` and `127.0.0.1:8082`). Check the Nginx error logs with `sudo tail -f /var/log/nginx/error.log`.

## Maintenance

*   **Updating the Application:**
    1.  Pull the latest changes for the repositories.
    2.  Rebuild the Docker images that have changed (`docker build ...`).
    3.  Restart the services to apply the new images:
        ```bash
        docker-compose up -d --force-recreate
        ```
*   **Data Persistence:** The PostgreSQL data is stored in a Docker volume (`postgres_data`), so it will persist even if the container is removed or recreated.

## Data Backup

It is critical to implement a regular backup strategy for your production database.

```bash
# Example command to create a compressed backup file
docker-compose exec -T postgres pg_dump -U <DB_USER> -d <DB_NAME> | gzip > backup-$(date +%F).sql.gz
```

Replace `<DB_USER>` and `<DB_NAME>` with the values from your `.env` file. This command should be run regularly via a cron job or other scheduling tool.