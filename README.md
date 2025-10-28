# Docker Torrent Box with Surfshark VPN

A secure Docker-based torrenting setup using qBittorrent routed through Surfshark VPN with automatic kill switch.

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Initial Setup](#initial-setup)
- [Usage](#usage)
- [Configuration Management](#configuration-management)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Maintenance](#maintenance)

## Features

- **qBittorrent** - Modern web-based torrent client
- **Surfshark VPN** - All traffic routed through VPN
- **Kill Switch** - No internet access if VPN disconnects
- **Auto-restart** - Containers restart automatically on boot
- **Web UI** - Manage torrents from your browser
- **Port Forwarding** - NAT-PMP enabled (limited Surfshark support)

## Architecture

This setup uses two containers:

1. **Gluetun** - VPN client that handles Surfshark connection
2. **qBittorrent** - Torrent client that routes all traffic through Gluetun's network

If the VPN connection drops, qBittorrent loses all internet access (kill switch).

## Prerequisites

- Docker and Docker Compose installed
- Surfshark VPN subscription
- Surfshark VPN credentials (see setup below)

## Quick Start

For experienced users who want to get started immediately:

```bash
# 1. Clone the repository
git clone https://github.com/imgntn/torrentbox_docker.git
cd torrentbox_docker

# 2. Copy environment file and configure credentials
cp .env.example .env
nano .env  # Add your Surfshark credentials

# 3. Create config directory
mkdir -p ~/Documents/torrentbox-config/qbittorrent ~/Documents/torrentbox-config/gluetun

# 4. Start containers
docker compose up -d

# 5. Check logs to verify VPN connection
docker compose logs -f gluetun

# 6. Access Web UI at http://localhost:9469
# Default user: admin, password: check logs with:
docker compose logs qbittorrent | grep "password"
```

For detailed instructions, continue reading below.

## Initial Setup

### 1. Get Surfshark Credentials

Choose one of the following methods:

#### Option A: WireGuard (Recommended - Faster)

1. Login to [Surfshark Account](https://my.surfshark.com)
2. Navigate to **VPN** → **Manual Setup** → **WireGuard**
3. Click **Generate configuration**
4. Note down:
   - **Private Key** (long string starting with letters/numbers)
   - **Address** (usually `10.14.0.2/16`)

#### Option B: OpenVPN (More Compatible)

1. Login to [Surfshark Account](https://my.surfshark.com)
2. Navigate to **VPN** → **Manual Setup** → **OpenVPN**
3. Note down your **Service credentials**:
   - **Username** (different from your account email)
   - **Password** (different from your account password)

### 2. Configure Environment Variables

1. Copy the example environment file:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` file:
   ```bash
   nano .env  # or use your preferred editor
   ```

3. Fill in your credentials based on your chosen method:

   **For WireGuard:**
   ```env
   VPN_TYPE=wireguard
   WIREGUARD_PRIVATE_KEY=your_actual_private_key_here
   WIREGUARD_ADDRESSES=10.14.0.2/16
   SERVER_COUNTRIES=Netherlands
   ```

   **For OpenVPN:**
   ```env
   VPN_TYPE=openvpn
   OPENVPN_USER=your_service_username
   OPENVPN_PASSWORD=your_service_password
   SERVER_COUNTRIES=Netherlands
   ```

4. (Optional) Customize other settings:
   - `TZ` - Your timezone
   - `PUID`/`PGID` - Your user/group IDs (run `id` command)
   - `SERVER_COUNTRIES` - Preferred VPN server location
   - `DOWNLOADS_PATH` - Where to save downloads

### 3. Create Required Directories

The directories will be created automatically when you start the containers, but you can create them manually if desired:

```bash
mkdir -p gluetun/config
mkdir -p qbittorrent/config
mkdir -p qbittorrent/downloads
```

### 4. Start the Containers

```bash
docker compose up -d
```

This will:
- Download the necessary Docker images (first time only)
- Start the Gluetun VPN container
- Start the qBittorrent container (routed through VPN)

### 5. Verify VPN Connection

Check the logs to ensure VPN is connected:

```bash
docker compose logs gluetun
```

Look for messages like:
- `[INFO] Connected to Surfshark VPN`
- `[INFO] ip: xxx.xxx.xxx.xxx` (should be VPN IP, not your real IP)

### 6. Access qBittorrent Web UI

1. Open your browser and navigate to: **http://localhost:9469**

2. Login with default credentials:
   - **Username:** `admin`
   - **Password:** Check the qBittorrent logs for temporary password:
     ```bash
     docker compose logs qbittorrent | grep "password"
     ```

3. **IMPORTANT:** Change the password immediately:
   - Go to **Tools** → **Options** → **Web UI**
   - Change the password under Authentication

## Usage

### Managing Containers

Start containers:
```bash
docker compose up -d
```

Stop containers:
```bash
docker compose down
```

Restart containers:
```bash
docker compose restart
```

View logs:
```bash
# All containers
docker compose logs -f

# Gluetun only
docker compose logs -f gluetun

# qBittorrent only
docker compose logs -f qbittorrent
```

### Verify Your IP is Hidden

Test that torrents use VPN IP:

1. Check your public IP through qBittorrent:
   - The Gluetun logs show your VPN IP when connected
   - Compare with your real IP (google "what is my ip")
   - They should be different!

2. Use a torrent IP checker:
   - Add this magnet link in qBittorrent: `magnet:?xt=urn:btih:dd8255ecdc7ca55fb0bbf81323d87062db1f6d1c`
   - Check results at: https://ipleak.net/
   - The IP shown should be your VPN IP, not your real IP

## Configuration Management

### Config File Locations

This setup stores configuration files in `~/Documents/torrentbox-config/` (outside the repository) to keep them persistent and separate from the code. The directory structure is:

```
~/Documents/torrentbox-config/
├── gluetun/
│   └── servers.json           # VPN server list (auto-generated)
└── qbittorrent/
    └── qBittorrent/
        ├── qBittorrent.conf   # Main settings (port, limits, etc.)
        ├── categories.json    # Download categories
        ├── watched_folders.json # Auto-import folders
        └── rss/
            └── feeds.json     # RSS feed subscriptions
```

### Backed Up Configs

The `config-backup/` directory in this repository contains version-controlled backups of important configuration files. These backups:

- **Are NOT used by the running containers** (they read from `~/Documents/torrentbox-config/`)
- Serve as reference configurations
- Can be used to restore settings after a fresh install
- Should be updated after making significant configuration changes

### Restoring from Backup

If you need to restore your configuration (e.g., on a new machine):

```bash
# 1. Ensure the config directory exists
mkdir -p ~/Documents/torrentbox-config/qbittorrent/qBittorrent/rss
mkdir -p ~/Documents/torrentbox-config/gluetun

# 2. Copy backed up configs
cp config-backup/qbittorrent/*.conf ~/Documents/torrentbox-config/qbittorrent/qBittorrent/
cp config-backup/qbittorrent/*.json ~/Documents/torrentbox-config/qbittorrent/qBittorrent/
cp config-backup/qbittorrent/rss/feeds.json ~/Documents/torrentbox-config/qbittorrent/qBittorrent/rss/
cp config-backup/gluetun/servers.json ~/Documents/torrentbox-config/gluetun/

# 3. Restart containers to apply
docker compose restart
```

### Updating Config Backups

After making important configuration changes in qBittorrent, update the backups:

```bash
# Copy current configs to backup directory
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/qBittorrent.conf config-backup/qbittorrent/
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/qBittorrent-data.conf config-backup/qbittorrent/
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/categories.json config-backup/qbittorrent/
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/watched_folders.json config-backup/qbittorrent/
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/rss/feeds.json config-backup/qbittorrent/rss/

# Commit to git
git add config-backup/
git commit -m "Update config backups"
git push
```

## Troubleshooting

### VPN Not Connecting

1. Check Gluetun logs:
   ```bash
   docker compose logs gluetun
   ```

2. Common issues:
   - **Invalid credentials:** Double-check your `.env` file
   - **Wrong VPN_TYPE:** Ensure it matches your credential type (wireguard/openvpn)
   - **Server issues:** Try different `SERVER_COUNTRIES`

3. Restart containers:
   ```bash
   docker compose restart
   ```

### Can't Access Web UI

1. Verify containers are running:
   ```bash
   docker compose ps
   ```

2. Check if port 9469 is in use:
   ```bash
   lsof -i :9469
   ```

3. Try accessing via IP:
   ```bash
   http://127.0.0.1:9469
   ```

### Downloads Not Starting

1. Check qBittorrent logs for errors:
   ```bash
   docker compose logs qbittorrent
   ```

2. Verify disk space:
   ```bash
   df -h
   ```

3. Check download path permissions:
   ```bash
   ls -la qbittorrent/downloads
   ```

### Port Forwarding Not Working

**Note:** Surfshark has limited port forwarding support. This setup attempts NAT-PMP but may not work consistently.

1. Check Gluetun logs for port forwarding status:
   ```bash
   docker compose logs gluetun | grep -i "port"
   ```

2. If port forwarding fails:
   - This is normal with Surfshark
   - You can still download torrents (just can't accept incoming connections)
   - For better port forwarding, consider VPN providers like ProtonVPN or PIA

3. Set the forwarded port in qBittorrent (if successful):
   - Check Gluetun logs for forwarded port number
   - In qBittorrent: **Tools** → **Options** → **Connection**
   - Set "Port used for incoming connections" to the forwarded port
   - Uncheck "Use UPnP / NAT-PMP"

### Containers Keep Restarting

If containers are in a restart loop:

```bash
# Check container status
docker compose ps

# View recent logs for errors
docker compose logs --tail=100

# Common causes:
# 1. Invalid VPN credentials - Check .env file
# 2. Port already in use - Check with: lsof -i :9469
# 3. Permission issues - Check volume permissions
# 4. Docker out of resources - Check with: docker system df
```

### Slow Download Speeds

1. Try different VPN servers:
   ```bash
   # Edit .env to change SERVER_COUNTRIES
   # Then restart
   docker compose restart
   ```

2. Check if VPN server is overloaded:
   ```bash
   docker compose logs gluetun | grep -i "latency"
   ```

3. Adjust qBittorrent connection settings:
   - **Tools** → **Options** → **Connection**
   - Increase "Number of connections globally" (default 500)
   - Increase "Number of connections per torrent" (default 100)

4. Use WireGuard instead of OpenVPN (faster):
   - Change `VPN_TYPE=wireguard` in `.env`
   - Get WireGuard credentials from Surfshark
   - Restart containers

### Permission Denied Errors

If you see permission errors in logs:

```bash
# Check your user/group IDs
id

# Update .env with your actual PUID and PGID
PUID=1000  # Your uid from above command
PGID=1000  # Your gid from above command

# Fix permissions on existing files
sudo chown -R $(id -u):$(id -g) ~/Documents/torrentbox-config

# Restart containers
docker compose restart
```

### Can't Push to GitHub (SSH Issues)

If you set up this repository and can't push via SSH:

```bash
# Test SSH connection
ssh -T git@github.com

# If it fails, add your SSH key
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# Copy your public key and add it to GitHub at:
# https://github.com/settings/keys
cat ~/.ssh/id_ed25519.pub
```

## File Locations

### Repository Structure

```
torrentbox_docker/
├── docker-compose.yml         # Container definitions
├── .env                       # Your credentials (NOT in git)
├── .env.example              # Template for credentials
├── README.md                 # This file
├── config-backup/            # Backed up configs (in git)
│   ├── qbittorrent/         # qBittorrent config backups
│   └── gluetun/             # Gluetun config backups
├── gluetun/                  # Runtime Gluetun data (NOT in git)
└── qbittorrent/             # Runtime qBittorrent data (NOT in git)
    └── downloads/           # Downloaded files
```

### Runtime Data (External)

Configuration files are stored outside the repository in:

```
~/Documents/torrentbox-config/
├── gluetun/                  # Gluetun VPN data
│   └── servers.json         # VPN server list
└── qbittorrent/             # qBittorrent data
    └── qBittorrent/
        ├── qBittorrent.conf # Main config
        ├── categories.json  # Download categories
        ├── rss/            # RSS feeds
        └── BT_backup/      # Torrent state files
```

This separation keeps your personal configs out of version control while still allowing you to back them up in `config-backup/`.

## Security Best Practices

1. **Never commit `.env` file** - Contains your VPN credentials
2. **Change default qBittorrent password** - Do this immediately after first login
3. **Use strong Web UI password** - In qBittorrent settings
4. **Don't expose publicly** - Only access from localhost/LAN unless you set up proper authentication
5. **Keep containers updated** - See [Updating](#updating) below
6. **Regular backups** - See [Backup](#backup) below
7. **Verify VPN connection** - Regularly check your IP isn't leaking
8. **Review qBittorrent logs** - Check for suspicious activity

## Maintenance

### Updating

To update containers to the latest versions:

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d

# Clean up old images (optional)
docker image prune -a
```

Your settings and downloads are preserved in the volume directories.

**Update Schedule Recommendation:** Check for updates monthly or when security issues are announced.

### Backup

#### Quick Backup (Configs Only)

Use the built-in config backup system:

```bash
# Update the backed up configs in the repository
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/qBittorrent.conf config-backup/qbittorrent/
cp ~/Documents/torrentbox-config/qbittorrent/qBittorrent/categories.json config-backup/qbittorrent/
# ... (see "Updating Config Backups" section above)

# Commit to git
git add config-backup/
git commit -m "Update config backups"
git push
```

#### Full Backup (Everything)

For complete disaster recovery:

```bash
# Backup runtime configs and data
tar -czf torrentbox-backup-$(date +%Y%m%d).tar.gz \
  ~/Documents/torrentbox-config \
  .env \
  gluetun/ \
  qbittorrent/downloads/

# Or exclude large download files
tar -czf torrentbox-backup-$(date +%Y%m%d).tar.gz \
  ~/Documents/torrentbox-config \
  .env
```

**Backup Schedule Recommendation:**
- Configs: After significant changes (automatic if you use git)
- Full backup: Weekly if actively torrenting

### Server Location Recommendations

Good server locations for torrenting:

- **Netherlands** - Fast, P2P-friendly laws
- **Switzerland** - Strong privacy laws
- **Spain** - P2P-friendly
- **Romania** - Good speeds, privacy-friendly

Avoid for torrenting:
- **United States** - DMCA notices
- **United Kingdom** - Strict copyright enforcement

### Performance Tips

To optimize download speeds and system resources:

1. **Use WireGuard over OpenVPN**
   - WireGuard is significantly faster
   - Set `VPN_TYPE=wireguard` in `.env`

2. **Choose optimal VPN servers**
   - Netherlands, Germany, Switzerland typically offer best speeds
   - Test different servers if experiencing slowness
   - Closer servers = lower latency = better performance

3. **Adjust qBittorrent Connection Settings**
   - **Tools** → **Options** → **Connection**
   - Global connections: 500-1000 (depending on your bandwidth)
   - Per-torrent connections: 100-200
   - Enable uTP for better performance

4. **Limit Active Torrents**
   - **Tools** → **Options** → **BitTorrent**
   - Set max active torrents: 5-10
   - Set max active downloads: 3-5
   - Prevents resource exhaustion

5. **Set Bandwidth Limits Appropriately**
   - **Tools** → **Options** → **Speed**
   - Upload limit: 80% of your upload bandwidth
   - Download limit: 90% of your download bandwidth
   - Prevents network congestion

6. **Monitor Resource Usage**
   ```bash
   # Real-time container stats
   docker stats

   # Check disk space
   df -h

   # View container resource limits
   docker compose ps
   ```

7. **Clean Up Completed Torrents**
   - Regularly remove completed torrents from qBittorrent
   - Keeps the interface responsive

### Monitoring

Check container health and performance:

```bash
# Container status
docker compose ps

# Live logs
docker compose logs -f

# Resource usage
docker stats gluetun qbittorrent

# Check VPN IP
docker compose logs gluetun | grep -i "ip"

# Network throughput
docker stats --no-stream --format "table {{.Name}}\t{{.NetIO}}"
```

## Additional Configuration

### Change Web UI Port

Edit `docker-compose.yml`:
```yaml
ports:
  - "9090:8080"  # Change 9090 to your preferred port
```

Then restart:
```bash
docker compose up -d
```

### Custom Download Path

Edit `.env`:
```env
DOWNLOADS_PATH=/path/to/your/downloads
```

### Enable Categories

In qBittorrent Web UI:
- Right-click in left sidebar → **Add category**
- Create categories like: Movies, TV, Music, Software
- Assign torrents to categories for organization

## Additional Resources

### Useful Links

- **This Repository:** https://github.com/imgntn/torrentbox_docker
- **qBittorrent Documentation:** https://github.com/qbittorrent/qBittorrent/wiki
- **Gluetun Documentation:** https://github.com/qdm12/gluetun/wiki
- **Surfshark Support:** https://support.surfshark.com
- **Docker Compose Reference:** https://docs.docker.com/compose/

### Getting Help

1. **Check the logs first:**
   ```bash
   docker compose logs
   ```

2. **Search existing issues:**
   - Gluetun: https://github.com/qdm12/gluetun/issues
   - qBittorrent: https://github.com/qbittorrent/qBittorrent/issues

3. **Common issues are covered in [Troubleshooting](#troubleshooting)**

4. **For Surfshark-specific VPN issues:** Contact Surfshark support

### Contributing

Feel free to submit issues or pull requests to improve this setup!

## License

This configuration is provided as-is for educational purposes. Use responsibly and in accordance with your local laws.

**Important:** Respect copyright laws. This tool is intended for legal torrenting of open-source software, public domain content, and content you have rights to download.

---

**Last Updated:** 2025-10-27
**Maintained by:** [@imgntn](https://github.com/imgntn)
