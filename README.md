# Docker Torrent Box with Surfshark VPN

A secure Docker-based torrenting setup using qBittorrent routed through Surfshark VPN with automatic kill switch.

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

## File Locations

- **Config:** `./qbittorrent/config` - qBittorrent settings, categories, RSS feeds
- **Downloads:** `./qbittorrent/downloads` - Downloaded files (configurable)
- **Gluetun Data:** `./gluetun` - VPN configuration and state

## Security Best Practices

1. **Never commit `.env` file** - Contains your VPN credentials
2. **Change default qBittorrent password** - Do this immediately after first login
3. **Keep containers updated:**
   ```bash
   docker compose pull
   docker compose up -d
   ```
4. **Use strong Web UI password** - In qBittorrent settings
5. **Don't expose publicly** - Only access from localhost/LAN

## Updating

To update to the latest versions:

```bash
# Pull latest images
docker compose pull

# Restart with new images
docker compose up -d
```

Your settings and downloads are preserved in the volume directories.

## Backup

To backup your configuration:

```bash
# Backup qBittorrent settings
tar -czf qbittorrent-backup-$(date +%Y%m%d).tar.gz qbittorrent/config

# Backup everything (including downloads)
tar -czf torrentbox-backup-$(date +%Y%m%d).tar.gz qbittorrent gluetun .env
```

## Server Location Recommendations

Good server locations for torrenting:

- **Netherlands** - Fast, P2P-friendly laws
- **Switzerland** - Strong privacy laws
- **Spain** - P2P-friendly
- **Romania** - Good speeds, privacy-friendly

Avoid for torrenting:
- **United States** - DMCA notices
- **United Kingdom** - Strict copyright enforcement

## Performance Tips

1. **Limit active torrents** - Too many can slow downloads
2. **Adjust upload/download limits** - In qBittorrent settings
3. **Choose fast servers** - Netherlands, Germany usually fast
4. **Use WireGuard** - Faster than OpenVPN
5. **Monitor resource usage:**
   ```bash
   docker stats
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

## Support

- **Gluetun Issues:** https://github.com/qdm12/gluetun
- **qBittorrent Issues:** https://github.com/qbittorrent/qBittorrent
- **Surfshark Support:** https://support.surfshark.com

## License

This configuration is provided as-is. Use responsibly and in accordance with your local laws.
