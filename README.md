# Dozzle Setup for Container Log Monitoring

This document provides a step-by-step guide to set up Dozzle, a lightweight log viewer for Docker containers, including the Dozzle server, optional agent for remote servers, generating an admin password, and optionally configuring an Nginx reverse proxy for secure access.

## 🧱 Architecture Overview

```
+--------------------------+      +--------------------------+
|    Dozzle Agent (VM A)   |      |    Dozzle Agent (VM B)   |
|  hostname: Stage-Droplet |      |  hostname: Prod-Droplet  |
|  Port: 7007              |      |  Port: 7007              |
+------------+-------------+      +-------------+------------+
             \                              /
              \                            /
               \                          /
              +------------------------------+
              |     Dozzle Main Server       |
              | Port: 8085                   |
              | Auth: Simple (users.yml)     |
              +------------------------------+
```

- The **Dozzle Main Server** runs on a central server, providing the web interface for viewing logs.
- **Dozzle Agents** (optional) run on remote servers to collect and forward container logs to the main server.
- **Note**: If running Dozzle on a single server, the agent is not needed, as the main `docker-compose.yml` configuration can directly access local container logs via `/var/run/docker.sock`.

## Prerequisites
- Docker installed on your system.
- Docker Compose installed on your system.
- Access to a terminal with root or sudo privileges.
- Network connectivity between the Dozzle server and agent hosts (if using agents).
- (Optional) Nginx installed and Let’s Encrypt SSL certificates configured for reverse proxy setup.

## Step 1: Set Up the Dozzle Agent (Optional for Remote Servers)
The Dozzle agent collects logs from Docker containers on remote servers and forwards them to the Dozzle server. If running Dozzle on a single server, skip this step, as the main server configuration handles local container logs.

1. **Create the Agent Docker Compose File**
   Create a file named `docker-compose-agent.yml` with the following content:

   ```yaml
   services:
     dozzle-agent:
       image: amir20/dozzle:latest
       container_name: dozzle-agent
       command: agent --hostname Stage-Droplet
       restart: unless-stopped
       ports:
         - 7007:7007
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock:ro
       healthcheck:
         test: ["CMD", "/dozzle", "healthcheck"]
         interval: 5s
         retries: 5
         start_period: 5s
         start_interval: 5s
   ```

2. **Deploy the Agent**
   Navigate to the directory containing `docker-compose-agent.yml` and run:
   ```bash
   docker-compose -f docker-compose-agent.yml up -d
   ```

   This starts the Dozzle agent, exposing port `7007` for communication with the Dozzle server.

## Step 2: Set Up the Dozzle Server
The Dozzle server provides the web interface to view container logs, either from local containers or remote agents.


1. **Create the Server Docker Compose File**
   Create a file named `docker-compose.yml` with the following content:

   ```yaml
   services:
     dozzle:
       image: amir20/dozzle:latest
       container_name: dozzle
       environment:
         - DOZZLE_AUTH_PROVIDER=simple
         - DOZZLE_REMOTE_AGENT=10.200.0.4:7007,10.100.0.2:7007
       volumes:
         - /var/run/docker.sock:/var/run/docker.sock
         - ./data:/data
       ports:
         - 8085:8080

   ```

   - `DOZZLE_AUTH_PROVIDER=simple` enables basic authentication.
   - `DOZZLE_REMOTE_AGENT` specifies the IP addresses and ports of remote Dozzle agents (e.g., `10.118.0.4:7007` and `10.118.0.2:7007`). Omit this if only monitoring local containers.
   - The `./data:/data` volume maps a local directory to persist authentication data.

2. **Create the Data Directory**
   Create the `./data` directory in the same location as `docker-compose.yml`:
   ```bash
   mkdir data
   ```

3. **Deploy the Server**
   Navigate to the directory containing `docker-compose.yml` and run:
   ```bash
   docker-compose up -d
   ```

   This starts the Dozzle server, accessible at `http://<server-ip>:8085`.

## Step 3: Generate Admin Password
To secure access to the Dozzle web interface, generate an admin user with a password and save it to a `users.yml` file.

1. **Run the Password Generation Command**
   Execute the following command to generate an admin user:
   ```bash
   docker run --rm amir20/dozzle generate --name Admin --email me@email.net --password secret admin
   ```

   - Replace `Admin` with the desired username.
   - Replace `me@email.net` with the admin email.
   - Replace `secret` with a secure password.
   - The `admin` argument specifies the role.

2. **Save the Output to `users.yml`**
   The command outputs a YAML configuration. Save this output to a file named `users.yml` or `users.yaml`. For example:
   ```bash
   docker run --rm amir20/dozzle generate --name Admin --email me@email.net --password secret admin > users.yml
   ```

3. **Copy the File to the Data Directory**
   Move the `users.yml` file to the `./data` directory:
   ```bash
   mv users.yml data/
   ```

   Ensure the file is readable by the Dozzle server.

4. **Verify Authentication**
   Access the Dozzle web interface at `http://<server-ip>:8085` and log in using the credentials specified in the generation command.

## Step 4: (Optional) Nginx Reverse Proxy Configuration
To securely expose the Dozzle service over HTTPS, configure an Nginx reverse proxy with SSL.

1. **Create Nginx Configuration**
   Create an Nginx configuration file (e.g., `/etc/nginx/sites-available/dozzle`) with the following content:

   ```nginx
   server {
       listen 80;
       server_name x.example.com;
       return 301 https://$host$request_uri;
   }

   server {
       listen 443 ssl;
       server_name x.example.com;

       ssl_certificate /etc/letsencrypt/live/x.example.com/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/x.example.com/privkey.pem;
       include /etc/letsencrypt/options-ssl-nginx.conf;
       ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

       location / {
           proxy_pass http://localhost:8085;
           proxy_http_version 1.1;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;

           # Headers required by Dozzle in forward-proxy mode
           # proxy_set_header Remote-User "admin";
           # proxy_set_header Remote-Email "admin@example.com";
           # proxy_set_header Remote-Name "Admin";
       }
   }
   ```

   - Replace `x.example.com` with your actual domain.
   - Ensure the SSL certificate paths (`/etc/letsencrypt/...`) match your Let’s Encrypt setup.
   - The commented headers (`Remote-User`, `Remote-Email`, `Remote-Name`) are optional and used for forward-proxy authentication if required by Dozzle.
   - DOZZLE_AUTH_PROVIDER=forward-proxy

2. **Enable the Nginx Configuration**
   Create a symbolic link to enable the configuration:
   ```bash
   sudo ln -s /etc/nginx/sites-available/dozzle /etc/nginx/sites-enabled/
   ```

3. **Test and Reload Nginx**
   Test the Nginx configuration for errors:
   ```bash
   sudo nginx -t
   ```
   If the test is successful, reload Nginx:
   ```bash
   sudo systemctl reload nginx
   ```

4. **Access Dozzle via HTTPS**
   Access the Dozzle interface at `https://x.example.com` and log in with the admin credentials.

## Step 5: Access Dozzle
- Open a web browser and navigate to `http://<server-ip>:8085` (or `https://x.example.com` if using Nginx).
- Log in with the admin credentials.
- You should now see the logs from the containers managed by the connected Dozzle agents (or local containers if no agents are used).

## Troubleshooting
- **Agent Not Connecting**: Ensure the agent is running and the specified IP addresses and ports in `DOZZLE_REMOTE_AGENT` are correct and accessible. If monitoring local containers, ensure `DOZZLE_REMOTE_AGENT` is not set unnecessarily.
- **Authentication Issues**: Verify that the `users.yml` file is in the `./data` directory and readable by the Dozzle server.
- **Port Conflicts**: Ensure ports `8085` (server) and `7007` (agent) are not used by other services.
- **Nginx Errors**: Check Nginx logs (`/var/log/nginx/error.log`) for issues related to SSL or proxy configuration.

## Notes
- The `restart: unless-stopped` policy ensures the agent and server restart automatically unless explicitly stopped.
- The healthcheck configuration in the agent ensures the container is healthy and responsive.
- Use strong passwords for production environments to secure the Dozzle interface.
- Ensure your SSL certificates are up-to-date if using the Nginx reverse proxy.
