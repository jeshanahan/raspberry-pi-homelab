# Architecture

This document describes the overall architecture of the Raspberry Pi homelab, including the media request pipeline, container relationships, storage flow, networking model, and major design decisions.

For deeper implementation details, see:

- [Storage](storage.md)
- [Networking](networking.md)
- [Troubleshooting](troubleshooting.md)

---

## System Overview

The homelab runs on a Raspberry Pi 4 using Ubuntu Server and Docker Compose.

Most services are deployed as containers within a single Docker Compose stack. The media pipeline is centered around Jellyfin, with Sonarr and Radarr managing media acquisition and organization.

The system is designed around several goals:

- Keep services isolated using Docker
- Keep torrent traffic behind a VPN
- Store large media files on external storage
- Avoid unnecessary file duplication
- Allow secure remote administration
- Automate media requests, importing, and subtitle management
- Minimize dependence on the Raspberry Pi microSD card

---

## High-Level Architecture

```text
                              User
                               |
                               v
                             Seerr
                         /             \
                        v               v
                     Sonarr           Radarr
                        \               /
                         \             /
                          v           v
                            Prowlarr
                               |
                               v
                         qBittorrent
                               |
                     network_mode:
                    service:gluetun
                               |
                               v
                            Gluetun
                               |
                    ProtonVPN / WireGuard
                               |
                            Internet


Completed downloads:

 /data/torrents/tv                 /data/torrents/movies
          |                                  |
          v                                  v
       Sonarr                              Radarr
          |                                  |
          v                                  v
 /data/media/tv                    /data/media/movies
          \                                  /
           \                                /
            +------------+-----------------+
                         |
                         v
                      Jellyfin
                         |
               +---------+---------+
               |                   |
        Local Network          Tailscale
      rasp4serv.local          rasp4serv
```

---

## Container Layout

The primary media services run inside the same Docker Compose project.

| Service | Purpose |
|---|---|
| Jellyfin | Media playback and library management |
| Seerr | User-facing media request interface |
| Sonarr | TV and anime management |
| Radarr | Movie management |
| Prowlarr | Centralized indexer management |
| Bazarr | Subtitle management |
| qBittorrent | Torrent download client |
| Gluetun | VPN gateway for qBittorrent |
| Portainer | Docker administration |

Most containers communicate over the Compose project's default Docker bridge network using Docker DNS.

Examples:

```text
Seerr  -> http://sonarr:8989
Seerr  -> http://radarr:7878
Bazarr -> http://sonarr:8989
Bazarr -> http://radarr:7878
```

Using container names avoids depending on the Raspberry Pi's LAN IP for communication between services.

---

## Media Request Flow

Seerr provides the main request interface.

### Movies

A movie request follows this path:

```text
Seerr
  |
  v
Radarr
  |
  v
Prowlarr / Indexers
  |
  v
qBittorrent
```

Radarr monitors the download and imports the completed file into the movie library.

### TV and Anime

A TV or anime request follows:

```text
Seerr
  |
  v
Sonarr
  |
  v
Prowlarr / Indexers
  |
  v
qBittorrent
```

Sonarr monitors the download and imports completed episodes into the TV library.

Seerr does not download media itself. It acts as a request and orchestration layer above Sonarr and Radarr.

---

## Download Architecture

qBittorrent is deliberately isolated from the host's normal network connection.

Its Docker configuration uses:

```yaml
network_mode: "service:gluetun"
```

Instead of having its own independent Docker network interface, qBittorrent shares Gluetun's network namespace.

The effective network path is:

```text
qBittorrent
     |
     v
  Gluetun
     |
     v
WireGuard
     |
     v
 ProtonVPN
     |
     v
 Internet
```

Gluetun also provides firewall and kill-switch behavior so that qBittorrent cannot simply fall back to the Raspberry Pi's normal Internet connection if the VPN becomes unavailable.

ProtonVPN dynamic port forwarding is integrated with qBittorrent so the torrent client's listening port can be updated to match the port assigned by the VPN provider.

More details are documented in [Networking](networking.md).

---

## Storage Architecture

The external media drive is mounted on the host at:

```text
/mnt/media
```

The primary media data structure is:

```text
/mnt/media/data/
├── torrents/
│   ├── incomplete/
│   ├── movies/
│   └── tv/
└── media/
    ├── movies/
    └── tv/
```

Containers that work with media use a consistent mapping:

```text
Host:
/mnt/media/data

Container:
/data
```

For example:

```text
Host                           Container

/mnt/media/data/torrents/tv -> /data/torrents/tv
/mnt/media/data/media/tv    -> /data/media/tv
```

Using the same `/data` path across qBittorrent, Sonarr, and Radarr allows completed downloads to be imported using hardlinks where possible.

This avoids creating an unnecessary second physical copy of a media file.

See [Storage](storage.md) for filesystem, mount, and hardlink details.

---

## Import Flow

After qBittorrent finishes a TV download:

```text
/data/torrents/tv
        |
        v
      Sonarr
        |
        | rename / organize / hardlink
        v
/data/media/tv
        |
        v
     Jellyfin
```

Movies follow the same process through Radarr:

```text
/data/torrents/movies
        |
        v
      Radarr
        |
        | rename / organize / hardlink
        v
/data/media/movies
        |
        v
     Jellyfin
```

The torrent directory and media library remain separate even though both reside on the same filesystem.

This allows qBittorrent to continue managing torrent data while Sonarr and Radarr maintain clean library structures for Jellyfin.

---

## Subtitle Architecture

Bazarr integrates with Sonarr and Radarr and monitors the existing media libraries for missing subtitles.

```text
Sonarr / Radarr
      |
      v
    Bazarr
      |
      v
Subtitle Providers
      |
      v
External subtitle files
      |
      v
/data/media/...
```

Downloaded subtitles are stored alongside their corresponding media files.

Example:

```text
Show Name/
└── Season 01/
    ├── Show Name - S01E01.mkv
    └── Show Name - S01E01.en.srt
```

Jellyfin can then detect and present these subtitle tracks during playback.

Jellyfin's Subtitle Extract plugin is separate from Bazarr. It extracts subtitle tracks already embedded inside media files.

Because extracted subtitle data can become large, its storage is mapped to the external drive rather than the microSD card.

---

## Jellyfin

Jellyfin provides the playback layer of the system.

Its media mount is read-only:

```text
/mnt/media/data/media -> /media
```

Jellyfin reads:

```text
/media/movies
/media/tv
```

Sonarr and Radarr are responsible for modifying and organizing the media directories.

Keeping Jellyfin's media mount read-only reduces the risk of accidental library modification from the playback service.

---

## Local and Remote Access

The Raspberry Pi has two main hostname mechanisms.

### Local Network

Avahi provides mDNS:

```text
rasp4serv.local
```

This resolves to the Raspberry Pi's local Ethernet address.

### Remote Access

Tailscale provides private remote connectivity and MagicDNS:

```text
rasp4serv
```

This allows services and SSH to be reached remotely without exposing administrative ports directly to the public Internet.

Examples:

```text
http://rasp4serv:8096    Jellyfin
http://rasp4serv:8989    Sonarr
http://rasp4serv:7878    Radarr
http://rasp4serv:6767    Bazarr
http://rasp4serv:5055    Seerr
http://rasp4serv:9000    Portainer
```

Remote SSH access is also available:

```bash
ssh <user>@rasp4serv
```

---

## Design Decisions

### Shared `/data` Filesystem

Sonarr, Radarr, and qBittorrent all use the same internal `/data` mapping.

This:

- simplifies paths between applications
- avoids remote path mappings
- allows hardlinks
- reduces unnecessary disk usage

### VPN Isolation

Only qBittorrent is routed through Gluetun.

Applications such as Jellyfin, Sonarr, Radarr, Seerr, and Bazarr retain normal network access.

This prevents the VPN from unnecessarily affecting unrelated services.

### External Media Storage

Large and frequently changing data is kept off the microSD card whenever possible.

The external drive stores:

- media
- torrents
- extracted Jellyfin subtitles
- other large application-generated data where appropriate

### Private Remote Access

Tailscale was chosen instead of exposing administrative applications through router port forwarding.

This keeps services such as Sonarr, Radarr, Portainer, and qBittorrent private.

### Docker DNS

Containers communicate using service names such as:

```text
sonarr
radarr
jellyfin
```

instead of LAN IP addresses.

This keeps internal service communication independent of DHCP or host IP changes.

---

## Failure Boundaries

The architecture separates several components so that failures can be isolated.

For example:

```text
Indexer failure
    -> Sonarr/Radarr may not find releases
    -> Jellyfin continues working

VPN failure
    -> qBittorrent stops functioning
    -> Jellyfin remains available

Bazarr failure
    -> automatic subtitle downloads stop
    -> existing media continues playing

Tailscale failure
    -> remote access becomes unavailable
    -> local access remains available
```

The external media drive is a more significant dependency because Sonarr, Radarr, qBittorrent, and Jellyfin all depend on it.

Docker bind mounts therefore use safeguards to avoid silently creating media directories on the microSD card if the external filesystem is unavailable.

---

## Related Documentation

- [Storage](storage.md)
- [Networking](networking.md)
- [Troubleshooting](troubleshooting.md)
