# Troubleshooting

This document records significant problems encountered while building and operating the Raspberry Pi homelab.

Rather than serving as a complete command history, each section documents:

```text
Symptoms
    ->
Investigation
    ->
Root Cause
    ->
Resolution
    ->
Prevention / Lessons Learned
```

This provides a record of both technical problems and the reasoning used to solve them.

---

# 1. External USB Drive Instability

## Symptoms

During initial setup and filesystem formatting, the external Seagate USB drive became unresponsive.

Kernel logs included errors involving:

```text
USB disconnect
I/O errors
UAS abort handlers
USB resets
xHCI host controller not responding
```

The drive would disappear or stop responding during sustained disk activity.

---

## Investigation

The external enclosure was identified as a Seagate device.

Kernel logs showed that Linux was using the UAS driver for the USB storage device.

During sustained I/O, the UAS connection experienced repeated errors and eventually caused the USB host controller to fail.

This indicated that the problem was below the Docker or application layer.

---

## Root Cause

The Seagate USB storage device was unstable when using Linux UAS.

The issue appeared under sustained I/O rather than simple file access.

---

## Resolution

UAS was disabled specifically for the affected USB device using a kernel USB-storage quirk.

The kernel command line was configured with a value following this pattern:

```text
usb-storage.quirks=<VID>:<PID>:u
```

After rebooting, the device used the traditional `usb-storage` driver instead of UAS.

A sustained disk read test was then performed.

The drive remained stable.

---

## Lessons Learned

Application failures can originate far below the application layer.

Useful diagnostic tools included:

```bash
dmesg
lsusb
lsblk
```

Kernel logs were essential for distinguishing a Docker problem from a USB controller or storage-driver problem.

---

# 2. External Drive Failed to Mount

## Symptoms

Sonarr reported:

```text
Missing root folder: /data/media/tv
```

and:

```text
qBittorrent places downloads in /data/torrents/tv
but this directory does not appear to exist inside the container
```

The media applications could see `/data`, but expected media and torrent directories were missing.

---

## Investigation

The first check was:

```bash
findmnt /mnt/media
```

It returned no output.

The drive was then checked with:

```bash
lsblk -f
```

The external `ext4` partition was still detected and its filesystem UUID matched the `/etc/fstab` entry.

This showed that the drive itself was present, but it had not been mounted.

---

## Root Cause

The external media filesystem had failed to mount at:

```text
/mnt/media
```

The containers therefore could not access the expected host directories.

---

## Resolution

The drive was mounted using:

```bash
sudo mount /mnt/media
```

The mount was verified with:

```bash
findmnt /mnt/media
```

and:

```bash
df -hT /mnt/media
```

The expected directory structure was confirmed:

```text
/mnt/media/data/media
/mnt/media/data/torrents
```

Storage-dependent containers were then restarted so their bind mounts referenced the restored filesystem.

---

## Prevention

Docker storage mounts use:

```yaml
bind:
  create_host_path: false
```

This reduces the chance of Docker silently creating replacement directories on the microSD card if the external drive is missing.

The external filesystem is referenced by UUID in `/etc/fstab` rather than a device name such as `/dev/sda2`.

---

## Lessons Learned

A valid directory path does not guarantee that the expected filesystem is mounted underneath it.

For storage-related failures, the host mount should be verified before troubleshooting Sonarr, Radarr, or qBittorrent.

---

# 3. Raspberry Pi Became Partially Unresponsive During Heavy Activity

## Symptoms

While torrents and media services were active:

- web applications stopped loading
- SSH established a TCP connection but never displayed the remote SSH banner
- ping continued responding
- the Raspberry Pi activity LED remained continuously active

The server was therefore reachable at the network layer but unable to service normal application requests.

---

## Investigation

Ping tests showed that Ethernet connectivity remained active.

Verbose SSH output showed:

```text
Connection established
```

but the SSH server never completed the protocol handshake.

This indicated that the operating system was partially alive but processes were likely blocked or under severe resource or I/O pressure.

---

## Possible Causes Investigated

The main suspects included:

```text
USB storage stalls
filesystem I/O
high torrent activity
Docker resource pressure
SD-card exhaustion
```

Previous USB instability made storage I/O an important area to investigate.

---

## Recovery

When normal access could not be restored, the server was eventually rebooted.

After recovery, kernel and system logs were inspected for:

```text
I/O errors
USB resets
xHCI failures
filesystem errors
OOM events
blocked tasks
```

---

## Lessons Learned

A system can remain responsive to ICMP ping while application processes are effectively frozen.

Network availability alone does not prove that the operating system is healthy.

---

# 4. Jellyfin Subtitle Extraction Filled the SD Card

## Symptoms

Sonarr reported that the root filesystem had no free space.

The Raspberry Pi's microSD card showed:

```text
100% used
```

even though the external media drive still had significant free capacity.

---

## Investigation

Filesystem usage was checked with:

```bash
df -h /
```

Large top-level directories were identified using:

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

The `/srv` directory was unexpectedly large.

Further inspection showed that Jellyfin was using approximately 17 GB inside:

```text
/srv/media-stack/jellyfin/config/data/subtitles
```

---

## Root Cause

The Jellyfin Subtitle Extract plugin had generated a large collection of extracted subtitle files.

Because Jellyfin's `/config` directory was stored on the microSD card, the extracted subtitles were also stored there.

The subtitle data eventually exhausted the root filesystem.

---

## Resolution

Jellyfin was stopped and the generated subtitle extraction data was cleared.

This immediately recovered approximately 17 GB of space.

The extracted subtitle directory was then moved to the external drive.

Host location:

```text
/mnt/media/appdata/jellyfin/subtitles
```

Container location:

```text
/config/data/subtitles
```

Example nested Docker bind mount:

```yaml
- type: bind
  source: /mnt/media/appdata/jellyfin/subtitles
  target: /config/data/subtitles
  bind:
    create_host_path: false
```

Jellyfin could then regenerate extracted subtitles without consuming the microSD card.

---

## Lessons Learned

Application configuration directories may contain large generated data even when they are described as "config" storage.

Disk usage should be monitored at both the filesystem and application-directory level.

---

# 5. Sonarr Could Not Find Media Paths

## Symptoms

Sonarr displayed errors such as:

```text
Missing root folder: /data/media/tv
```

and:

```text
Download client qBittorrent places downloads in /data/torrents/tv
but this directory does not appear to exist inside the container
```

---

## Investigation

The host filesystem was checked first.

The external drive mount was missing, which meant the expected `/mnt/media/data` structure was unavailable to Docker.

After remounting the disk, the directories were confirmed on the host.

The container view was checked using:

```bash
docker exec sonarr sh -c \
'ls -ld /data /data/media /data/media/tv /data/torrents /data/torrents/tv'
```

---

## Root Cause

Sonarr's Docker bind mount depended on the external media filesystem.

When the host filesystem was unavailable, the paths Sonarr expected under `/data` were also unavailable.

---

## Resolution

The external filesystem was restored and the affected containers were restarted.

No Remote Path Mapping was required because qBittorrent and Sonarr already used the same internal `/data` layout.

---

## Lessons Learned

Consistent Docker paths simplify troubleshooting.

Because both applications use:

```text
/data
```

the problem could be identified as a host mount issue rather than a path translation issue.

---

# 6. Avahi Advertised the Wrong Network Interface

## Symptoms

The hostname:

```text
rasp4serv.local
```

resolved unpredictably.

At one point, Avahi returned a Docker bridge address rather than the Raspberry Pi's Ethernet address.

An IPv4 lookup returned:

```text
172.18.x.x
```

instead of the expected LAN address.

---

## Investigation

Network interfaces were inspected with:

```bash
ip addr
```

The system had:

```text
eth0
wlan0
Docker bridge interfaces
```

Avahi was able to see more than one interface and advertised an undesirable address.

---

## Resolution

Avahi was configured to advertise only on Ethernet.

In:

```text
/etc/avahi/avahi-daemon.conf
```

the interface configuration was set to:

```text
allow-interfaces=eth0
```

Avahi was restarted:

```bash
sudo systemctl restart avahi-daemon
```

The hostname then resolved to the Ethernet address.

---

## Lessons Learned

Routing preference and hostname advertisement are separate systems.

Even when Linux prefers Ethernet for outbound traffic, mDNS can still advertise another interface unless explicitly configured.

---

# 7. Seerr Could Not Initially Communicate Correctly With Services

## Symptoms

During initial setup, Seerr reported errors while communicating with Sonarr.

A request for an anime series returned:

```text
timeout of 10000ms exceeded
```

---

## Investigation

Docker DNS was tested:

```bash
docker exec seerr getent hosts sonarr
```

Sonarr resolved correctly.

The Sonarr API was then called directly from inside the Seerr container.

A request to:

```text
http://sonarr:8989/api/v3/system/status
```

returned successfully.

A series lookup was also tested and completed within a few seconds.

---

## Root Cause

The failure was not caused by Docker DNS or basic Sonarr API connectivity.

The original request appeared to involve a transient timeout during series lookup or automatic search.

---

## Resolution

Seerr and Sonarr configuration were verified.

Anime-specific Seerr settings were configured with:

```text
Series Type: Anime
Anime Root Folder: /data/media/tv
Anime Quality Profile: configured Sonarr anime profile
```

Subsequent troubleshooting moved to Sonarr release matching and indexer results.

---

## Lessons Learned

Testing the exact API path from one container to another is more useful than assuming a timeout means general network failure.

Docker connectivity can be verified independently of application-level behavior.

---

# 8. Anime Season Packs Were Rejected During Episode Searches

## Symptoms

Sonarr interactive searches returned releases with warnings such as:

```text
Full season pack
```

and:

```text
Unknown is not wanted in profile
```

---

## Investigation

The searches were being performed for individual episodes.

Many anime releases are distributed as complete season packs.

Sonarr correctly rejected those packs when the search context expected a single episode.

---

## Resolution

Season-level searches were used instead of individual episode searches.

This allowed Sonarr to consider complete season packs.

Anime quality profiles and custom formats were also adjusted to better handle anime releases.

---

## Lessons Learned

Search context matters.

An individual episode search and a season search can evaluate the same release differently.

This is particularly important for anime, where season packs are common.

---

# 9. Incorrect Audio Language Releases

## Symptoms

One anime series was downloaded with only Italian dubbed audio.

The preferred playback configuration is:

```text
Japanese audio
English subtitles
```

---

## Root Cause

The release met the existing quality requirements but there were no strong rules preventing foreign dub-only releases.

---

## Resolution

A dedicated anime Sonarr profile was configured using Custom Formats.

Important negative-scoring formats include:

```text
Language: Not Original
Dubs Only
Anime Raws
VOSTFR
```

These receive strong negative scores so releases missing the original-language audio fall below the minimum accepted Custom Format score.

Dual-audio releases remain acceptable but are not necessarily preferred.

Bazarr separately handles missing English subtitles.

---

## Lessons Learned

Video quality alone is not sufficient for anime release selection.

Audio language, subtitle availability, release groups, and release type may also need to be included in automated scoring.

---

# 10. Bazarr Subtitle Matching and Synchronization

## Symptoms

Bazarr sometimes failed to automatically download subtitles even though manual search showed results.

Other downloaded subtitles were occasionally out of sync with the video.

---

## Investigation

Manual search showed candidate subtitles with scores below the configured automatic download threshold.

Some subtitle files were designed for different encodes of the same episode.

---

## Resolution

Bazarr language profiles were configured for English subtitles.

Embedded subtitles were enabled so Bazarr does not download redundant English subtitles when an English track already exists inside the media file.

Automatic download score thresholds were adjusted where appropriate.

For synchronization problems, Bazarr's audio synchronization functionality can be used against the original-language audio track.

---

## Lessons Learned

A subtitle can match the correct show and episode while still being authored for a different video encode.

Subtitle matching scores help estimate compatibility, but synchronization may still be necessary.

---

# 11. Gluetun Reported an Unhealthy State

## Symptoms

Portainer reported:

```text
gluetun: unhealthy
```

Health output included DNS-related timeouts such as:

```text
lookup cloudflare.com: i/o timeout
lookup github.com: i/o timeout
```

---

## Investigation

DNS was tested from inside Gluetun:

```bash
docker exec gluetun nslookup cloudflare.com
```

HTTPS connectivity was also tested:

```bash
docker exec gluetun wget -T 10 -qO- https://ipinfo.io/ip
```

DNS and HTTPS later worked normally, showing that the failure was intermittent rather than a permanent configuration error.

---

## Possible Causes

Potential causes included:

```text
temporary VPN server issues
DNS timeouts
Internet instability
network saturation
WireGuard tunnel instability
```

---

## Resolution Strategy

The health check was kept enabled rather than disabled.

Gluetun and qBittorrent can be recreated together because qBittorrent shares Gluetun's network namespace.

DNS resolver configuration can also be adjusted if recurring DNS instability is confirmed.

qBittorrent bandwidth limits can reduce the chance of saturating the server or Internet connection.

---

## Lessons Learned

An unhealthy Docker state does not necessarily mean the container is permanently broken.

Testing actual DNS and HTTPS connectivity from inside the container helps distinguish:

```text
health-check state problem
```

from:

```text
real VPN connectivity failure
```

---

# Useful Diagnostic Commands

## Storage

```bash
lsblk -f
```

```bash
findmnt /mnt/media
```

```bash
df -hT
```

```bash
sudo du -xhd1 / 2>/dev/null | sort -h
```

---

## Docker

```bash
docker ps
```

```bash
docker stats --no-stream
```

```bash
docker system df
```

```bash
docker logs <container> --tail 100
```

---

## Networking

```bash
ip addr
```

```bash
ip route
```

```bash
ip route get 8.8.8.8
```

---

## Kernel / Hardware

```bash
dmesg -T | tail -100
```

```bash
dmesg -T | grep -Ei \
'error|i/o|usb|uas|xhci|reset|ext4|mmc|oom|under-voltage'
```

---

## Previous Boot

After a crash or forced restart:

```bash
sudo journalctl -b -1 -k
```

This is particularly useful because the current boot's `dmesg` may not contain the failure that caused the previous system to become unresponsive.

---

# Troubleshooting Principles

Several general principles emerged from operating the homelab:

1. **Check the host before the container**

   If a container cannot find `/data`, verify the external filesystem before changing Docker configuration.

2. **Check the lowest failing layer**

   Application errors can originate from USB hardware, filesystem mounts, DNS, or Docker networking.

3. **Avoid destructive fixes until the root cause is understood**

   Missing directories should not immediately be recreated if they may indicate an unmounted filesystem.

4. **Keep large generated data off the microSD card**

   Caches and extracted media data can grow much larger than expected.

5. **Use consistent container paths**

   Shared `/data` paths make Sonarr, Radarr, qBittorrent, and Bazarr easier to integrate and troubleshoot.

6. **Design for failure**

   `create_host_path: false`, VPN isolation, read-only Jellyfin media mounts, and private remote access all help failures remain contained.

---

## Related Documentation

- [Architecture](architecture.md)
- [Storage](storage.md)
- [Networking](networking.md)
