# Complete Guide to Installing Overleaf Community Edition on VPS

## Disclaimer
I am not affiliated with the Overleaf project in any way. I would like to thank the creators of Overleaf (https://github.com/overleaf/overleaf) for their amazing work.

Given the recent Overleaf downtime, I decided to self-host it on a VPS (ubuntu-s-2vcpu-4gb) to use at least in emergency situations. Currently, I haven't been able to make all complex libraries work, but it functions with basic features and allows me to continue working. This is the dummies guide I wish I had found.

### Important Clarifications
This repository **contains only a guide** on how to install Overleaf Community Edition. It does **not** provide the Overleaf software itself, which is only available through the official repository at https://github.com/overleaf/overleaf. For any software updates, official documentation, or support, please refer to the official Overleaf repository.

All instructions in this guide reference the official Overleaf Community Edition software. If you encounter any issues with the software itself, please direct them to the official Overleaf project, not to this guide repository.

If Overleaf or any authorized representative of Overleaf wishes for this guide to be removed for any reason, they may simply contact me directly, and I will promptly comply with their request. This guide was created with the sole intention of helping individuals in emergency situations and with no intention to infringe upon Overleaf's rights or interests.

### Important Warning and Liability Disclaimer
This guide is provided "AS IS" without any warranties of any kind, either express or implied. By using this guide, you assume all risks and responsibilities for:

- Any potential damage (including severe damage) to your system
- Data loss or corruption 
- Security vulnerabilities in your self-hosted instance
- Unprotected or improperly protected data
- Any other consequences that may result from following these instructions

The self-hosted Overleaf instance described in this guide may not include all security features of the official service. Documents stored in your self-hosted instance might not have the same level of protection against unauthorized access, data breaches, or data loss compared to the official service.

Users should implement appropriate security measures, regular backups, and access controls to protect their data. The author of this guide assumes no liability for any damages or issues that may arise from following these instructions.

**USE AT YOUR OWN RISK.**

## Table of Contents
- [Prerequisites](#prerequisites)
- [VPS Preparation](#vps-preparation)
- [Docker Installation](#docker-installation)
- [Overleaf Installation](#overleaf-installation)
- [Overleaf Configuration](#overleaf-configuration)
- [Administrator Account Creation](#administrator-account-creation)
- [Complete TeX Live Installation](#complete-tex-live-installation)
- [Automation and Boot Start](#automation-and-boot-start)
- [Backup and Maintenance](#backup-and-maintenance)
- [Security and Production](#security-and-production)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Minimum Recommended Hardware
- **CPU**: 2 cores (with AVX instruction support)
- **RAM**: 4 GB
- **Disk space**: 25-30 GB (complete TeX Live requires about 5-6 GB)
- **Operating system**: Ubuntu 20.04 LTS or higher

### Software Requirements
- Root access to VPS
- Domain name (optional, but recommended for production environments)

### Note on AVX Support
A critical requirement is that the CPU supports the AVX instruction set, necessary for MongoDB 5.0+ used by Overleaf. Many budget VPS providers or those with older hardware might not support AVX. Without this support, the installation will fail.

## VPS Preparation

### System Update

```bash
# Update packages
apt update && apt upgrade -y

# Install basic utilities
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release git ufw screen
```

### Verify AVX Support

MongoDB 5.0+ used by Overleaf requires processors with AVX support:

```bash
grep -q avx /proc/cpuinfo && echo "AVX supported" || echo "AVX NOT supported"
```

If not supported, you'll need to use a VPS with newer hardware.

### Firewall Configuration

```bash
# Set default rules
ufw default deny incoming
ufw default allow outgoing

# Allow SSH and HTTP
ufw allow ssh
ufw allow http
ufw allow https

# Enable firewall
ufw enable
```

## Docker Installation

### Remove Previous Versions (if present)

```bash
apt remove -y docker docker-engine docker.io containerd runc
```

### Docker and Docker Compose Installation

```bash
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

# Add Docker repository
add-apt-repository "deb [arch=$(dpkg --print-architecture)] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# Update and install Docker
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### Verify Installation

```bash
docker --version
docker compose version
```

## Overleaf Installation

### Download Overleaf Toolkit

```bash
# Create directory for Overleaf
mkdir -p /opt/overleaf
cd /opt/overleaf

# Clone the toolkit repository
git clone https://github.com/overleaf/toolkit.git .
```

### Initialization

```bash
# Initialize configuration
./bin/init
```

## Overleaf Configuration

### Remote Access Configuration

```bash
# Edit configuration file
nano config/overleaf.rc
```

Add/modify the following lines:

```
# Set Overleaf to listen on all network interfaces
OVERLEAF_LISTEN_IP=0.0.0.0

# Set site URL (replace with your VPS's public IP)
OVERLEAF_SITE_URL=http://YOUR-VPS-IP

# Disable separate containers for compilation (for now)
SIBLING_CONTAINERS_ENABLED=false
```

**IMPORTANT**: Replace `YOUR-VPS-IP` with your VPS's actual public IP address.

### Using Screen for Persistent Sessions

Screen is a utility that allows processes to keep running even after disconnecting from the VPS, essential for long installations.

#### Install Screen (if not already installed)

```bash
apt install -y screen
```

#### Start a Screen Session

```bash
# Start a new screen session with a descriptive name
screen -S overleaf
```

#### Main Screen Commands

- **Detach**: To exit the session while keeping processes running, press `Ctrl+A` followed by `D`.
- **Reattach**: To return to an active session:
  ```bash
  screen -r overleaf
  ```
- **List sessions**: To see all active screen sessions:
  ```bash
  screen -ls
  ```
- **Terminate a session**: To completely close a session, reattach to it and then press `Ctrl+A` followed by `K`, or simply type `exit` in the session.

### Start Overleaf with Screen

Within the screen session, start Overleaf:

```bash
# Make sure you're in the correct directory
cd /opt/overleaf

# Start Overleaf in detached mode
./bin/up -d
```

After starting Overleaf, you can safely detach from the screen session (Ctrl+A, D) and even disconnect from the VPS. The processes will continue running.

### Verify Installation

```bash
# Check that containers are running
docker ps
```

You should see three containers: `sharelatex`, `mongo`, and `redis`.

## Administrator Account Creation and User Management

### Create the First Administrator

1. Open a browser and go to:
   ```
   http://YOUR-VPS-IP/launchpad
   ```

2. Fill in the form with your data:
   - Email: enter your email
   - Password: choose a secure password

3. Click "Register"

4. Go to the login page (http://YOUR-VPS-IP/login) and log in with the credentials you created

### Creating Additional User Accounts

To create new normal user accounts, you have two options:

#### Option 1: Registration from Launchpad (for new administrators)
New administrators can be created using the launchpad page, as shown above.

#### Option 2: Manual Creation from Command Line

```bash
# To create a normal user
docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex; grunt user:create --email=user@example.com"

# To create a user and set them as administrator
docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex; grunt user:create-admin --email=admin@example.com"
```

Users will receive an email with a link to set their password (if email configuration is active) or will need to use the password recovery function.

### User Administration

#### Reset a User's Password
```bash
docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex; grunt user:reset-password --email=user@example.com"
```

#### Promote a User to Administrator
```bash
docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex; grunt user:make-admin --email=user@example.com"
```

#### Delete a User
```bash
docker exec sharelatex /bin/bash -c "cd /var/www/sharelatex; grunt user:delete --email=user@example.com"
```

## Complete TeX Live Installation

### Basic Package Installation

```bash
# Reconnect to the screen session if necessary
screen -r overleaf

# Install all TeX Live packages
docker exec sharelatex tlmgr install scheme-full

# Update PATH
docker exec sharelatex tlmgr path add
```

Complete TeX Live installation will take about 30-60 minutes depending on connection speed.

### Additional Package Installation

```bash
# Install additional packages for language support and extra functionality
docker exec sharelatex apt-get update
docker exec sharelatex apt-get install -y \
    texlive-lang-all \
    texlive-bibtex-extra \
    texlive-fonts-extra \
    texlive-science \
    texlive-humanities \
    texlive-publishers \
    texlive-pstricks \
    latexmk \
    xindy \
    fonts-lmodern \
    fontconfig

# Additional tools for full functionality
docker exec sharelatex apt-get install -y \
    gnuplot \
    python3-pygments \
    python3-matplotlib \
    asymptote \
    inkscape
```

### TeX Live Installation Persistence

To avoid losing the complete TeX Live installation during updates or reboots, create a custom image:

```bash
# Create an image with a custom name
docker commit sharelatex sharelatex/sharelatex:with-texlive-full
```

This process will take a few minutes.

### Configuration to Use Custom Image

```bash
# Stop containers
./bin/stop

# Tag (create an alias for) the custom image with the original version name
docker tag sharelatex/sharelatex:with-texlive-full sharelatex/sharelatex:$(cat config/version)

# Restart containers
./bin/up -d
```

### Verify Complete Installation

```bash
# Check that the image size is correct (should be >10GB)
docker images
```

## Automation and Boot Start

### Create systemd Service

```bash
nano /etc/systemd/system/overleaf.service
```

File content:

```
[Unit]
Description=Overleaf Community Edition
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/overleaf
ExecStart=/opt/overleaf/bin/up -d
ExecStop=/opt/overleaf/bin/stop

[Install]
WantedBy=multi-user.target
```

### Enable Service

```bash
systemctl enable overleaf.service
```

## Backup and Maintenance

### Create Backup Script

```bash
mkdir -p /root/scripts
nano /root/scripts/backup-overleaf.sh
```

Script content:

```bash
#!/bin/bash
BACKUP_DIR="/root/overleaf-backups"
DATE=$(date +%Y-%m-%d)
mkdir -p $BACKUP_DIR

# Backup MongoDB
cd /opt/overleaf
./bin/docker-compose exec -T mongo mongodump --archive > $BACKUP_DIR/mongo-$DATE.archive

# Backup files
tar -czf $BACKUP_DIR/overleaf-data-$DATE.tar.gz /opt/overleaf/data

# Keep only the last 7 backups
find $BACKUP_DIR -name "mongo-*.archive" -type f -mtime +7 -delete
find $BACKUP_DIR -name "overleaf-data-*.tar.gz" -type f -mtime +7 -delete
```

Make the script executable:

```bash
chmod +x /root/scripts/backup-overleaf.sh
```

### Configure Automatic Backup

```bash
crontab -e
```

Add this line for a daily backup at 3 AM:

```
0 3 * * * /root/scripts/backup-overleaf.sh
```

## Security and Production

### Compilation in Separate Containers (Optional but Recommended)

For a more secure environment, enable compilation in separate containers:

```bash
# Edit configuration
nano /opt/overleaf/config/overleaf.rc

# Set
SIBLING_CONTAINERS_ENABLED=true

# Restart
cd /opt/overleaf
./bin/stop
./bin/up -d
```

### HTTPS Configuration (for Production Environments)

For a production environment, it's highly recommended to configure HTTPS using Let's Encrypt or another SSL certificate provider.

HTTPS configuration can be done following the official Overleaf documentation.

## Troubleshooting

### Problem: Container Not Reachable from Outside

**Symptoms**: Overleaf service is running but not accessible from outside.

**Solutions**:

1. Check the listening IP configuration:
   ```bash
   grep LISTEN_IP /opt/overleaf/config/overleaf.rc
   ```
   It should be `0.0.0.0`.

2. Check the firewall:
   ```bash
   ufw status
   ```
   Port 80 must be open.

3. Check logs:
   ```bash
   docker logs sharelatex
   ```

4. Check port mapping:
   ```bash
   docker ps
   ```
   It should show `0.0.0.0:80->80/tcp` for the sharelatex container.

### Problem: Error When Using Custom Image

**Symptoms**: Error "manifest for sharelatex/sharelatex:xxx-with-texlive-full not found"

**Solution**:
```bash
# Check available images
docker images

# Create a tag corresponding to the required version
docker tag sharelatex/sharelatex:with-texlive-full sharelatex/sharelatex:$(cat /opt/overleaf/config/version)

# Restart
cd /opt/overleaf
./bin/up -d
```

### Problem: Activation Link with "localhost"

**Symptoms**: User activation links contain "localhost" instead of the public IP.

**Solution**:
```bash
# Check site URL configuration
grep SITE_URL /opt/overleaf/config/overleaf.rc

# Fix if necessary
nano /opt/overleaf/config/overleaf.rc
# Set OVERLEAF_SITE_URL=http://YOUR-VPS-IP

# Restart
cd /opt/overleaf
./bin/stop
./bin/up -d
```

### Problem: MongoDB Won't Start Due to Lack of AVX Support

**Symptoms**: 
- "Illegal instruction" error in MongoDB logs
- MongoDB container starts but immediately terminates
- "Cannot connect to database" error in Overleaf logs

**Verify**:
```bash
# Check if CPU supports AVX
grep -q avx /proc/cpuinfo && echo "AVX supported" || echo "AVX NOT supported"

# Check MongoDB logs
docker logs mongo
```

**Solution**: 
The only solution is to use a server with a CPU that supports the AVX instruction set. MongoDB 5.0+ requires this instruction to function correctly. You'll need to migrate to a VPS with newer hardware.

### Problem: Incorrect Hostname/IP in Access Links

**Symptoms**: 
- Activation or access links contain an incorrect hostname (localhost or wrong IP)
- Users receive links that don't work

**Solution**:
```bash
# Check site URL configuration
grep SITE_URL /opt/overleaf/config/overleaf.rc

# Fix by setting the correct IP or domain
nano /opt/overleaf/config/overleaf.rc
# Edit the line with OVERLEAF_SITE_URL or SHARELATEX_SITE_URL

# Restart Overleaf
cd /opt/overleaf
./bin/stop
./bin/up -d
```

For links already sent, you can manually modify the URL replacing "localhost" or the incorrect IP with the correct one.

---

This guide was created to install and configure Overleaf Community Edition on a VPS. For questions, issues, or updates, consult the official Overleaf documentation or open an issue on GitHub.
