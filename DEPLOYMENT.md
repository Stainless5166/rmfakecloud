# rmfakecloud Enterprise Deployment Guide

**Version:** 0.0.25+
**Document Date:** January 2026
**License:** GNU AGPL v3.0

## Executive Summary

rmfakecloud is a self-hosted replacement for the reMarkable cloud service, enabling organizations to maintain complete control over document synchronization, storage, and backup for reMarkable tablets. This deployment guide provides verified features, security considerations, and best practices for deploying rmfakecloud on a corporate intranet for multiple devices.

## Table of Contents

1. [Verified Feature Set](#verified-feature-set)
2. [System Architecture](#system-architecture)
3. [Deployment Requirements](#deployment-requirements)
4. [Deployment Scenarios](#deployment-scenarios)
5. [Security Considerations](#security-considerations)
6. [Multi-Device Configuration](#multi-device-configuration)
7. [Backup and Data Management](#backup-and-data-management)
8. [Monitoring and Maintenance](#monitoring-and-maintenance)
9. [Troubleshooting](#troubleshooting)
10. [Known Limitations](#known-limitations)

---

## Verified Feature Set

### Supported Devices

| Device | Support Status | Notes |
|--------|---------------|-------|
| reMarkable 1 | ✅ Fully Supported | All features |
| reMarkable 2 | ✅ Fully Supported | All features |
| reMarkable Paper Pro | ✅ Fully Supported | All features |
| reMarkable Paper Pro Move | ✅ Fully Supported | All features |

**Software Compatibility:** Tested up to reMarkable software version 3.22.0

### Core Features (Verified)

#### ✅ File Synchronization
- **Status:** Production Ready
- **Protocols Supported:** Sync 1.0, 1.5, 2.0, 3.0, 4.0
- **Implementation:** Full bidirectional sync with conflict resolution
- **Verification:** Confirmed in `/internal/storage/` and sync version handling in middleware
- **Data Storage:** File-based storage with configurable data directory

#### ✅ Multi-User Support
- **Status:** Production Ready
- **Authentication:** JWT-based with configurable secret key
- **User Management:** CLI tools for user creation and profile management
- **Verification:** Confirmed in `/internal/model/user.go` and `/internal/cli/managerusers.go`
- **Isolation:** Per-user data directories with access control

#### ✅ Email Integration
- **Status:** Production Ready
- **Protocol:** SMTP with TLS/STARTTLS support
- **Features:**
  - Send documents as email attachments
  - Configurable SMTP server
  - Custom From/Reply-To headers
  - Plain auth support
- **Verification:** Confirmed in `/internal/email/smtp.go`

#### ✅ Handwriting Recognition (HWR)
- **Status:** Production Ready (requires external service)
- **Provider:** MyScript Cloud API
- **Requirements:**
  - MyScript developer account (free tier: 2,000 recognitions/month)
  - Application Key and HMAC
- **Features:**
  - Language override support
  - Configurable recognition locale
- **Verification:** Confirmed in `/internal/hwr/client.go`

#### ✅ Screen Sharing
- **Status:** Production Ready
- **Protocol:** MQTT over TLS (port 8883)
- **Requirements:**
  - TLS certificates (required)
  - TCP reverse proxy for MQTT
  - Optional: STUN/TURN servers for remote access
- **Network Scope:**
  - Local network: Works without ICE servers
  - Remote access: Requires ICE server configuration
- **Verification:** Confirmed in `/internal/mqtt/broker.go`

#### ✅ Storage Integrations
- **WebDAV:** Production Ready (Nextcloud, Owncloud, generic WebDAV)
- **FTP:** Production Ready
- **Local Filesystem:** Experimental (single-user only)
- **Verification:** Confirmed in `/internal/integrations/`

#### ⚠️ Partial Support
- **Dropbox:** Work in Progress (code exists but incomplete)
- **Google Drive:** Work in Progress (PR #241)

#### ❌ Not Supported
- **OneDrive:** Not implemented
- **Handwriting Search:** Not implemented
- **Document Rendering in Web UI:** Work in Progress (Issue #255)

### Web Interface Features

- User registration and authentication
- Document browsing (list view)
- Document upload/download
- Code generation for device pairing
- Integration management
- User profile editing

---

## System Architecture

### Components

```
┌─────────────────────────────────────────┐
│         reMarkable Tablets              │
│  (RM1, RM2, Paper Pro, Paper Pro Move)  │
└──────────────┬──────────────────────────┘
               │ HTTPS (Port 3000)
               │ MQTT/TLS (Port 8883)
               ▼
┌─────────────────────────────────────────┐
│      Reverse Proxy (Optional)           │
│    (nginx, Apache, Traefik)             │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│         rmfakecloud Server              │
│                                         │
│  ┌────────────────────────────────┐   │
│  │  API Server (Go)               │   │
│  │  - Port 3000 (HTTP/HTTPS)      │   │
│  │  - JWT Authentication          │   │
│  └────────────────────────────────┘   │
│                                         │
│  ┌────────────────────────────────┐   │
│  │  MQTT Broker                   │   │
│  │  - Port 8883 (TLS required)    │   │
│  └────────────────────────────────┘   │
│                                         │
│  ┌────────────────────────────────┐   │
│  │  Web UI (React)                │   │
│  │  - Embedded in binary          │   │
│  └────────────────────────────────┘   │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│      File System Storage                │
│      /data/                             │
│        ├── users/                       │
│        │   ├── user1/                   │
│        │   └── user2/                   │
│        └── blob storage/                │
└─────────────────────────────────────────┘
```

### Technology Stack

- **Backend:** Go (Golang)
- **Frontend:** Node.js + React + TypeScript
- **Web Framework:** Gin
- **Authentication:** JWT (golang-jwt/jwt)
- **MQTT:** Embedded broker with TLS
- **Storage:** File-based (configurable directory)
- **Build:** Docker multi-stage build (from scratch image)

---

## Deployment Requirements

### Server Requirements

#### Minimum Specifications
- **CPU:** 2 cores
- **RAM:** 2 GB
- **Storage:** 50 GB + (10-20 GB per user estimated)
- **OS:** Linux (Docker recommended)
- **Network:** Private network/VLAN recommended

#### Recommended Specifications (10-50 tablets)
- **CPU:** 4 cores
- **RAM:** 4-8 GB
- **Storage:** 500 GB SSD (with expansion capability)
- **OS:** Ubuntu 22.04 LTS / Debian 12 / RHEL 9
- **Network:** Dedicated VLAN with firewall rules

### Network Requirements

- **Inbound Ports:**
  - `3000/tcp` - HTTP/HTTPS API and Web UI
  - `8883/tcp` - MQTT over TLS (for screen sharing)

- **Outbound Ports (optional):**
  - `587/tcp` or `465/tcp` - SMTP for email
  - `443/tcp` - MyScript HWR API (if enabled)
  - `443/tcp` - STUN/TURN servers (if remote screen sharing needed)

- **DNS Requirements:**
  - Tablets must resolve `my.remarkable.com` to rmfakecloud server
  - Tablets must resolve `local.appspot.com` to rmfakecloud server
  - Multiple DNS overrides required (see tablet setup)

### Prerequisites

- Docker and Docker Compose (recommended) OR
- Go 1.21+ and Node.js 18+ (for source build)
- OpenSSL (for certificate generation)
- Reverse proxy (nginx/Apache/Traefik) for production deployments

---

## Deployment Scenarios

### Scenario 1: Internal Network Only (Recommended for Corporate)

**Use Case:** All tablets and users on corporate network

**Configuration:**
```yaml
# docker-compose.yml
version: "3"
services:
  rmfakecloud:
    image: ddvk/rmfakecloud:latest
    container_name: rmfakecloud
    restart: unless-stopped
    ports:
      - "3000:3000"
      - "8883:8883"
    environment:
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
      - STORAGE_URL=https://local.appspot.com
      - DATADIR=/data
      - LOGLEVEL=info
      - RM_HTTPS_COOKIE=true
      - RM_TRUST_PROXY=false
      - TLS_CERT=/certs/server.crt
      - TLS_KEY=/certs/server.key
      # Optional: Email
      - RM_SMTP_SERVER=smtp.company.internal:587
      - RM_SMTP_USERNAME=${SMTP_USER}
      - RM_SMTP_PASSWORD=${SMTP_PASS}
      - RM_SMTP_STARTTLS=true
      - RM_SMTP_FROM=remarkable@company.internal
    volumes:
      - ./data:/data
      - ./certs:/certs:ro
    networks:
      - remarkable_net

networks:
  remarkable_net:
    driver: bridge
```

**Environment File (`.env`):**
```bash
# Generate with: openssl rand -base64 48
JWT_SECRET_KEY=CHANGE_THIS_TO_A_RANDOM_64_CHARACTER_STRING

# SMTP Credentials
SMTP_USER=smtp_username
SMTP_PASS=smtp_password
```

**Security:**
- Place server on isolated VLAN
- Use internal DNS or `/etc/hosts` on tablets
- Generate self-signed CA and trust on all devices
- Enable firewall rules limiting access to internal network

### Scenario 2: With Reverse Proxy (Production Recommended)

**Use Case:** Multiple services, centralized SSL, better logging

**nginx Configuration:**
```nginx
# /etc/nginx/sites-available/rmfakecloud

# HTTP API and Web UI
upstream rmfakecloud_http {
    server 127.0.0.1:3000;
}

server {
    listen 443 ssl http2;
    server_name remarkable.company.internal;

    ssl_certificate /etc/ssl/certs/company-ca-signed.crt;
    ssl_certificate_key /etc/ssl/private/company-ca-signed.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Logging
    access_log /var/log/nginx/rmfakecloud-access.log;
    error_log /var/log/nginx/rmfakecloud-error.log;

    location / {
        proxy_pass http://rmfakecloud_http;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Increase timeouts for large file uploads
        proxy_connect_timeout 300;
        proxy_send_timeout 300;
        proxy_read_timeout 300;
        send_timeout 300;

        # Large file upload support
        client_max_body_size 100M;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name remarkable.company.internal;
    return 301 https://$server_name$request_uri;
}
```

**nginx Stream Configuration for MQTT:**
```nginx
# /etc/nginx/nginx.conf - add to main context
stream {
    upstream rmfakecloud_mqtt {
        server 127.0.0.1:8883;
    }

    server {
        listen 8883;
        proxy_pass rmfakecloud_mqtt;
        proxy_connect_timeout 5s;

        # Enable SSL passthrough
        ssl_preread on;
    }
}
```

**rmfakecloud Configuration:**
```bash
JWT_SECRET_KEY=<your-secret>
STORAGE_URL=https://remarkable.company.internal
RM_TRUST_PROXY=true
RM_HTTPS_COOKIE=true
```

### Scenario 3: High Availability Setup

**Use Case:** Critical infrastructure, requires redundancy

**Not Recommended:** rmfakecloud uses file-based storage which doesn't support clustering out-of-the-box.

**Workaround Options:**
1. **Active-Passive with Shared Storage:**
   - Mount shared NFS/CIFS volume for `/data`
   - Use keepalived or Pacemaker for failover
   - Risk: File locking issues

2. **Scheduled Failover:**
   - Primary server with regular backups to standby
   - Manual failover process
   - Use DNS or load balancer to switch traffic

3. **Multiple Instances (User Sharding):**
   - Separate rmfakecloud instance per department/group
   - Different subdomain/DNS per instance
   - Easier to manage and isolate

---

## Security Considerations

### Authentication and Authorization

#### JWT Secret Management
```bash
# Generate strong secret (64+ characters recommended)
openssl rand -base64 48

# Store in .env file (never commit to git)
echo "JWT_SECRET_KEY=$(openssl rand -base64 48)" >> .env
chmod 600 .env
```

**Important:** If JWT secret changes, all users must re-authenticate.

#### User Management

```bash
# Create admin user
docker exec -it rmfakecloud /rmfakecloud-docker -createuser admin admin@company.internal

# List users
docker exec -it rmfakecloud /rmfakecloud-docker -listusers

# Delete user
docker exec -it rmfakecloud /rmfakecloud-docker -deleteuser admin
```

**Recommendation:** Disable open registration in production:
```bash
# Do NOT set OPEN_REGISTRATION=true
# Create users manually via CLI only
```

### Network Security

#### Firewall Rules (iptables example)
```bash
# Allow only internal network to rmfakecloud
iptables -A INPUT -p tcp --dport 3000 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 3000 -j DROP

iptables -A INPUT -p tcp --dport 8883 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 8883 -j DROP
```

#### TLS/SSL Certificates

**Option 1: Internal CA (Recommended for Intranet)**
```bash
# 1. Generate CA
openssl req -x509 -new -nodes -keyout ca.key -sha256 -days 3650 \
  -out ca.crt -subj "/C=US/ST=State/L=City/O=Company/CN=Company Root CA"

# 2. Generate server key
openssl genrsa -out server.key 2048

# 3. Generate CSR
openssl req -new -key server.key -out server.csr \
  -subj "/C=US/ST=State/L=City/O=Company/CN=*.appspot.com"

# 4. Sign with CA
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt -days 365 -sha256

# 5. Copy to server
cp server.crt server.key /path/to/rmfakecloud/certs/
```

**Option 2: Let's Encrypt (if publicly accessible)**
- Use certbot with nginx/Apache
- Automatic renewal setup
- Requires public DNS and port 80/443 accessible

### Brute Force Protection (fail2ban)

**Installation:**
```bash
# Install fail2ban
apt-get install fail2ban

# Create filter
cat > /etc/fail2ban/filter.d/rmfakecloud.conf <<EOF
[Definition]
failregex = ^.*, login failed ip:\s+<ADDR>.*$
EOF

# Create jail
cat > /etc/fail2ban/jail.d/rmfakecloud.conf <<EOF
[rmfakecloud]
enabled   = true
filter    = rmfakecloud
action    = iptables-multiport[name=rmfakecloud, port="3000,8883"]
journalmatch = _SYSTEMD_UNIT=docker-rmfakecloud.service
maxretry  = 5
bantime   = 3600
findtime  = 600
EOF

# Enable and configure reverse proxy trust
# In rmfakecloud config:
RM_TRUST_PROXY=true

# Restart fail2ban
systemctl restart fail2ban
```

### Data Security

#### At Rest
- **File Permissions:**
  ```bash
  chown -R 1000:1000 /path/to/data
  chmod 700 /path/to/data
  ```

- **Disk Encryption:** Use LUKS or equivalent for data volume
  ```bash
  # Example: LUKS encrypted volume
  cryptsetup luksFormat /dev/sdb1
  cryptsetup luksOpen /dev/sdb1 rmfakecloud_data
  mkfs.ext4 /dev/mapper/rmfakecloud_data
  ```

#### In Transit
- **Always use HTTPS** for API/Web UI
- **Always use TLS** for MQTT (port 8883)
- **Enforce TLS 1.2+** in reverse proxy
- **Use internal CA** or valid certificates

### Compliance Considerations

- **AGPL License:** If you modify rmfakecloud, you must provide source code to users
- **Data Sovereignty:** All user data stored on your infrastructure
- **Audit Logging:** Enable detailed logging for compliance
  ```bash
  LOGLEVEL=debug
  RM_LOGFILE=/data/logs/rmfakecloud.log
  ```

---

## Multi-Device Configuration

### Tablet Setup Methods

#### Method 1: Automated Installer (Recommended)

**Prerequisites:**
- SSH access to tablets (default: `ssh root@10.11.99.1`)
- rmfakecloud-proxy installer script

**Steps:**
```bash
# 1. Download installer
wget https://github.com/ddvk/rmfakecloud-proxy/releases/latest/download/installer-rm12.sh
# For Paper Pro:
# wget https://github.com/ddvk/rmfakecloud-proxy/releases/latest/download/installer-rmpro.sh

# 2. Copy to tablet
scp installer-rm12.sh root@10.11.99.1:

# 3. SSH to tablet
ssh root@10.11.99.1

# 4. Install
chmod +x installer-rm12.sh
./installer-rm12.sh install

# 5. Follow prompts:
# - Enter rmfakecloud URL: https://remarkable.company.internal
# - Certificate paths: (leave empty if using DNS method)
```

#### Method 2: toltec Package Manager

```bash
# On tablet with toltec installed:
opkg install rmfakecloud-proxy
rmfakecloudctl set-upstream https://remarkable.company.internal
rmfakecloudctl enable
```

#### Method 3: Manual Configuration

See tablet setup documentation for advanced manual configuration.

### DNS Configuration Options

#### Option A: Router/DNS Server (Easiest for multiple devices)

Add A records to internal DNS:
```
my.remarkable.com                                               IN A 10.0.10.50
local.appspot.com                                              IN A 10.0.10.50
hwr-production-dot-remarkable-production.appspot.com           IN A 10.0.10.50
service-manager-production-dot-remarkable-production.appspot.com IN A 10.0.10.50
ping.remarkable.com                                            IN A 10.0.10.50
internal.cloud.remarkable.com                                  IN A 10.0.10.50
```

**Advantages:**
- One-time setup
- All tablets automatically configured
- Easy to update

**Disadvantages:**
- Tablets cannot use official cloud
- Requires DNS infrastructure control

#### Option B: Per-Tablet /etc/hosts

Edit `/etc/hosts` on each tablet:
```bash
# On tablet
cat >> /etc/hosts <<EOF
10.0.10.50 my.remarkable.com
10.0.10.50 local.appspot.com
10.0.10.50 hwr-production-dot-remarkable-production.appspot.com
10.0.10.50 service-manager-production-dot-remarkable-production.appspot.com
10.0.10.50 ping.remarkable.com
10.0.10.50 internal.cloud.remarkable.com
10.0.10.50 eu.tectonic.remarkable.com
10.0.10.50 backtrace-proxy.cloud.remarkable.engineering
10.0.10.50 dev.ping.remarkable.com
10.0.10.50 dev.tectonic.remarkable.com
10.0.10.50 dev.internal.cloud.remarkable.com
10.0.10.50 eu.internal.tctn.cloud.remarkable.com
EOF
```

**Note:** Changes are lost after system updates.

### Certificate Distribution

#### For Self-Signed CA:

1. **Install CA on tablets:**
   ```bash
   # Copy CA cert to tablet
   scp ca.crt root@10.11.99.1:/usr/local/share/ca-certificates/

   # SSH to tablet
   ssh root@10.11.99.1

   # Update certificates
   update-ca-certificates

   # Verify (should show "1 added")
   ```

2. **Install CA on desktop clients (Windows/Mac):**
   - Windows: Import to "Trusted Root Certification Authorities"
   - Mac: Add to Keychain and trust
   - Linux: Copy to `/usr/local/share/ca-certificates/` and run `update-ca-certificates`

### User Enrollment Process

1. **Admin creates user account:**
   ```bash
   docker exec -it rmfakecloud /rmfakecloud-docker -createuser john.doe john.doe@company.internal
   ```

2. **Admin generates one-time code:**
   - Navigate to https://remarkable.company.internal
   - Login as admin
   - Click "Code" → "Generate Code"
   - Note the 8-character code

3. **User enrolls tablet:**
   - On tablet: Menu → General → Account → Setup Account
   - Enter the 8-character code
   - Tablet syncs and is ready

4. **Verify sync:**
   - Menu → Storage → Check Sync
   - Should show "Up to date"

### Bulk Deployment Script

```bash
#!/bin/bash
# bulk-deploy.sh - Deploy rmfakecloud-proxy to multiple tablets

TABLETS=(
  "10.11.99.1"
  "10.11.99.2"
  "10.11.99.3"
)
RMFAKECLOUD_URL="https://remarkable.company.internal"
INSTALLER="installer-rm12.sh"

for tablet in "${TABLETS[@]}"; do
  echo "Deploying to $tablet..."

  # Copy installer
  scp "$INSTALLER" "root@$tablet:/" || continue

  # Install (non-interactive mode - modify installer script for this)
  ssh "root@$tablet" "chmod +x /$INSTALLER && /$INSTALLER install --url $RMFAKECLOUD_URL --auto"

  echo "Deployed to $tablet"
done

echo "Deployment complete!"
```

---

## Backup and Data Management

### Data Directory Structure

```
data/
├── <user-id>/
│   ├── .userprofile           # User settings (YAML)
│   ├── .metadata              # User metadata
│   ├── <document-id>/         # Document directories
│   │   ├── <document-id>.content
│   │   ├── <document-id>.metadata
│   │   ├── <document-id>.pdf
│   │   └── ...
│   └── blob/                  # File storage
└── system/                    # System data
```

### Backup Strategies

#### Strategy 1: File-Level Backup (Recommended)

```bash
#!/bin/bash
# backup-rmfakecloud.sh

BACKUP_DIR="/backups/rmfakecloud"
DATA_DIR="/path/to/rmfakecloud/data"
DATE=$(date +%Y%m%d-%H%M%S)

# Stop container (optional, for consistent backup)
# docker stop rmfakecloud

# Backup data
rsync -av --delete "$DATA_DIR/" "$BACKUP_DIR/latest/"
tar -czf "$BACKUP_DIR/rmfakecloud-$DATE.tar.gz" -C "$BACKUP_DIR" latest/

# Restart container
# docker start rmfakecloud

# Retention (keep 30 days)
find "$BACKUP_DIR" -name "rmfakecloud-*.tar.gz" -mtime +30 -delete

echo "Backup completed: rmfakecloud-$DATE.tar.gz"
```

**Cron schedule:**
```cron
# Daily at 2 AM
0 2 * * * /usr/local/bin/backup-rmfakecloud.sh
```

#### Strategy 2: Docker Volume Backup

```bash
# Create backup
docker run --rm -v rmfakecloud_data:/data -v $(pwd):/backup \
  alpine tar czf /backup/rmfakecloud-backup-$(date +%Y%m%d).tar.gz /data

# Restore backup
docker run --rm -v rmfakecloud_data:/data -v $(pwd):/backup \
  alpine sh -c "cd / && tar xzf /backup/rmfakecloud-backup-20260114.tar.gz"
```

#### Strategy 3: ZFS/Btrfs Snapshots

```bash
# ZFS snapshot
zfs snapshot tank/rmfakecloud@daily-$(date +%Y%m%d)

# Btrfs snapshot
btrfs subvolume snapshot /data/rmfakecloud /snapshots/rmfakecloud-$(date +%Y%m%d)
```

### Disaster Recovery

**Recovery Time Objective (RTO):** ~15 minutes
**Recovery Point Objective (RPO):** 24 hours (with daily backups)

**Recovery Steps:**
1. Deploy new rmfakecloud instance
2. Restore data directory from backup
3. Restore environment configuration (.env)
4. Start container
5. Verify service health
6. Update DNS if IP changed
7. Test tablet sync

### Data Retention

**Recommendations:**
- **Active Data:** Indefinite (on primary server)
- **Backups:** 30 days rolling
- **Deleted Users:** Archive for 90 days before purging
- **Logs:** 90 days (rotate and compress)

---

## Monitoring and Maintenance

### Health Checks

```bash
# Check if service is running
curl -f https://remarkable.company.internal/ || echo "Service down"

# Check Docker container
docker ps | grep rmfakecloud

# Check logs
docker logs rmfakecloud --tail 100

# Check disk usage
du -sh /path/to/data/*
```

### Monitoring Script

```bash
#!/bin/bash
# monitor-rmfakecloud.sh

ALERT_EMAIL="admin@company.internal"
RMFAKECLOUD_URL="https://remarkable.company.internal"

# Check HTTP
if ! curl -f -s "$RMFAKECLOUD_URL" > /dev/null; then
  echo "rmfakecloud HTTP check failed" | mail -s "rmfakecloud ALERT" "$ALERT_EMAIL"
fi

# Check disk space
USAGE=$(df -h /path/to/data | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$USAGE" -gt 90 ]; then
  echo "Disk usage at ${USAGE}%" | mail -s "rmfakecloud Disk Alert" "$ALERT_EMAIL"
fi

# Check container status
if ! docker ps | grep -q rmfakecloud; then
  echo "rmfakecloud container not running" | mail -s "rmfakecloud ALERT" "$ALERT_EMAIL"
  docker start rmfakecloud
fi
```

### Log Management

```yaml
# docker-compose.yml - Add logging config
services:
  rmfakecloud:
    # ... other config ...
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```

**Centralized Logging (optional):**
```bash
# Send to syslog
logging:
  driver: syslog
  options:
    syslog-address: "udp://syslog-server:514"
    tag: "rmfakecloud"
```

### System Updates

#### Update rmfakecloud

```bash
# Pull latest image
docker pull ddvk/rmfakecloud:latest

# Stop and remove old container
docker stop rmfakecloud
docker rm rmfakecloud

# Start new container (or use docker-compose up -d)
docker-compose up -d

# Verify
docker logs rmfakecloud
curl -f https://remarkable.company.internal/
```

#### Update Tablets (After reMarkable System Update)

**Tablets lose proxy configuration after system updates.**

```bash
# Re-run installer on each tablet
ssh root@10.11.99.1
./installer-rm12.sh install

# Or via toltec
rmfakecloudctl enable
```

### Performance Tuning

#### Database/Storage Optimization
- Use SSD for data directory
- Enable filesystem compression (ZFS/Btrfs)
- Monitor inode usage for many small files

#### Resource Limits
```yaml
# docker-compose.yml
services:
  rmfakecloud:
    # ... other config ...
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
```

---

## Troubleshooting

### Common Issues

#### Issue 1: Tablets Cannot Connect

**Symptoms:** "Sync failed" or "Cannot connect to cloud"

**Diagnosis:**
```bash
# On tablet
ping my.remarkable.com
# Should resolve to rmfakecloud IP, not 0.0.0.0

ping <rmfakecloud-ip>
# Should succeed

wget -qO- http://<rmfakecloud-ip>:3000
# Should return "Working..."

# Check proxy status
systemctl status proxy

# Check certificates
echo Q | openssl s_client -connect localhost:443 \
  -verify_hostname local.appspot.com \
  -CAfile /etc/ssl/certs/ca-certificates.crt 2>&1 | grep Verify
# Should show: Verify return code: 0 (ok)
```

**Solutions:**
- Verify DNS/hosts file entries
- Check CA certificate installation
- Verify proxy is running
- Check firewall rules

#### Issue 2: Authentication Failures

**Symptoms:** "Invalid token" or "Unauthorized"

**Diagnosis:**
```bash
# Check JWT secret hasn't changed
docker exec rmfakecloud env | grep JWT_SECRET_KEY

# Check logs
docker logs rmfakecloud | grep -i auth
```

**Solutions:**
- Regenerate code and re-pair tablet
- Verify JWT_SECRET_KEY is consistent
- Check system time on server and tablet (NTP sync)

#### Issue 3: File Upload Failures

**Symptoms:** Documents don't sync from tablet

**Diagnosis:**
```bash
# Check storage permissions
ls -la /path/to/data/
# Should be owned by container user (typically 1000:1000)

# Check disk space
df -h /path/to/data

# Check logs
docker logs rmfakecloud | grep -i "storage\|upload"
```

**Solutions:**
- Fix permissions: `chown -R 1000:1000 /path/to/data`
- Free up disk space
- Check client_max_body_size in reverse proxy

#### Issue 4: Screen Sharing Not Working

**Symptoms:** Cannot start screen share session

**Diagnosis:**
```bash
# Check MQTT is running
netstat -tuln | grep 8883

# Check TLS cert configured
docker exec rmfakecloud env | grep TLS_

# Check logs
docker logs rmfakecloud | grep -i mqtt
```

**Solutions:**
- Ensure TLS_CERT and TLS_KEY are set
- Verify certificates are valid
- Check MQTT port is open in firewall
- For remote access, configure ICE_SERVERS

#### Issue 5: Email Sending Fails

**Symptoms:** Cannot email documents from tablet

**Diagnosis:**
```bash
# Check SMTP configuration
docker exec rmfakecloud env | grep SMTP

# Test SMTP connection
telnet smtp.company.internal 587

# Check logs
docker logs rmfakecloud | grep -i smtp
```

**Solutions:**
- Verify SMTP credentials
- Check SMTP server allows relay from rmfakecloud IP
- Test with RM_SMTP_INSECURE_TLS=true (temporarily)
- Check RM_SMTP_STARTTLS and RM_SMTP_NOTLS settings

### Debug Mode

```bash
# Enable debug logging
docker-compose down
# Edit .env or docker-compose.yml
LOGLEVEL=debug

docker-compose up -d

# Watch logs
docker logs -f rmfakecloud
```

### Support Resources

- **GitHub Issues:** https://github.com/ddvk/rmfakecloud/issues
- **Documentation:** https://ddvk.github.io/rmfakecloud/
- **Community:** reMarkable Discord, Reddit r/RemarkableTablet

---

## Known Limitations

### Feature Limitations

1. **No Document Rendering:** Web UI cannot display document contents (planned)
2. **No Handwriting Search:** Cannot search within handwritten notes
3. **Single Server:** No native clustering/HA support
4. **File-Based Storage:** Not optimized for extremely large document counts (>10,000 per user)

### Compatibility Limitations

1. **System Updates:** Tablet configuration reset after reMarkable firmware updates
2. **Desktop Clients:** Requires hosts file modification and CA trust
3. **Mobile Apps:** Difficult to configure (requires CA installation on mobile devices)
4. **Software Version:** Tested up to 3.22.0; newer versions may have issues

### Integration Limitations

1. **Dropbox/Google Drive:** Incomplete implementation
2. **OneDrive:** Not supported
3. **Handwriting Recognition:** Requires external MyScript account
4. **Third-Party Apps:** Some third-party remarkable apps may not work

### Security Limitations

1. **AGPL License:** Modifications must be made available to users
2. **File-Based Storage:** No encryption at rest (must use filesystem-level encryption)
3. **No 2FA:** JWT-based authentication only
4. **No Role-Based Access Control:** All users have equal permissions

### Performance Considerations

1. **Large Files:** Very large PDFs (>100MB) may cause sync delays
2. **Many Devices:** >50 concurrent devices may require resource scaling
3. **Network Latency:** Screen sharing quality depends on network conditions

---

## Deployment Checklist

### Pre-Deployment

- [ ] Server meets minimum requirements
- [ ] Network ports planned and documented
- [ ] Firewall rules defined
- [ ] TLS certificates generated or obtained
- [ ] JWT secret generated and secured
- [ ] Backup strategy defined
- [ ] Monitoring plan in place

### Installation

- [ ] Docker and Docker Compose installed
- [ ] rmfakecloud image pulled
- [ ] docker-compose.yml configured
- [ ] .env file created with secrets
- [ ] Data directory created with correct permissions
- [ ] TLS certificates in place
- [ ] Container started successfully

### Configuration

- [ ] DNS records created OR hosts file distribution planned
- [ ] CA certificate distributed to tablets
- [ ] Reverse proxy configured (if used)
- [ ] fail2ban configured (if used)
- [ ] SMTP settings tested (if used)
- [ ] First admin user created

### Tablet Setup

- [ ] rmfakecloud-proxy installed on test tablet
- [ ] DNS/hosts configuration verified
- [ ] CA certificate installed on tablet
- [ ] Test user enrolled and synced
- [ ] Document upload/download tested
- [ ] Email functionality tested (if enabled)

### Post-Deployment

- [ ] Backup script installed and tested
- [ ] Monitoring configured
- [ ] Log rotation configured
- [ ] Documentation updated with local specifics
- [ ] User enrollment process documented
- [ ] Support contact information distributed
- [ ] Disaster recovery plan tested

---

## Conclusion

rmfakecloud provides a robust, self-hosted alternative to the reMarkable cloud service suitable for corporate intranet deployments. Key advantages include:

- **Data Sovereignty:** Complete control over document storage
- **No Subscription Fees:** One-time deployment cost
- **Privacy:** Documents never leave your network
- **Integration:** WebDAV, FTP, and webhook integrations
- **Multi-User:** Support for multiple tablets and users

**Recommended for:**
- Organizations with data sovereignty requirements
- Environments with >5 reMarkable tablets
- Companies requiring integration with existing infrastructure
- Privacy-conscious deployments

**Not recommended for:**
- Remote/distributed teams (complex VPN setup needed)
- Organizations without IT infrastructure
- Users requiring mobile app sync
- Deployments requiring high availability (file-based storage limitation)

For successful deployment, allocate time for proper security hardening, backup configuration, and user training. Regular maintenance including backups, updates, and monitoring is essential for reliable operation.

---

**Document Version:** 1.0
**Last Updated:** January 14, 2026
**Maintainer:** Internal IT Documentation
**Review Cycle:** Quarterly
