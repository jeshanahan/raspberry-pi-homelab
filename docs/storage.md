# Storage

This document describes the storage architecture of the Raspberry Pi homelab, including the external media drive, filesystem layout, Docker bind mounts, hardlinks, SD-card usage, and safeguards against accidental writes to the wrong disk.

For the overall system design, see [Architecture](architecture.md).

For storage-related incidents and recovery procedures, see [Troubleshooting](troubleshooting.md).

---

## Storage Overview

The server uses two primary storage devices:

| Device | Purpose |
|---|---|
| 32 GB microSD card | Ubuntu Server, Docker, container configuration, application databases |
| 2 TB Seagate external USB drive | Media, torrent downloads, extracted subtitles, other large data |

The design intentionally keeps large and frequently changing media data off the microSD card.

This reduces:

- SD-card wear
- risk of filling the root filesystem
- unnecessary write load on the boot device
- difficulty recovering media after an operating system reinstall

---

## External Media Drive

The external drive is formatted as `ext4`.

It is mounted at:

```text
/mnt/media
```

The drive is referenced by filesystem UUID rather than a device name such as:

```text
/dev/sda2
```

or:

```text
/dev/sdb2
```

Linux device names can change between boots or USB reconnects (i.e. from sda2 to sdb2), while the filesystem UUID remains stable.

The `/etc/fstab` configuration follows this pattern:

```text
UUID=<MEDIA_DISK_UUID> /mnt/media ext4 defaults,nofail 0 2
```

The `nofail` option allows the server to continue booting if the external drive is temporarily unavailable.

The mount can be verified with:

```bash
findmnt /mnt/media
```

and:

```bash
df -hT /mnt/media
```

A healthy mount should show the external `ext4` filesystem rather than the microSD root filesystem.

---

## Directory Layout

The main data structure is:

```text
/mnt/media/
├── data/
│   ├── torrents/
│   │   ├── incomplete/
│   │   ├── movies/
│   │   └── tv/
│   └── media/
│       ├── movies/
│       └── tv/
└── appdata/
    └── jellyfin/
        └── subtitles/
```

The separation between `torrents` and `media` is intentional.

qBittorrent owns the download directories, while Sonarr and Radarr manage the organized media library.

---

## Shared `/data` Mapping

qBittorrent, Sonarr, Radarr, and other media-management containers use the same host path:

```text
/mnt/media/data
```

mapped inside containers as:

```text
/data
```

For example:

```text
Host                                 Container

/mnt/media/data/torrents/tv       -> /data/torrents/tv
/mnt/media/data/torrents/movies   -> /data/torrents/movies
/mnt/media/data/media/tv          -> /data/media/tv
/mnt/media/data/media/movies      -> /data/media/movies
```

A typical Docker Compose bind mount looks like:

```yaml
volumes:
  - type: bind
    source: /mnt/media/data
    target: /data
    bind:
      create_host_path: false
```

Using the same `/data` path across containers avoids unnecessary path translation and allows hardlinks to work correctly.

---

## Torrent Download Layout

qBittorrent stores downloads under:

```text
/data/torrents
```

Categories separate TV and movie downloads:

```text
/data/torrents/tv
/data/torrents/movies
```

Incomplete downloads can be stored separately:

```text
/data/torrents/incomplete
```

The equivalent host paths are:

```text
/mnt/media/data/torrents/tv
/mnt/media/data/torrents/movies
/mnt/media/data/torrents/incomplete
```

Sonarr and Radarr monitor completed downloads through the qBittorrent API.

---

## Media Library Layout

Sonarr imports TV and anime into:

```text
/data/media/tv
```

Radarr imports movies into:

```text
/data/media/movies
```

On the host, these correspond to:

```text
/mnt/media/data/media/tv
/mnt/media/data/media/movies
```

A typical TV library may look like:

```text
/data/media/tv/
└── Show Name/
    ├── Season 01/
    │   ├── Show Name - S01E01.mkv
    │   └── Show Name - S01E02.mkv
    └── Season 02/
```

Jellyfin reads from this organized media library.

---

## Hardlinks

One of the main reasons for using a shared `/data` filesystem is to support hardlinks.

When Sonarr or Radarr imports a completed torrent, the same underlying file can appear in both:

```text
/data/torrents/...
```

and:

```text
/data/media/...
```

without storing two complete physical copies.

Conceptually:

```text
/data/torrents/tv/show/episode.mkv
             |
             | same filesystem data
             |
/data/media/tv/Show/Season 01/episode.mkv
```

This allows:

- qBittorrent to continue seeding the original torrent
- Sonarr or Radarr to maintain a clean library structure
- Jellyfin to access organized media
- disk space to be conserved

Hardlinks require both paths to exist on the same filesystem.

Using separate Docker mounts such as:

```text
/downloads
/tv
/movies
```

can make hardlinking more difficult because applications may treat the paths as separate filesystems.

The shared `/data` design avoids that problem.

---

## Jellyfin Media Mount

Jellyfin receives the organized media library separately:

```text
/mnt/media/data/media -> /media
```

The mount is read-only:

```yaml
volumes:
  - type: bind
    source: /mnt/media/data/media
    target: /media
    read_only: true
    bind:
      create_host_path: false
```

Jellyfin then sees:

```text
/media/movies
/media/tv
```

Sonarr and Radarr remain responsible for modifying and organizing the library.

Giving Jellyfin read-only access reduces the chance of accidental modification from the playback layer.

---

## Jellyfin Extracted Subtitles

The Jellyfin Subtitle Extract plugin can generate a large amount of extracted subtitle data.

Originally, extracted subtitles were stored inside Jellyfin's normal configuration directory on the microSD card:

```text
/config/data/subtitles
```

This eventually consumed approximately 17 GB and filled the root filesystem.

To prevent this from happening again, the subtitle directory is stored on the external drive:

```text
/mnt/media/appdata/jellyfin/subtitles
```

and mounted into Jellyfin as:

```text
/config/data/subtitles
```

Example Compose configuration:

```yaml
volumes:
  - /srv/media-stack/jellyfin/config:/config
  - /srv/media-stack/jellyfin/cache:/cache

  - type: bind
    source: /mnt/media/appdata/jellyfin/subtitles
    target: /config/data/subtitles
    bind:
      create_host_path: false
```

This keeps Jellyfin's primary database and configuration on the system disk while moving potentially large extracted subtitle data to the external drive.

Bazarr subtitles are separate from this directory.

Bazarr stores downloaded subtitle files directly alongside media:

```text
/data/media/tv/Show Name/Season 01/
├── Show Name - S01E01.mkv
└── Show Name - S01E01.en.srt
```

---

## Preventing Accidental SD-Card Writes

A significant risk occurs when `/mnt/media` is not mounted.

Without safeguards, `/mnt/media` still exists as a normal directory on the root filesystem.

A container could therefore write:

```text
/mnt/media/data/torrents
```

to the microSD card without realizing that the external filesystem is missing.

To reduce this risk, storage-related Docker bind mounts use:

```yaml
bind:
  create_host_path: false
```

This prevents Docker from automatically creating a missing host directory.

If the expected directory does not exist, container creation should fail rather than silently creating the path on the SD card.

---

## Verifying the Media Drive

Before starting or troubleshooting storage-dependent services, the following commands are useful:

```bash
findmnt /mnt/media
```

```bash
df -hT /mnt/media
```

```bash
lsblk -f
```

A healthy system should show:

```text
External ext4 filesystem
        |
        v
   /mnt/media
        |
        v
 /mnt/media/data
```

If:

```bash
findmnt /mnt/media
```

returns nothing, the media drive should be investigated before restarting qBittorrent, Sonarr, Radarr, or Jellyfin.

---

## SD-Card Usage

The microSD card stores:

```text
Ubuntu Server
Docker
Container configuration
Application databases
Logs
Small caches
```

The external drive stores:

```text
Media
Torrents
Large subtitle extraction data
Other large application-generated files
```

Root filesystem usage can be monitored with:

```bash
df -h /
```

Large directories can be identified with:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

Application configuration sizes can be checked with:

```bash
sudo du -sh /srv/media-stack/* 2>/dev/null | sort -h
```

---

## Permissions

Media directories are owned by the primary server user and containers generally run using UID/GID `1000`.

For LinuxServer containers, this is configured using:

```yaml
environment:
  - PUID=1000
  - PGID=1000
```

Keeping ownership consistent simplifies shared access between:

```text
qBittorrent
Sonarr
Radarr
Bazarr
```

---

## Backup Considerations

The media library itself is large and can often be recreated, but container configuration is comparatively small and important.

High-value paths include:

```text
/srv/media-stack/sonarr
/srv/media-stack/radarr
/srv/media-stack/prowlarr
/srv/media-stack/bazarr
/srv/media-stack/seerr
/srv/media-stack/qbittorrent
/srv/media-stack/gluetun
/srv/media-stack/jellyfin/config
```

Future backup work should prioritize:

- Docker Compose files
- `.env` files stored securely
- application databases
- configuration directories
- custom scripts
- Jellyfin metadata where appropriate

Backups should ideally exist on a device other than the Raspberry Pi itself.

---

## Design Summary

The storage architecture is designed around four main principles:

1. **Large data belongs on the external drive**
2. **Containers use consistent paths**
3. **Hardlinks avoid unnecessary duplication**
4. **A missing external mount should fail safely instead of filling the SD card**

These decisions allow the Raspberry Pi to operate as a lightweight application host while the external drive handles the storage-intensive parts of the media stack.

---

## Related Documentation

- [Architecture](architecture.md)
- [Networking](networking.md)
- [Troubleshooting](troubleshooting.md)
