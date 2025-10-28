# Config Backup

This directory contains backup copies of configuration files from `~/Documents/torrentbox-config/`.

## Purpose

These configs are backed up in the repo so they can be:
- Version controlled
- Easily restored if needed
- Referenced when setting up on a new machine

## Contents

### qBittorrent
- `qBittorrent.conf` - Main qBittorrent configuration
- `qBittorrent-data.conf` - Additional qBittorrent settings
- `categories.json` - Download categories
- `watched_folders.json` - Folders monitored for new torrents
- `rss/feeds.json` - RSS feed subscriptions

### Gluetun
- `servers.json` - VPN server configurations

## Notes

- These are BACKUPS only - the actual configs used by Docker are in `~/Documents/torrentbox-config/`
- Logs, lock files, and torrent data are excluded from backups
- Update these backups after making significant configuration changes
