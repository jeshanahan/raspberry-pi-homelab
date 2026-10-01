# raspberry-pi-homelab
Documenting my Raspberry Pi homelab projects.

# Raspberry Pi Homelab

A self-hosted Linux homelab built on a Raspberry Pi 4 using Ubuntu Server, Docker, and Docker Compose.

The project began as a media server but evolved into a larger exercise in Linux system administration, container networking, storage management, VPN routing, remote access, automation, and troubleshooting.

## Overview

The server hosts a containerized media automation stack built around Jellyfin. Media requests can be submitted through Seerr, automatically processed by Sonarr or Radarr, downloaded through qBittorrent, and then imported into the Jellyfin media library.

Torrent traffic is isolated through a Gluetun container connected to ProtonVPN using WireGuard.

Remote administration is provided through Tailscale, allowing secure access to the server without exposing administrative services directly to the public Internet. This makes it best for personal use.

## Hardware

- Raspberry Pi 4 Model B (4GB RAM)
- 32GB microSD card for Ubuntu Server and application configuration
- 2 TB Seagate external USB drive for media and downloads

## Software

### Host

- Ubuntu Server
- Docker
- Docker Compose
- Portainer
- Tailscale
- Avahi / mDNS

### Media Stack

- Jellyfin — media server
- Sonarr — TV/anime management
- Radarr — movie management
- Bazarr — subtitle management
- Seerr — media request interface
- Prowlarr — indexer management
- qBittorrent — download client
- Gluetun — VPN gateway for qBittorrent

## Architecture

```text
                           Internet
                              |
                              |
                    ProtonVPN / WireGuard
                              |
                           Gluetun
                              |
                         qBittorrent
                              |
                 /data/torrents/{tv,movies}
                              |
                     +--------+--------+
                     |                 |
                   Sonarr            Radarr
                     |                 |
                     +--------+--------+
                              |
                   /data/media/{tv,movies}
                              |
                           Jellyfin
                              |
                  +-----------+-----------+
                  |                       |
             Local Network             Tailscale
           rasp4serv.local             rasp4serv
