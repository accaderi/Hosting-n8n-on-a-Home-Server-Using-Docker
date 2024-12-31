<h1 align="center"><strong>Hosting n8n on a Home Server Using Docker</strong></h1>

<!-- <p align="center">
  <img src="screenshot.jpg" alt="Screenshot">
</p> -->

<p align="center">
  <a href="https://youtu.be/8FJhilTUhxQ">
    <img src="https://img.youtube.com/vi/8FJhilTUhxQ/0.jpg" alt="Youtube Video">
  </a>
</p>

<p align="center">
  <a href="https://youtu.be/8FJhilTUhxQ">Hosting n8n on Home Server</a>
</p>
This guide outlines the steps to host n8n on a home server using Docker. It includes prerequisites like setting up DNS and port forwarding, installing and configuring Docker, securing the server with UFW firewall rules, and setting up Nginx as a reverse proxy. It also covers obtaining an SSL certificate with Certbot for HTTPS support, ensuring secure access to your self-hosted n8n instance.

## Prerequisites

- Installed Linux on the server machine.
- In your domain provider, add a DNS record:
  - **Type**: A
  - **Domain**: your-domain.com
  - **IP**: your router's static IP (WAN IP)

## 1. Update the Package Index

```bash
sudo apt update
```

## 2. Install Docker
```bash
sudo apt install docker.io
```

## 3. Start Docker
```bash
sudo systemctl start docker
```

## 4. Enable Docker to Start at Boot
```bash
sudo systemctl enable docker
```

## 5. Set Up Port Forwarding and Firewall Rules

### Port Forwarding Configuration for ASUS AX5400 Router
1. Go to **WAN > Virtual Server / Port Forwarding** tab in your router settings
2. Enable port forwarding:
   - **Service Name**: Your preferred name
   - **Protocol**: TCP
   - **External Port**: 443 (use 443 for HTTPS, 80 for HTTP, or 5678 for n8n directly with the built-in Apache2 of Debian)
   - **Internal Port**: Same as the external port (443, 80, or 5678)
   - **Internal IP Address**: The internal IP of your home server (use ifconfig to check)
   - **Source IP**: Empty

### Set Up Firewall Rules
To configure your firewall to allow only traffic on port 443 (HTTPS) using UFW (Uncomplicated Firewall) on a Debian server:

#### Initial Setup
Deny all incoming connections by default:

Deny All Incoming Connections (if not already set): This ensures that all incoming connections are blocked by default, except those explicitly allowed.

Note: If you're using SSH (usually on port 22), make sure to allow this port as well or skip this step if you only need access through the port defined in the previous step.

```bash
sudo ufw default deny incoming
```
Allow Outgoing Connections (if not already set): Allows all outgoing connections (usually the default setting).

```bash
sudo ufw default allow outgoing
```
Allow HTTPS Traffic on Port 443: Allow incoming traffic on port 443 for HTTPS.

```bash
sudo ufw allow 443/tcp
```
Remove Existing Rules for Port 80 (if necessary): If you previously allowed port 80 (HTTP) and want to block it, you can remove that rule:

```bash
sudo ufw delete allow 80/tcp
```
Enable UFW (if it’s not already enabled):

```bash
sudo ufw enable
```
Check UFW Status: Verify the firewall rules with:

```bash
sudo ufw status verbose
```
Ensure that only port 443 is allowed.

Remove Other Incoming Rules (if needed): If other ports like 5678 are still allowed, you can remove them:

```bash
sudo ufw delete allow 5678/tcp
```

## 6. Run the Docker Container
To run n8n with Docker, use the following command:

```bash
sudo docker run -d --restart unless-stopped -it \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="your-domain.com" \
  -e WEBHOOK_TUNNEL_URL="https://your-domain.com/" \
  -e WEBHOOK_URL="https://your-domain.com/" \
  -v ~/.n8n:/root/.n8n \
  n8nio/n8n
```

## 7. Install Nginx
```bash
sudo apt install nginx
```

## 8. Create a New Nginx Configuration File
```bash
sudo nano /etc/nginx/sites-available/n8n.conf
```
Add the following configuration for Nginx:

**Important:** If you're using HTTPS, the port forwarding must be set to 443 for HTTPS, 80 for HTTP, or 5678 for n8n using the built-in Apache2 server in Debian.

```nginx
server {
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

## 9. Enable the Configuration
Create a symbolic link to enable the configuration:

```bash
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
```

## 10. Test the Nginx Configuration and Restart
Test the Nginx configuration for errors:

```bash
sudo nginx -t
```
Restart Nginx to apply the changes:

```bash
sudo systemctl restart nginx
```
If you encounter an error related to missing configuration files, follow these steps to remove existing configuration files:

```bash
sudo rm -rf /etc/nginx/sites-available/*
sudo rm -rf /etc/nginx/sites-enabled/*
```
Then, repeat the Nginx configuration steps.

### If Port 80 is Already in Use (by Apache2)
If you receive an error indicating that port 80 is already in use, check if Apache2 is using the port:

```bash
sudo netstat -tulpn | grep ':80'
```
If Apache2 is running, stop it:

```bash
sudo systemctl stop apache2
```
## 11. Install Certbot and the Nginx Plugin
Install Certbot and the necessary Nginx plugin:

```bash
sudo apt install certbot python3-certbot-nginx
```
12. Obtain an SSL Certificate
To obtain an SSL certificate for your domain:

```bash
sudo certbot --nginx -d your-domain.com
```
To check the certificate, you can use [SSL Labs](https://www.ssllabs.com/)