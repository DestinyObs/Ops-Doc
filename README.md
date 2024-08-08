### Tunneling Service Documentation

---

## Project Overview

### Objective:
Develop an alternative tunneling service to `ngrok` and `serveo.net` that utilizes SSH reverse forwarding to provide dynamic public URLs for accessing local applications.

### Key Features:
- **SSH Reverse Forwarding:** Allow dynamic port forwarding using SSH reverse tunneling.
- **Proxy Management:** Properly manage HTTP/HTTPS proxy settings to forward traffic on port 80 and 443.
- **Wildcard Domains:** Support a range of subdomains for tunneling, providing flexibility for users.
- **Automatic Port Management:** Dynamically allocate available ports for reverse forwarding.

---

## How the Tunnel Service Works

### 1. Initialization and Setup

The script begins by defining the domain name and log file location. It also sets up a base port for dynamic port allocation:

```bash
#!/bin/bash

DOMAIN="mobyme.site"
LOG_FILE="/home/tunnel/tunnel_debug.log"
PORTS_FILE="/home/tunnel/used_ports.txt"
BASE_PORT=10000

echo "$(date): Script started" >> "$LOG_FILE"
```

### 2. Generating Random Subdomains

The script generates a random subdomain for each tunnel session:

```bash
function generate_subdomain() {
    cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 8 | head -n 1
}
```

### 3. Logging and Port Management

Logging is handled by a simple function that appends log entries to the log file:

```bash
function log() {
    echo "$(date): $1" >> "$LOG_FILE"
}
```

Dynamic port allocation is done through a function that finds an available port and marks it as used:

```bash
function get_available_port() {
    while true; do
        PORT=$((BASE_PORT + RANDOM % 55535))
        if ! grep -q "^$PORT$" "$PORTS_FILE" 2>/dev/null; then
            echo "$PORT" >> "$PORTS_FILE"
            echo "$PORT"
            return
        fi
    done
}
```

Ports are released when no longer in use:

```bash
function remove_port() {
    local port=$1
    sed -i "/^$port$/d" "$PORTS_FILE"
}
```

### 4. Handling Connections

This function sets up the tunnel by generating a random subdomain and allocating a port, then it configures iptables and updates Nginx:

```bash
function handle_connection() {
    local subdomain=$(generate_subdomain)
    local local_port=$1
    local remote_port=$(get_available_port)

    log "Handling connection: $subdomain.$DOMAIN -> localhost:$local_port (Remote port: $remote_port)"
    echo "Tunnel established: https://$subdomain.$DOMAIN" | tee /home/tunnel/tunnel_url.txt

    # Set up iptables rule for this specific subdomain
    sudo iptables -t nat -A PREROUTING -p tcp -d "$subdomain.$DOMAIN" --dport 80 -j REDIRECT --to-port $remote_port
    sudo iptables -t nat -A PREROUTING -p tcp -d "$subdomain.$DOMAIN" --dport 443 -j REDIRECT --to-port $remote_port

    # Update Nginx configuration
    sudo tee /etc/nginx/sites-available/$subdomain.conf > /dev/null <<EOF
server {
    listen 80;
    server_name $subdomain.$DOMAIN;
    return 301 https://\$host\$request_uri;
}

server {
    listen 443 ssl;
    server_name $subdomain.$DOMAIN;

    ssl_certificate /etc/letsencrypt/live/mobyme.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mobyme.site/privkey.pem;

    location / {
        proxy_pass http://localhost:$remote_port;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

    sudo ln -s /etc/nginx/sites-available/$subdomain.conf /etc/nginx/sites-enabled/
    sudo nginx -s reload

    log "Tunnel active. Press Ctrl+C to exit."
    # Keep the script running
    while true; do
        sleep 10
    done

    # Cleanup
    sudo rm /etc/nginx/sites-enabled/$subdomain.conf /etc/nginx/sites-available/$subdomain.conf
    sudo nginx -s reload
    sudo iptables -t nat -D PREROUTING -p tcp -d "$subdomain.$DOMAIN" --dport 80 -j REDIRECT --to-port $remote_port
    sudo iptables -t nat -D PREROUTING -p tcp -d "$subdomain.$DOMAIN" --dport 443 -j REDIRECT --to-port $remote_port
    remove_port $remote_port
}
```

### 5. Running the Script

The main script calls the `handle_connection` function with the desired port:

```bash
log "Script running. Waiting for connection."
handle_connection 3000
log "Script ended"
```

---

## Nginx Configuration

The Nginx server is configured to handle both HTTP and HTTPS requests. It redirects HTTP traffic to HTTPS and proxies HTTPS traffic to the appropriate local port:

```conf

server {
    listen 80;
    server_name *.mobyme.site;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    server_name *.mobyme.site;

    ssl_certificate /etc/letsencrypt/live/mobyme.site/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mobyme.site/privkey.pem;

    location / {
        return 404;
    }
}

# Individual subdomain configurations will be included here
include /etc/nginx/sites-enabled/*.conf;
```

---

## Usage

To use the tunneling service, run the following SSH command:

```bash
ssh -R 8080:localhost:3000 tunnel@mobyme.site
```

This command reverse forwards your local application running on port 3000 to a dynamically generated subdomain on `mobyme.site`, allowing public access.

---

## Conclusion

This tunneling service provides an efficient alternative to existing solutions like `ngrok` and `serveo.net`. It leverages SSH reverse forwarding, dynamic subdomain allocation, and Nginx proxy management to create secure, publicly accessible URLs for local applications.

---
