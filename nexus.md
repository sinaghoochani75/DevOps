# Sonatype Nexus Repository with Docker Compose and Nginx

## Overview

This project describes how to deploy Sonatype Nexus Repository on Ubuntu Server using Docker Compose and Nginx as an HTTPS reverse proxy.

The deployment provides:

- Containerized Nexus Repository.
- Nginx reverse proxy.
- HTTPS access using TLS certificates.
- Persistent storage for Nexus data.
- HTTP-to-HTTPS redirection.
- Docker networking between Nexus and Nginx.
- Basic deployment verification and troubleshooting commands.
- Backup and recovery procedures.

## Architecture

```text
                         Client
                           |
                           |
                 nexus.example.test
                           |
                    TCP 80 / 443
                           |
                 +------------------+
                 |  Nginx Container |
                 |  Reverse Proxy   |
                 |  TLS Termination|
                 +------------------+
                           |
                           |
                  Docker Bridge Network
                           |
                           |
                 +------------------+
                 | Nexus Container  |
                 | Port 8081        |
                 +------------------+
                           |
                           |
                       /nexus-data
                           |
                           |
                 Host Persistent Storage
                    /opt/nexus/data
```

### Architecture Components

| Component | Description |
|---|---|
| Nexus Repository | Stores and manages software artifacts and packages. |
| Docker Compose | Defines and manages the application containers. |
| Nginx | Provides HTTPS termination and reverse proxy functionality. |
| Docker Bridge Network | Enables communication between the Nexus and Nginx containers. |
| TLS Certificate | Encrypts client connections to Nginx. |
| Persistent Volume | Stores Nexus application data on the host filesystem. |

## 1. Prerequisites

The following components are required:

- Ubuntu Server.
- Docker Engine.
- Docker Compose plugin.
- OpenSSL.
- Sufficient memory and disk space.
- Administrative privileges.

Verify Docker installation:

```bash
docker --version
docker compose version
```

Check system resources:

```bash
free -h
df -h /opt
```

## 2. Configuration Variables

The following values are used throughout this documentation.

Replace them with the appropriate values for your environment before deployment.

| Parameter | Example |
|---|---|
| Server IP | `192.0.2.10` |
| Hostname | `nexus.example.test` |
| Installation Directory | `/opt/nexus` |
| Nexus Data Directory | `/opt/nexus/data` |
| Nginx Configuration | `/opt/nexus/nginx/nexus.conf` |
| TLS Certificate | `/opt/nexus/certs/nexus.example.test.crt` |
| TLS Private Key | `/opt/nexus/certs/nexus.key` |
| Nexus Internal Port | `8081` |
| HTTP Port | `80` |
| HTTPS Port | `443` |

> Note: `192.0.2.10` belongs to a reserved documentation network. It is an example address and must be replaced with a real server IP before deployment.

## 3. Create the Directory Structure

Create the required directories:

```bash
sudo mkdir -p /opt/nexus/{data,nginx,certs}
```

The directory structure will be:

```text
/opt/nexus/
├── data/
├── docker-compose.yml
├── nginx/
│   └── nexus.conf
└── certs/
    ├── nexus.key
    └── nexus.example.test.crt
```

### Directory Description

- `/opt/nexus/data`: Persistent Nexus application data.
- `/opt/nexus/nginx`: Nginx configuration files.
- `/opt/nexus/certs`: TLS certificates and private keys.
- `/opt/nexus/docker-compose.yml`: Docker Compose configuration.

## 4. Configure Nexus Data Permissions

The Nexus container commonly uses UID `200` to access its data directory.

Set ownership:

```bash
sudo chown -R 200:200 /opt/nexus/data
```

The directory will be mounted into the container using:

```text
/opt/nexus/data:/nexus-data
```

This bind mount ensures that Nexus data remains on the host filesystem when the container is recreated.

> Important: Verify the UID and directory permissions for the selected Nexus image before changing ownership in an existing deployment.

## 5. Generate a Self-Signed TLS Certificate

For testing purposes, a self-signed certificate can be generated using OpenSSL.

Navigate to the certificate directory:

```bash
cd /opt/nexus/certs
```

Generate a private key:

```bash
sudo openssl genrsa -out nexus.key 2048
```

Restrict private-key permissions:

```bash
sudo chmod 600 nexus.key
```

Generate a self-signed certificate:

```bash
sudo openssl req -new -x509 \
  -key nexus.key \
  -out nexus.example.test.crt \
  -days 365 \
  -subj "/CN=nexus.example.test" \
  -addext "subjectAltName=DNS:nexus.example.test,IP:192.0.2.10"
```

Set certificate permissions:

```bash
sudo chmod 644 nexus.example.test.crt
```

Verify the certificate:

```bash
openssl x509 \
  -in /opt/nexus/certs/nexus.example.test.crt \
  -noout -subject -ext subjectAltName
```

List the certificate files:

```bash
ls -l /opt/nexus/certs/
```

### Certificate Notes

- The private key must be protected.
- The certificate must contain the hostname or IP address used by clients.
- Self-signed certificates are not automatically trusted by browsers.
- Production environments should use certificates issued by a trusted certificate authority.

## 6. Configure Nginx

Create the Nginx configuration file:

```bash
sudo nano /opt/nexus/nginx/nexus.conf
```

Add the following configuration:

```nginx
server {
    listen 80 default_server;
    server_name nexus.example.test;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl default_server;
    http2 on;

    server_name nexus.example.test;

    ssl_certificate /etc/nginx/certs/nexus.example.test.crt;
    ssl_certificate_key /etc/nginx/certs/nexus.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 2g;

    proxy_connect_timeout 60s;
    proxy_send_timeout 300s;
    proxy_read_timeout 300s;

    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_pass http://nexus:8081;
        proxy_http_version 1.1;

        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host $http_host;
        proxy_set_header X-Forwarded-Port 443;
    }
}
```

### Nginx Configuration Explanation

**HTTP to HTTPS Redirection**

The first server block listens on port `80` and redirects HTTP requests to HTTPS.

```nginx
return 301 https://$host$request_uri;
```

**TLS Configuration**

The following directives configure the certificate and private key:

```nginx
ssl_certificate /etc/nginx/certs/nexus.example.test.crt;
ssl_certificate_key /etc/nginx/certs/nexus.key;
```

**Reverse Proxy**

Nginx forwards requests to the Nexus container:

```nginx
proxy_pass http://nexus:8081;
```

The hostname `nexus` refers to the Nexus service on the shared Docker network.

**Forwarded Headers**

The `X-Forwarded-*` headers provide information about the original client connection.

For example:

```nginx
proxy_set_header X-Forwarded-Proto https;
proxy_set_header X-Forwarded-Port 443;
```

These headers inform the upstream application that the original client connection used HTTPS.

**Upload Size and Timeouts**

```nginx
client_max_body_size 2g;
proxy_connect_timeout 60s;
proxy_send_timeout 300s;
proxy_read_timeout 300s;
```

These settings accommodate large artifact uploads and longer-running repository operations at the Nginx layer.

Actual upload limits may also depend on Nexus and repository configuration.

## 7. Create the Docker Compose File

Create the Compose configuration:

```bash
sudo nano /opt/nexus/docker-compose.yml
```

Add the following content:

```yaml
services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped

    expose:
      - "8081"

    volumes:
      - /opt/nexus/data:/nexus-data

    environment:
      INSTALL4J_ADD_VM_PARAMS: >-
        -Xms1200m
        -Xmx1200m
        -XX:MaxDirectMemorySize=1200m
        -Djava.util.prefs.userRoot=/nexus-data/javaprefs

    networks:
      - nexus-network

    stop_grace_period: 120s

  nginx:
    image: nginx:alpine
    container_name: nexus-nginx
    restart: unless-stopped

    depends_on:
      - nexus

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - /opt/nexus/nginx/nexus.conf:/etc/nginx/conf.d/default.conf:ro
      - /opt/nexus/certs:/etc/nginx/certs:ro

    networks:
      - nexus-network

networks:
  nexus-network:
    driver: bridge
```

### Docker Compose Configuration Explanation

**Nexus Service**

The Nexus service runs the repository manager.

```yaml
image: sonatype/nexus3:latest
```

For repeatable production deployments, consider pinning the image to a tested version or immutable digest instead of relying on `latest`.

**Internal Port**

```yaml
expose:
  - "8081"
```

This makes the Nexus application port available to services on the Docker network without publishing it directly on the host.

**Persistent Storage**

```yaml
volumes:
  - /opt/nexus/data:/nexus-data
```

This maps the host data directory to the Nexus data directory inside the container.

**Memory Configuration**

```yaml
environment:
  INSTALL4J_ADD_VM_PARAMS: >-
    -Xms1200m
    -Xmx1200m
    -XX:MaxDirectMemorySize=1200m
    -Djava.util.prefs.userRoot=/nexus-data/javaprefs
```

These JVM parameters configure the initial and maximum Java heap size, the maximum direct memory allocation, and the Java preferences directory.

Memory requirements should be adjusted according to the workload and available system resources.

**Nginx Ports**

```yaml
ports:
  - "80:80"
  - "443:443"
```

These mappings publish HTTP and HTTPS on the Docker host.

**Read-Only Configuration Mounts**

```yaml
volumes:
  - /opt/nexus/nginx/nexus.conf:/etc/nginx/conf.d/default.conf:ro
  - /opt/nexus/certs:/etc/nginx/certs:ro
```

The `:ro` suffix mounts the configuration and certificate directory as read-only inside the container.

**Docker Network**

Both services use the same bridge network:

```yaml
networks:
  - nexus-network
```

This allows Nginx to reach Nexus using the service name `nexus`.

## 8. Validate the Configuration

Navigate to the installation directory:

```bash
cd /opt/nexus
```

Validate the Compose file:

```bash
sudo docker compose config
```

If the configuration is valid, Docker Compose displays the resolved configuration without reporting a YAML or Compose configuration error.

## 9. Start the Services

Start the containers:

```bash
sudo docker compose up -d
```

Check container status:

```bash
sudo docker compose ps
```

Expected services:

```text
nexus
nexus-nginx
```

Both containers should be running.

Check published ports:

```bash
sudo ss -ntlp
```

Inspect the Nexus logs:

```bash
sudo docker compose logs --tail=100 nexus
```

Inspect the Nginx logs:

```bash
sudo docker compose logs --tail=100 nginx
```

Follow Nexus logs in real time:

```bash
sudo docker compose logs -f nexus
```

Press `Ctrl+C` to stop following the logs. This does not stop the container.

## 10. Configure Client Name Resolution

Clients must resolve the Nexus hostname to the server IP address.

For a temporary test, add an entry to the client's hosts file.

### Linux

Edit the hosts file:

```bash
sudo nano /etc/hosts
```

Add:

```text
192.0.2.10 nexus.example.test
```

### Windows

Open the following file with administrator privileges:

```text
C:\Windows\System32\drivers\etc\hosts
```

Add:

```text
192.0.2.10 nexus.example.test
```

Replace the example IP address with the actual server IP.

For production environments, configure an appropriate DNS record instead of maintaining hosts-file entries on individual clients.

## 11. Verify HTTPS Access

Test the HTTPS endpoint:

```bash
curl -k -I --connect-timeout 5 https://192.0.2.10/
```

Test the hostname:

```bash
curl -k -I --connect-timeout 5 https://nexus.example.test/
```

The `-k` option allows curl to connect without validating the certificate's trust chain. It is useful for testing self-signed certificates but should not be used as a permanent security workaround.

Inspect the response headers:

```bash
curl -k -I https://nexus.example.test/
```

Verify the TLS certificate:

```bash
openssl s_client \
  -connect nexus.example.test:443 \
  -servername nexus.example.test
```

Open the following URL in a browser:

```text
https://nexus.example.test/
```

A browser warning may appear because the certificate is self-signed.

## 12. Initial Nexus Login

The default administrative username is:

```text
admin
```

Retrieve the initial password from the persistent data directory:

```bash
sudo cat /opt/nexus/data/admin.password
```

Alternatively, retrieve it from the running container:

```bash
sudo docker exec nexus cat /nexus-data/admin.password
```

Use the initial password to log in and complete the initial setup process.

Follow the setup wizard to configure the administrator password and other requested settings.

> Security note: Do not publish the initial password, administrator credentials, or other secrets in a public Git repository.

## 13. Useful Docker Compose Commands

Run commands from the installation directory:

```bash
cd /opt/nexus
```

### View Service Status

```bash
sudo docker compose ps
```

### Start Services

```bash
sudo docker compose start
```

### Stop Services

```bash
sudo docker compose stop
```

### Restart Services

```bash
sudo docker compose restart
```

### Restart Only Nginx

```bash
sudo docker compose restart nginx
```

### Restart Only Nexus

```bash
sudo docker compose restart nexus
```

### View Logs

```bash
sudo docker compose logs --tail=100
```

### Follow All Logs

```bash
sudo docker compose logs -f
```

### Stop and Remove Containers

```bash
sudo docker compose down
```

This removes the Compose-managed containers and network but does not delete the bind-mounted Nexus data directory.

Do not use `docker compose down -v` without first understanding the volume configuration and the consequences for persistent data.

### Apply Compose Configuration Changes

After editing the Compose file:

```bash
sudo docker compose config
sudo docker compose up -d
```

### Apply Nginx Configuration Changes

After editing the Nginx configuration:

```bash
sudo docker exec nexus-nginx nginx -t
sudo docker compose restart nginx
```

The `nginx -t` command validates the configuration currently mounted inside the running container. If the container has not yet been updated or recreated after a file change, ensure the latest configuration is available before relying on the test result.

## 14. Troubleshooting

### Check Container Status

```bash
sudo docker compose ps
```

### Check Nexus Logs

```bash
sudo docker compose logs --tail=200 nexus
```

### Check Nginx Logs

```bash
sudo docker compose logs --tail=200 nginx
```

### Check Nginx Configuration

```bash
sudo docker exec nexus-nginx nginx -t
```

### Check the Docker Network

```bash
sudo docker network ls
```

Inspect the Compose network:

```bash
sudo docker network inspect nexus_nexus-network
```

The network name may vary if the Compose project name has been customized.

### Check Host Ports

```bash
sudo ss -ntlp
```

Confirm that ports `80` and `443` are listening.

### Test HTTPS Locally

```bash
curl -k -I --connect-timeout 5 https://127.0.0.1/
```

### Common Problems

| Problem | Possible Cause | Suggested Check |
|---|---|---|
| Nexus container repeatedly restarts | Memory pressure, startup failure, or permissions | Inspect Nexus logs and system memory. |
| Nginx returns `502 Bad Gateway` | Nexus is not ready or cannot be reached | Check Nexus status and Nginx logs. |
| Browser reports a certificate warning | Self-signed or untrusted certificate | Verify the certificate and configure client trust. |
| Hostname does not resolve | Missing DNS or hosts-file entry | Check name resolution on the client. |
| HTTPS connection fails | Port conflict, firewall, or TLS configuration issue | Check listening ports and Nginx logs. |
| Upload fails for large artifacts | Request-size limit or repository configuration | Check Nginx and Nexus upload settings. |
| Nexus data is inaccessible | Incorrect ownership or filesystem permissions | Check `/opt/nexus/data` permissions. |

## 15. Backup and Recovery

Nexus stores its application data under:

```text
/opt/nexus/data
```

A backup should include the complete data directory and any additional configuration required to recreate the deployment.

### Create a Consistent Offline Backup

For a simple offline backup, stop the services before archiving the data.

```bash
cd /opt/nexus
sudo docker compose stop
```

Create a timestamped archive:

```bash
sudo tar -czf \
  "/opt/nexus/nexus-data-backup-$(date +%F-%H%M%S).tar.gz" \
  -C /opt/nexus data
```

Restart the services:

```bash
sudo docker compose start
```

List the resulting archive:

```bash
ls -lh /opt/nexus/nexus-data-backup-*.tar.gz
```

### Backup Considerations

- Keep backups outside the Nexus data directory.
- Store backup copies on separate storage.
- Protect backup files because they may contain sensitive repository data and configuration.
- Test restoration procedures before relying on a backup.
- Include the Compose file, Nginx configuration, and certificate-management information in the disaster-recovery plan.
- Protect private keys and secrets separately.
- Ensure the backup process has enough free disk space.

For production environments, use a documented backup and recovery strategy that accounts for the Nexus version, repository format, and storage configuration.

## 16. Security Recommendations

For production deployments:

1. Use a trusted TLS certificate.
2. Protect the TLS private key.
3. Restrict access to administrative interfaces.
4. Use strong administrator credentials.
5. Keep Docker, the host OS, Nexus, and Nginx updated.
6. Pin container images to tested versions instead of relying on `latest`.
7. Monitor disk usage and available memory.
8. Back up repository data regularly.
9. Restrict unnecessary inbound network access.
10. Avoid publishing secrets, credentials, private keys, or sensitive configuration in Git repositories.
11. Configure DNS properly for the repository hostname.
12. Test certificate renewal and backup restoration procedures.

## 17. Deployment Summary

The deployment consists of two containers:

- `nexus`: The Nexus Repository application.
- `nexus-nginx`: The Nginx HTTPS reverse proxy.

Nginx publishes ports `80` and `443` on the host. Nexus listens on port `8081` inside the Docker network and is not directly published to the host by this Compose configuration.

Persistent Nexus data is stored at:

```text
/opt/nexus/data
```

The client accesses the repository through:

```text
https://nexus.example.test/
```

The final architecture separates the repository application from the reverse proxy and provides persistent storage, HTTPS access, and a manageable Docker Compose deployment.

---
