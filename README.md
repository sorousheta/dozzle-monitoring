# dozzle-monitoring
Dockerized-dozzle-monitoring Stack _ Server + Agent + Simple Auth


Dozzle Setup for Container Log Monitoring
This document provides a step-by-step guide to set up Dozzle, a lightweight log viewer for Docker containers, including the Dozzle server and agent, generating an admin password, and optionally configuring an Nginx reverse proxy for secure access.
Prerequisites

Docker installed on your system.
Docker Compose installed on your system.
Access to a terminal with root or sudo privileges.
Network connectivity between the Dozzle server and agent hosts.
(Optional) Nginx installed and Let’s Encrypt SSL certificates configured for reverse proxy setup.

Step 1: Set Up the Dozzle Agent
The Dozzle agent collects logs from Docker containers and forwards them to the Dozzle server.

Create the Agent Docker Compose FileCreate a file named docker-compose-agent.yml with the following content:
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


Deploy the AgentNavigate to the directory containing docker-compose-agent.yml and run:
docker-compose -f docker-compose-agent.yml up -d

This starts the Dozzle agent, exposing port 7007 for communication with the Dozzle server.


Step 2: Set Up the Dozzle Server
The Dozzle server provides the web interface to view container logs.

Create a Volume for Persistent DataCreate an external Docker volume to store Dozzle data:
docker volume create dozzle-data


Create the Server Docker Compose FileCreate a file named docker-compose.yml with the following content:
services:
  dozzle:
    image: amir20/dozzle:latest
    container_name: dozzle
    environment:
      - DOZZLE_AUTH_PROVIDER=simple
      - DOZZLE_REMOTE_AGENT=10.118.0.4:7007,10.118.0.2:7007
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    ports:
      - 8085:8080
volumes:
  dozzle-data:
    external: true


DOZZLE_AUTH_PROVIDER=simple enables basic authentication.
DOZZLE_REMOTE_AGENT specifies the IP addresses and ports of the Dozzle agents (e.g., 10.118.0.4:7007 and 10.118.0.2:7007).
The ./data:/data volume maps a local directory to persist authentication data.


Create the Data DirectoryCreate the ./data directory in the same location as docker-compose.yml:
mkdir data


Deploy the ServerNavigate to the directory containing docker-compose.yml and run:
docker-compose up -d

This starts the Dozzle server, accessible at http://<server-ip>:8085.


Step 3: Generate Admin Password
To secure access to the Dozzle web interface, generate an admin user with a password.

Run the Password Generation CommandExecute the following command to generate an admin user:
docker run --rm amir20/dozzle generate --name Admin --email me@email.net --password secret admin


Replace Admin with the desired username.
Replace me@email.net with the admin email.
Replace secret with a secure password.
The admin argument specifies the role.

This command generates a configuration file in the ./data directory, which is used by the Dozzle server for authentication.

Verify AuthenticationAccess the Dozzle web interface at http://<server-ip>:8085 and log in using the credentials specified in the generation command.


Step 4: (Optional) Nginx Reverse Proxy Configuration
To securely expose the Dozzle service over HTTPS, configure an Nginx reverse proxy with SSL.

Create Nginx ConfigurationCreate an Nginx configuration file (e.g., /etc/nginx/sites-available/dozzle) with the following content:
server {
    listen 80;
    server_name dlogs.inovativai.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name dlogs.inovativai.com;

    ssl_certificate /etc/letsencrypt/live/dlogs.inovativai.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/dlogs.inovativai.com/privkey.pem;
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


Replace dlogs.inovativai.com with your domain.
Ensure the SSL certificate paths (/etc/letsencrypt/...) match your Let’s Encrypt setup.
The commented headers (Remote-User, Remote-Email, Remote-Name) are optional and used for forward-proxy authentication if required by Dozzle.


Enable the Nginx ConfigurationCreate a symbolic link to enable the configuration:
sudo ln -s /etc/nginx/sites-available/dozzle /etc/nginx/sites-enabled/


Test and Reload NginxTest the Nginx configuration for errors:
sudo nginx -t

If the test is successful, reload Nginx:
sudo systemctl reload nginx


Access Dozzle via HTTPSAccess the Dozzle interface at https://dlogs.inovativai.com and log in with the admin credentials.


Step 5: Access Dozzle

Open a web browser and navigate to http://<server-ip>:8085 (or https://<your-domain> if using Nginx).
Log in with the admin credentials.
You should now see the logs from the containers managed by the connected Dozzle agents.

Troubleshooting

Agent Not Connecting: Ensure the agent is running and the specified IP addresses and ports in DOZZLE_REMOTE_AGENT are correct and accessible.
Authentication Issues: Verify that the ./data directory contains the generated authentication file and that the Dozzle server has read access to it.
Port Conflicts: Ensure ports 8085 (server) and 7007 (agent) are not used by other services.
Nginx Errors: Check Nginx logs (/var/log/nginx/error.log) for issues related to SSL or proxy configuration.

Notes

The restart: unless-stopped policy ensures the agent and server restart automatically unless explicitly stopped.
The healthcheck configuration in the agent ensures the container is healthy and responsive.
Use strong passwords for production environments to secure the Dozzle interface.
Ensure your SSL certificates are up-to-date if using the Nginx reverse proxy.

