# rmfakecloud Enterprise Deployment Guide

**Version:** 0.0.25+
**Document Date:** January 2026
**License:** GNU AGPL v3.0

## Executive Summary

rmfakecloud is a self-hosted replacement for the reMarkable cloud service, enabling organizations to maintain complete control over document synchronization, storage, and backup for reMarkable tablets. This deployment guide provides verified features, security considerations, and best practices for deploying rmfakecloud on a corporate intranet for multiple devices.

## Table of Contents

1. [Verified Feature Set](#verified-feature-set)
2. [System Architecture](#system-architecture)
3. [Healthcare/Medical Facility Deployment](#healthcaremedical-facility-deployment)
4. [Deployment Requirements](#deployment-requirements)
5. [Deployment Scenarios](#deployment-scenarios)
6. [Security Considerations](#security-considerations)
7. [Multi-Device Configuration](#multi-device-configuration)
8. [Document Lifecycle Management](#document-lifecycle-management)
9. [Backup and Data Management](#backup-and-data-management)
10. [Monitoring and Maintenance](#monitoring-and-maintenance)
11. [Troubleshooting](#troubleshooting)
12. [Known Limitations](#known-limitations)

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

## Healthcare/Medical Facility Deployment

### Overview

This section addresses specific considerations for deploying rmfakecloud in healthcare settings where reMarkable tablets are used for:
- Clinical note-taking
- Paperless form filling
- Patient chart annotations
- Medical documentation

**Critical Understanding:** Protected Health Information (PHI) and personally identifiable information (PII) will be stored on both the tablets and the rmfakecloud server. Proper security controls, lifecycle management, and compliance procedures are essential.

### Compliance Considerations

#### HIPAA Compliance (U.S. Healthcare)

rmfakecloud is **not HIPAA-compliant out of the box**. To use in a HIPAA-covered entity environment, you must implement additional controls:

**Required Technical Safeguards:**

1. **Access Controls (§164.312(a)(1))**
   - ✅ User authentication (JWT-based) - **Implemented**
   - ❌ Role-based access control - **Not implemented** (all users have equal permissions)
   - ❌ Automatic logoff - **Not implemented** (sessions expire after 24 hours)
   - ⚠️ Unique user identification - **Partially implemented** (email-based, but no audit trail)

2. **Audit Controls (§164.312(b))**
   - ❌ **NOT IMPLEMENTED** - rmfakecloud does not log PHI access by default
   - **Action Required:** Implement external audit logging (see [Audit Logging](#audit-logging) below)

3. **Integrity (§164.312(c)(1))**
   - ✅ Data integrity during transmission (TLS)
   - ⚠️ File checksums maintained for sync
   - ❌ No built-in integrity monitoring or tampering detection

4. **Transmission Security (§164.312(e)(1))**
   - ✅ TLS encryption for data in transit - **Can be implemented**
   - ⚠️ Requires proper TLS configuration (see Security section)

5. **Encryption at Rest (Addressable)**
   - ❌ **NOT IMPLEMENTED** in application
   - **Action Required:** Use filesystem-level encryption (LUKS, dm-crypt)

**Required Administrative Safeguards:**

- Business Associate Agreements (BAA): Open-source software has no vendor to sign BAA with
- Security policies and procedures: Must be documented separately
- Workforce training: Required for proper PHI handling
- Contingency planning: Backup and disaster recovery (covered in [Backup Section](#backup-and-data-management))

**Physical Safeguards:**

- Server must be in secure, access-controlled location
- Tablets must have screen locks and encryption enabled
- Consider asset management for tablet tracking

**Recommendation:** Consult with HIPAA compliance officer and legal counsel before deploying for PHI storage.

#### Other Regulatory Frameworks

- **GDPR (EU):** Personal data processing requires legal basis, data protection measures, and subject rights implementation
- **PIPEDA (Canada):** Similar requirements for personal health information
- **State Privacy Laws:** California CPRA, etc., may apply

### Document Lifecycle Management

#### Centralized Document Control

rmfakecloud supports **server-side document lifecycle management** with automatic synchronization to tablets:

**Key Capability:** When files are deleted from the server's user directory, they are automatically deleted from tablets on next sync.

**Verified in Code:** `/internal/storage/fs/documents.go` line 123-147

**How It Works:**

1. **Deletion from Web UI:**
   ```
   Admin deletes document via web interface
   → Document moved to `.trash` folder in user directory
   → On next sync, tablet receives deletion instruction
   → Document removed from tablet
   ```

2. **Deletion from Server Filesystem:**
   ```
   Admin deletes files from /data/users/<user-id>/ directory
   → On next sync, tablets detect missing files
   → Tablets delete local copies to match server state
   ```

3. **Bulk Deletion:**
   ```bash
   # Delete all documents for a specific user older than 90 days
   find /data/users/<user-id>/ -name "*.zip" -mtime +90 -delete
   find /data/users/<user-id>/ -name "*.metadata" -mtime +90 -delete
   # On next sync, tablets will remove these documents
   ```

**Important Warnings:**

⚠️ **Deletion is NOT immediate** - Tablets must connect and sync for deletion to take effect
⚠️ **No confirmation mechanism** - Cannot verify document was deleted from tablet without manual inspection
⚠️ **Offline tablets retain data** - If a tablet never syncs, it will keep deleted documents indefinitely
⚠️ **Trash folder persists** - Documents in `.trash` are not automatically purged

#### Document Retention Policies

**Implementing Retention:**

```bash
#!/bin/bash
# retention-policy.sh - Enforce document retention on rmfakecloud server

DATA_DIR="/path/to/rmfakecloud/data/users"
RETENTION_DAYS=2555  # 7 years for medical records
LOG_FILE="/var/log/rmfakecloud-retention.log"

# Find and remove documents older than retention period
for user_dir in "$DATA_DIR"/*; do
    user=$(basename "$user_dir")

    # Find old documents
    old_docs=$(find "$user_dir" -name "*.zip" -mtime +"$RETENTION_DAYS")

    if [ -n "$old_docs" ]; then
        echo "$(date): Removing expired documents for user $user" >> "$LOG_FILE"

        # Delete documents and metadata
        find "$user_dir" -name "*.zip" -mtime +"$RETENTION_DAYS" -delete
        find "$user_dir" -name "*.metadata" -mtime +"$RETENTION_DAYS" -delete

        # Log document IDs that were deleted
        echo "$old_docs" >> "$LOG_FILE"
    fi
done

# Purge trash older than 30 days
find "$DATA_DIR"/*/".trash" -type f -mtime +30 -delete

echo "$(date): Retention policy executed" >> "$LOG_FILE"
```

**Cron Schedule:**
```cron
# Run retention policy daily at 3 AM
0 3 * * * /usr/local/bin/retention-policy.sh
```

**Retention Recommendations by Document Type:**

| Document Type | Recommended Retention | Legal Basis |
|---------------|----------------------|-------------|
| Patient clinical notes | 7 years (2555 days) | HIPAA, state law |
| Pediatric records | 7 years after age 18 | State requirements |
| Consent forms | 7 years | Legal requirement |
| Administrative notes | 1 year (365 days) | Internal policy |
| Temporary forms | 30 days | Internal policy |

#### Remote Document Management

**Via Web Interface:**

Administrators can manage documents through the web UI:

```
https://rmfakecloud.company.internal/
→ Login as admin
→ Navigate to user's document list
→ Select documents
→ Delete (moves to trash)
→ Documents removed from tablets on next sync
```

**Via API (Programmatic):**

```bash
# Authenticate
TOKEN=$(curl -s -X POST https://rmfakecloud.company.internal/ui/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@company.internal","password":"admin-password"}')

# List user documents
curl -H "Cookie: .Authrmfakecloud=$TOKEN" \
  https://rmfakecloud.company.internal/ui/api/documents

# Delete document
curl -X DELETE -H "Cookie: .Authrmfakecloud=$TOKEN" \
  https://rmfakecloud.company.internal/ui/api/documents/<document-id>
```

**Via Direct Filesystem (Bulk Operations):**

```bash
# SSH to rmfakecloud server

# Delete specific document by ID
rm /data/users/<user-id>/<document-id>.zip
rm /data/users/<user-id>/<document-id>.metadata

# Delete all documents for terminated user
rm -rf /data/users/<user-id>/*

# Tablets will remove documents on next sync
```

#### Limitations and Workarounds

**Limitation 1: No Forced Sync**
- Cannot force tablets to sync immediately
- **Workaround:** Establish policy requiring daily syncs; monitor last-sync timestamps

**Limitation 2: No Remote Wipe**
- Cannot remotely wipe tablet if lost/stolen
- **Workaround:**
  - Enable reMarkable's built-in device password
  - Use tablet asset management
  - Document decommissioning procedure

**Limitation 3: No Deletion Confirmation**
- Cannot verify document was removed from tablet
- **Workaround:** Manual verification procedure or audit script

**Limitation 4: Offline Retention**
- Tablets that never connect retain all documents
- **Workaround:** Policy requiring regular connectivity; disable WiFi-only tablets

### Audit Logging

rmfakecloud does **not** provide built-in PHI access logging. For HIPAA compliance, implement external audit logging.

#### Option 1: Reverse Proxy Logging (nginx)

```nginx
# /etc/nginx/sites-available/rmfakecloud

log_format rmfakecloud_audit '$remote_addr - $remote_user [$time_local] '
                              '"$request" $status $body_bytes_sent '
                              '"$http_referer" "$http_user_agent" '
                              'auth="$http_authorization" cookie="$http_cookie"';

server {
    listen 443 ssl http2;
    server_name rmfakecloud.company.internal;

    # Separate audit log
    access_log /var/log/nginx/rmfakecloud-audit.log rmfakecloud_audit;

    # ... rest of config
}
```

**Log Rotation:**
```bash
# /etc/logrotate.d/rmfakecloud-audit
/var/log/nginx/rmfakecloud-audit.log {
    daily
    rotate 2555  # 7 years
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

#### Option 2: Syslog Integration

```yaml
# docker-compose.yml
services:
  rmfakecloud:
    # ... other config ...
    logging:
      driver: syslog
      options:
        syslog-address: "tcp://syslog-server.company.internal:514"
        tag: "rmfakecloud"
        syslog-format: "rfc5424"
```

**SIEM Integration:** Forward syslog to Splunk, ELK Stack, or QRadar for compliance monitoring.

#### Option 3: Filesystem Auditing (auditd)

```bash
# Monitor file access in data directory
auditctl -w /data/users -p rwxa -k rmfakecloud_phi_access

# Watch for deletions
auditctl -w /data/users -p d -k rmfakecloud_phi_delete

# Query audit logs
ausearch -k rmfakecloud_phi_access -ts recent
```

#### Audit Log Requirements

For HIPAA compliance, logs must include:
- User identity (email/username)
- Timestamp (accurate, synchronized with NTP)
- Action performed (create, read, update, delete)
- Document/resource accessed
- Source IP address
- Success/failure indication

**Retention:** Audit logs must be retained for **6 years** (HIPAA requirement).

### Device Management

#### Device Enrollment

Track all tablets in inventory management system:

```bash
# Device enrollment script
# /usr/local/bin/enroll-tablet.sh

DEVICE_IP="$1"
USER_EMAIL="$2"
ASSET_TAG="$3"

# SSH to tablet and get device info
DEVICE_ID=$(ssh root@"$DEVICE_IP" "cat /sys/class/net/wlan0/address")
SERIAL=$(ssh root@"$DEVICE_IP" "cat /proc/device-tree/serial-number")

# Record in inventory database/spreadsheet
echo "$(date),$ASSET_TAG,$SERIAL,$DEVICE_ID,$USER_EMAIL,Enrolled" >> /var/lib/rmfakecloud/device-inventory.csv

echo "Tablet enrolled: Asset $ASSET_TAG assigned to $USER_EMAIL"
```

**Inventory Tracking:**

| Asset Tag | Serial Number | MAC Address | Assigned User | Status | Last Sync |
|-----------|---------------|-------------|---------------|--------|-----------|
| RM-001 | ABC123 | 00:11:22:33:44:55 | dr.smith@hospital.internal | Active | 2026-01-14 08:30 |
| RM-002 | ABC124 | 00:11:22:33:44:56 | nurse.jones@hospital.internal | Active | 2026-01-14 09:15 |

#### Device Decommissioning

When a tablet is lost, stolen, or decommissioned:

```bash
#!/bin/bash
# decommission-tablet.sh

USER_EMAIL="$1"
REASON="$2"  # lost, stolen, retired, etc.

# 1. Delete user's data from server (forces deletion on tablet if it syncs)
rm -rf /data/users/"$USER_EMAIL"/*

# 2. Disable user account
docker exec rmfakecloud /rmfakecloud-docker deleteuser "$USER_EMAIL"

# 3. Log the decommissioning
echo "$(date): Decommissioned $USER_EMAIL - Reason: $REASON" >> /var/log/rmfakecloud-decommission.log

# 4. If stolen, contact security team
if [ "$REASON" = "stolen" ]; then
    echo "ALERT: Tablet stolen for user $USER_EMAIL" | mail -s "Security Alert" security@company.internal
fi

echo "User $USER_EMAIL decommissioned. Data deleted from server."
echo "NOTE: If tablet was lost/stolen, PHI may still be on device."
echo "Physical recovery or remote wipe not possible with rmfakecloud."
```

**Important:** rmfakecloud has **no remote wipe capability**. Lost or stolen tablets may still contain PHI. This is a significant limitation for healthcare deployments.

**Mitigation:**
- Require device passwords on all tablets
- Encrypt tablet storage (if supported by reMarkable OS)
- Maintain physical security of tablets
- Consider tablet insurance and tracking

#### Monitoring Last Sync Times

```bash
#!/bin/bash
# check-sync-status.sh - Alert on tablets that haven't synced recently

DATA_DIR="/data/users"
MAX_DAYS=1  # Alert if not synced in 1 day

for user_dir in "$DATA_DIR"/*; do
    user=$(basename "$user_dir")

    # Find most recent file modification
    last_sync=$(find "$user_dir" -type f -printf '%T@\n' | sort -n | tail -1)

    if [ -z "$last_sync" ]; then
        continue
    fi

    # Calculate days since last sync
    current_time=$(date +%s)
    days_since_sync=$(( (current_time - ${last_sync%.*}) / 86400 ))

    if [ "$days_since_sync" -gt "$MAX_DAYS" ]; then
        echo "WARNING: User $user has not synced in $days_since_sync days" | \
            mail -s "rmfakecloud Sync Alert" admin@company.internal
    fi
done
```

**Cron:**
```cron
# Check sync status every 6 hours
0 */6 * * * /usr/local/bin/check-sync-status.sh
```

### Data Sanitization

#### Permanent Deletion

When documents must be permanently deleted (e.g., patient request, legal requirement):

```bash
#!/bin/bash
# secure-delete.sh - Permanently delete document with verification

USER="$1"
DOCUMENT_ID="$2"

DATA_DIR="/data/users/$USER"

# 1. Remove from active storage
rm -f "$DATA_DIR/$DOCUMENT_ID.zip"
rm -f "$DATA_DIR/$DOCUMENT_ID.metadata"

# 2. Remove from trash
rm -f "$DATA_DIR/.trash/$DOCUMENT_ID.zip"
rm -f "$DATA_DIR/.trash/$DOCUMENT_ID.metadata"

# 3. Remove from cache
rm -f "$DATA_DIR/.cache/$DOCUMENT_ID"*

# 4. Remove from sync folder (if using sync15)
rm -rf "$DATA_DIR/sync/$DOCUMENT_ID"*

# 5. Overwrite with zeros (optional, for compliance)
# Note: Filesystem must not use copy-on-write (ZFS, Btrfs)
# shred -vfz -n 3 "$DATA_DIR/$DOCUMENT_ID"*

# 6. Log deletion for audit
echo "$(date): Securely deleted document $DOCUMENT_ID for user $USER" >> /var/log/rmfakecloud-deletions.log

# 7. Force sync on user's tablets (they must connect)
echo "Document deleted. User $USER must sync tablets to complete deletion."
```

**Limitation:** Secure deletion (overwriting) may not work on:
- Copy-on-write filesystems (ZFS, Btrfs)
- SSD drives (wear leveling)
- Compressed/deduplicated storage
- Backup media

**For true secure deletion:** Use encrypted storage and securely delete encryption keys.

#### User Account Purge

Complete removal of user and all data:

```bash
#!/bin/bash
# purge-user.sh - Complete user data removal

USER_EMAIL="$1"

# 1. Export data for archival if required by law
tar -czf "/backup/archive/user-$USER_EMAIL-$(date +%Y%m%d).tar.gz" /data/users/"$USER_EMAIL"/

# 2. Delete user directory
rm -rf /data/users/"$USER_EMAIL"

# 3. Remove user account
docker exec rmfakecloud /rmfakecloud-docker deleteuser "$USER_EMAIL"

# 4. Update inventory
sed -i "/$USER_EMAIL/d" /var/lib/rmfakecloud/device-inventory.csv

# 5. Log purge
echo "$(date): Purged user $USER_EMAIL completely" >> /var/log/rmfakecloud-purge.log

echo "User $USER_EMAIL purged from system"
```

### Best Practices for Healthcare Deployment

#### 1. Network Segmentation
- Place rmfakecloud server on dedicated healthcare VLAN
- Isolate from guest network and general corporate network
- Use firewall rules to restrict access to authorized devices only

#### 2. Regular Security Audits
- Quarterly review of access logs
- Annual penetration testing
- Regular vulnerability assessments

#### 3. User Training
- PHI handling procedures
- Tablet security (screen locks, physical security)
- Reporting lost/stolen devices
- Proper logout procedures

#### 4. Incident Response Plan
```
Lost/Stolen Tablet Procedure:
1. User reports loss immediately
2. IT runs decommission-tablet.sh
3. Data deleted from server
4. User account disabled
5. Security team notified
6. Incident logged
7. User assigned new tablet with new credentials
```

#### 5. Business Continuity
- Daily backups with 7-year retention
- Tested disaster recovery procedures
- Redundant hardware available
- Documentation of all procedures

#### 6. Acceptable Use Policy

**Sample Policy Elements:**
- Tablets are for authorized clinical use only
- No personal use of tablets
- Tablets must not leave facility premises (unless authorized)
- Tablets must be locked when unattended
- Users must sync tablets daily
- Lost/stolen tablets must be reported within 1 hour
- PHI must not be shared or displayed in public areas
- Screen sharing feature disabled (unless required and approved)

---

## Document Lifecycle Management

This section provides detailed procedures for managing document lifecycles directly on tablets through server-side operations.

### Understanding Sync Behavior

rmfakecloud uses a **bidirectional sync model**:

- **Tablet → Server:** New/modified documents uploaded to server
- **Server → Tablet:** Server state is authoritative; deletions propagate to tablets

**Critical Sync Rule:** Files deleted from the server user directory will be deleted from tablets on next sync.

**Verification:** Confirmed in `/home/user/rmfakecloud/README.md` line 59:
> "if you delete files from the users directory on the host, on the next sync those will be deleted from the device"

### Server-Side Document Operations

#### Viewing User Documents

```bash
# List all documents for a user
ls -lh /data/users/<user-email>/

# Output:
# -rw------- 1 1000 1000  2.3M Jan 14 10:30 abc123def.zip
# -rw------- 1 1000 1000   512 Jan 14 10:30 abc123def.metadata
# -rw------- 1 1000 1000  1.8M Jan 13 15:22 xyz789ghi.zip
# -rw------- 1 1000 1000   498 Jan 13 15:22 xyz789ghi.metadata

# For sync15 users, documents are in sync folder
ls -lh /data/users/<user-email>/sync/
```

#### Metadata Inspection

```bash
# View document metadata (JSON format)
cat /data/users/<user-email>/abc123def.metadata

# Extract document name
jq -r '.VisibleName' /data/users/<user-email>/abc123def.metadata

# Find all documents modified in last 7 days
find /data/users/<user-email>/ -name "*.metadata" -mtime -7 -exec jq -r '.VisibleName' {} \;
```

#### Bulk Document Management

```bash
#!/bin/bash
# bulk-document-operations.sh

USER_EMAIL="$1"
OPERATION="$2"  # list, delete-old, archive, export

USER_DIR="/data/users/$USER_EMAIL"

case "$OPERATION" in
    list)
        # List all documents with names
        for metadata in "$USER_DIR"/*.metadata; do
            doc_id=$(basename "$metadata" .metadata)
            doc_name=$(jq -r '.VisibleName' "$metadata" 2>/dev/null || echo "Unknown")
            doc_date=$(stat -c %y "$metadata" | cut -d' ' -f1)
            echo "$doc_id | $doc_name | $doc_date"
        done
        ;;

    delete-old)
        # Delete documents older than specified days
        DAYS="${3:-90}"
        echo "Deleting documents older than $DAYS days for $USER_EMAIL"

        find "$USER_DIR" -name "*.zip" -mtime +"$DAYS" | while read -r zipfile; do
            doc_id=$(basename "$zipfile" .zip)
            doc_name=$(jq -r '.VisibleName' "$USER_DIR/$doc_id.metadata" 2>/dev/null || echo "Unknown")

            echo "Deleting: $doc_name ($doc_id)"
            rm -f "$zipfile"
            rm -f "$USER_DIR/$doc_id.metadata"
        done
        ;;

    archive)
        # Archive documents to external storage
        ARCHIVE_DIR="/backup/archive/$USER_EMAIL"
        mkdir -p "$ARCHIVE_DIR"

        tar -czf "$ARCHIVE_DIR/archive-$(date +%Y%m%d-%H%M%S).tar.gz" \
            -C "$USER_DIR" .

        echo "Archived to $ARCHIVE_DIR"
        ;;

    export)
        # Export document list to CSV
        echo "DocumentID,Name,ModifiedDate,Size" > "/tmp/$USER_EMAIL-docs.csv"

        for metadata in "$USER_DIR"/*.metadata; do
            doc_id=$(basename "$metadata" .metadata)
            doc_name=$(jq -r '.VisibleName' "$metadata" 2>/dev/null || echo "Unknown")
            doc_date=$(stat -c %y "$metadata" | cut -d' ' -f1)
            doc_size=$(stat -c %s "$USER_DIR/$doc_id.zip" 2>/dev/null || echo "0")

            echo "$doc_id,\"$doc_name\",$doc_date,$doc_size" >> "/tmp/$USER_EMAIL-docs.csv"
        done

        echo "Exported to /tmp/$USER_EMAIL-docs.csv"
        ;;
esac
```

**Usage:**
```bash
# List all documents for user
./bulk-document-operations.sh dr.smith@hospital.internal list

# Delete documents older than 60 days
./bulk-document-operations.sh dr.smith@hospital.internal delete-old 60

# Archive all documents
./bulk-document-operations.sh dr.smith@hospital.internal archive

# Export document inventory
./bulk-document-operations.sh dr.smith@hospital.internal export
```

### Automated Lifecycle Policies

```bash
#!/bin/bash
# lifecycle-manager.sh - Automated document lifecycle management

DATA_DIR="/data/users"
POLICY_CONFIG="/etc/rmfakecloud/lifecycle-policy.conf"

# Load policy configuration
# Format: document_pattern,retention_days,action
# Example:
# "Temp*,7,delete"
# "Patient Chart*,2555,archive"
# "Form*,30,delete"

while IFS=',' read -r pattern retention action; do
    echo "Processing policy: Pattern=$pattern, Retention=$retention days, Action=$action"

    for user_dir in "$DATA_DIR"/*; do
        user=$(basename "$user_dir")

        # Find matching documents
        for metadata in "$user_dir"/*.metadata; do
            [ -f "$metadata" ] || continue

            doc_name=$(jq -r '.VisibleName' "$metadata" 2>/dev/null)
            doc_id=$(basename "$metadata" .metadata)
            doc_age_days=$(( ($(date +%s) - $(stat -c %Y "$metadata")) / 86400 ))

            # Check if document matches pattern and age
            if [[ "$doc_name" == $pattern ]] && [ "$doc_age_days" -gt "$retention" ]; then
                case "$action" in
                    delete)
                        echo "Deleting: $user/$doc_name (age: $doc_age_days days)"
                        rm -f "$user_dir/$doc_id.zip"
                        rm -f "$metadata"
                        ;;
                    archive)
                        echo "Archiving: $user/$doc_name"
                        mkdir -p "/archive/$user"
                        mv "$user_dir/$doc_id.zip" "/archive/$user/"
                        mv "$metadata" "/archive/$user/"
                        ;;
                esac
            fi
        done
    done
done < "$POLICY_CONFIG"
```

**Policy Configuration File:**
```bash
# /etc/rmfakecloud/lifecycle-policy.conf
# Format: pattern,retention_days,action

# Temporary documents - delete after 7 days
Temp*,7,delete
temp*,7,delete
TEMP*,7,delete

# Draft documents - delete after 30 days
Draft*,30,delete
draft*,30,delete

# Patient charts - archive after 7 years
Patient Chart*,2555,archive
Medical Record*,2555,archive

# Forms - delete after 1 year
Form*,365,delete
Consent*,2555,archive

# Notes - delete after 90 days
Notes*,90,delete
Daily Note*,90,delete
```

### Web UI Document Management

Administrators can manage documents through the web interface:

**Access Control:**
1. Login as admin user: `https://rmfakecloud.company.internal/`
2. Navigate to "Users" → Select user
3. View document list with metadata

**Operations Available:**
- **View:** List all documents with names and dates
- **Download:** Export document as PDF (requires HWR for handwritten notes)
- **Delete:** Move document to trash (syncs deletion to tablet)
- **Rename:** Change document name
- **Move:** Reorganize into folders

**API Access for Automation:**

```python
#!/usr/bin/env python3
# rmfakecloud-document-manager.py

import requests
import json

class RmFakeCloudManager:
    def __init__(self, base_url, admin_email, admin_password):
        self.base_url = base_url
        self.session = requests.Session()
        self.login(admin_email, admin_password)

    def login(self, email, password):
        """Authenticate and get session cookie"""
        response = self.session.post(
            f"{self.base_url}/ui/api/login",
            json={"email": email, "password": password}
        )
        response.raise_for_status()
        return response.text

    def list_documents(self, user_id):
        """List all documents for a user"""
        response = self.session.get(
            f"{self.base_url}/ui/api/documents",
            params={"userid": user_id}
        )
        response.raise_for_status()
        return response.json()

    def delete_document(self, doc_id):
        """Delete a document"""
        response = self.session.delete(
            f"{self.base_url}/ui/api/documents/{doc_id}"
        )
        response.raise_for_status()
        return True

    def bulk_delete_old_documents(self, user_id, days_old):
        """Delete documents older than specified days"""
        from datetime import datetime, timedelta

        docs = self.list_documents(user_id)
        cutoff_date = datetime.now() - timedelta(days=days_old)

        deleted_count = 0
        for doc in docs:
            doc_date = datetime.fromisoformat(doc['ModifiedClient'])
            if doc_date < cutoff_date:
                print(f"Deleting: {doc['VissibleName']} (modified: {doc_date})")
                self.delete_document(doc['ID'])
                deleted_count += 1

        return deleted_count

# Usage
if __name__ == "__main__":
    manager = RmFakeCloudManager(
        "https://rmfakecloud.company.internal",
        "admin@company.internal",
        "admin-password"
    )

    # Delete documents older than 90 days for user
    deleted = manager.bulk_delete_old_documents("dr.smith@hospital.internal", 90)
    print(f"Deleted {deleted} old documents")
```

### Sync Verification

After performing lifecycle operations, verify tablets receive changes:

```bash
#!/bin/bash
# verify-sync.sh - Monitor tablet sync after document operations

USER_DIR="/data/users/$1"
OPERATION_TIME=$(date +%s)

echo "Monitoring $USER_DIR for sync activity..."
echo "Performed operation at: $(date)"

# Watch for file access (indicates sync)
inotifywait -m -e access,modify,delete "$USER_DIR" | while read -r path action file; do
    echo "[$(date)] Sync detected: $action on $file"
done

# Alternative: Check modification times
echo "Waiting 5 minutes for sync..."
sleep 300

LATEST_CHANGE=$(find "$USER_DIR" -type f -printf '%T@\n' | sort -n | tail -1)
LATEST_CHANGE_INT=${LATEST_CHANGE%.*}

if [ "$LATEST_CHANGE_INT" -gt "$OPERATION_TIME" ]; then
    echo "✓ Tablet has synced (last change: $(date -d @$LATEST_CHANGE_INT))"
else
    echo "✗ No sync detected. Tablet may be offline."
fi
```

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
- Healthcare facilities (with proper security controls and compliance measures)
- Environments requiring centralized document lifecycle management

**Not recommended for:**
- Remote/distributed teams (complex VPN setup needed)
- Organizations without IT infrastructure
- Users requiring mobile app sync
- Deployments requiring high availability (file-based storage limitation)
- HIPAA-covered entities without additional security controls and audit logging
- Environments requiring immediate remote wipe capability

For successful deployment, allocate time for proper security hardening, backup configuration, and user training. Regular maintenance including backups, updates, and monitoring is essential for reliable operation.

**Healthcare/Medical Deployments:** Organizations deploying rmfakecloud for PHI storage must implement additional controls including audit logging, filesystem encryption, document retention policies, and device management procedures. Consult with compliance officers and legal counsel to ensure regulatory requirements are met. Review the [Healthcare/Medical Facility Deployment](#healthcaremedical-facility-deployment) section for detailed guidance on lifecycle management and compliance considerations.

---

**Document Version:** 1.0
**Last Updated:** January 14, 2026
**Maintainer:** Internal IT Documentation
**Review Cycle:** Quarterly
