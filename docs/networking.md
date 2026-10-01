# Networking

This document describes the networking architecture of the Raspberry Pi homelab, including local Ethernet connectivity, mDNS, Docker networking, Tailscale remote access, and VPN isolation for qBittorrent.

For the overall system design, see [Architecture](architecture.md).

For network-related failures and recovery notes, see [Troubleshooting](troubleshooting.md).

---

## Network Overview

The Raspberry Pi uses Ethernet as its primary network connection.

The network architecture separates traffic into three main categories:

```text
Normal server traffic
    -> Home network / normal Internet connection

Remote administration
    -> Tailscale

Torrent traffic
    -> Gluetun
    -> WireGuard
    -> ProtonVPN
```

This separation allows torrent traffic to use a VPN without forcing any other services through the same tunnel.

---

## Physical Network

The Raspberry Pi 4 is connected to the local network through Gigabit Ethernet.

The primary interface is:

```text
eth0
```

Wi-Fi may also be available through:

```text
wlan0
```

but Ethernet is preferred for server traffic.

Linux route metrics are used to determine which interface is preferred.

A typical configuration may appear as:

```text
eth0     lower metric     preferred
wlan0    higher metric    fallback
```

The active route can be checked with:

```bash
ip route
```

or:

```bash
ip route get 8.8.8.8
```

---

## Local Hostname

The server's local hostname is:

```text
rasp4serv.local
```

This is provided using Avahi and multicast DNS (mDNS).

Avahi advertises the Raspberry Pi on the local network so clients do not need to know the server's DHCP-assigned IPv4 address.

The local hostname can be used for services such as:

```text
http://rasp4serv.local:8096
http://rasp4serv.local:8989
http://rasp4serv.local:7878
```

The Avahi configuration is restricted to the Ethernet interface so Docker bridge addresses are not accidentally advertised.

Conceptually:

```text
rasp4serv.local
       |
       v
     mDNS
       |
       v
     eth0
       |
       v
 Raspberry Pi
```

---

## Tailscale Remote Access

Tailscale provides private remote access to the Raspberry Pi.

The server is available through Tailscale MagicDNS as:

```text
rasp4serv
```

This hostname is different from:

```text
rasp4serv.local
```

The two names use different resolution systems:

```text
rasp4serv.local
    -> Avahi / mDNS
    -> local network

rasp4serv
    -> Tailscale MagicDNS
    -> private Tailscale network
```

Tailscale allows remote access without exposing administrative services directly through router port forwarding.

Examples:

```text
http://rasp4serv:8096    Jellyfin
http://rasp4serv:8989    Sonarr
http://rasp4serv:7878    Radarr
http://rasp4serv:6767    Bazarr
http://rasp4serv:5055    Seerr
http://rasp4serv:9000    Portainer
```

SSH can also be accessed remotely:

```bash
ssh <user>@rasp4serv
```

---

## Docker Network

Most media-stack containers share the default Docker Compose bridge network.

Docker provides internal DNS, allowing containers to communicate using service names instead of IP addresses.

Examples:

```text
sonarr
radarr
jellyfin
seerr
bazarr
prowlarr
gluetun
```

This allows Seerr to reach Sonarr using:

```text
http://sonarr:8989
```

rather than:

```text
http://192.168.x.x:8989
```

Similarly:

```text
Seerr  -> http://radarr:7878
Bazarr -> http://sonarr:8989
Bazarr -> http://radarr:7878
```

Using Docker DNS makes container-to-container communication independent of the Raspberry Pi's DHCP address.

---

## qBittorrent Network Isolation

qBittorrent is intentionally different from the other containers.

Instead of using the default Docker network directly, it shares Gluetun's network namespace:

```yaml
network_mode: "service:gluetun"
```

The resulting path is:

```text
qBittorrent
     |
     | shared network namespace
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

qBittorrent therefore does not have its own independent network interface.

---

## Gluetun

Gluetun acts as the VPN gateway for qBittorrent.

Its responsibilities include:

- establishing the WireGuard tunnel
- routing qBittorrent traffic through ProtonVPN
- DNS resolution inside the VPN environment
- firewall / kill-switch behavior
- ProtonVPN port forwarding
- VPN health monitoring

A simplified configuration includes:

```yaml
environment:
  - VPN_SERVICE_PROVIDER=protonvpn
  - VPN_TYPE=wireguard
  - WIREGUARD_PRIVATE_KEY=${PROTON_WG_PRIVATE_KEY}
  - SERVER_COUNTRIES=United States
  - VPN_PORT_FORWARDING=on
```

Secrets are stored outside the public Compose file using environment variables.

---

## VPN Kill-Switch Behavior

Because qBittorrent shares Gluetun's network namespace, its network traffic depends on Gluetun.

If the VPN tunnel fails, Gluetun's firewall prevents qBittorrent from simply falling back to the Raspberry Pi's normal Internet connection.

This is an important design goal:

```text
VPN healthy
    -> qBittorrent can reach Internet

VPN unavailable
    -> qBittorrent connectivity stops

Normal Raspberry Pi Internet
    -> not used as qBittorrent fallback
```

Other applications such as Jellyfin and Sonarr continue using the normal network connection.

---

## qBittorrent Web Interface

Because qBittorrent shares Gluetun's network namespace, its Web UI port is published through Gluetun.

Example:

```yaml
gluetun:
  ports:
    - "8080:8080"
```

The qBittorrent Web UI is therefore available at:

```text
http://rasp4serv.local:8080
```

or through Tailscale:

```text
http://rasp4serv:8080
```

---

## ProtonVPN Port Forwarding

ProtonVPN dynamically assigns a forwarded port.

Gluetun stores the assigned value and uses an update command to configure qBittorrent's listening port through its Web API.

Conceptually:

```text
ProtonVPN
    |
    | assigns forwarded port
    v
 Gluetun
    |
    | qBittorrent API
    v
qBittorrent listen_port
```

The forwarded port can be checked with:

```bash
docker exec gluetun cat /tmp/gluetun/forwarded_port
```

qBittorrent's current listening port can be checked through its API.

The two values should match.

---

## DNS Inside Gluetun

Gluetun provides DNS resolution for applications sharing its network namespace.

DNS problems can cause Gluetun's health checks to fail even when the container itself remains running.

Useful tests include:

```bash
docker exec gluetun nslookup cloudflare.com
```

and:

```bash
docker exec gluetun wget -T 10 -qO- https://ipinfo.io/ip
```

The second command also verifies that outbound HTTPS traffic is using the VPN connection.

---

## Gluetun Health Checks

Gluetun continuously checks whether the VPN connection is usable.

Portainer may report the container as:

```text
healthy
```

or:

```text
unhealthy
```

An unhealthy state can be caused by:

- VPN connectivity loss
- DNS timeouts
- temporary provider issues
- Internet connectivity problems
- excessive network saturation

Health checks are intentionally kept enabled because they help detect and recover from VPN failures.

---

## Exposed Service Ports

The exact list may change over time, but the media stack uses ports similar to:

| Service | Port |
|---|---:|
| qBittorrent | 8080 |
| Jellyfin | 8096 |
| Sonarr | 8989 |
| Radarr | 7878 |
| Bazarr | 6767 |
| Seerr | 5055 |
| Portainer | 9000 |

These ports are accessible on the local network and, where allowed, through Tailscale.

They are not intentionally exposed directly to the public Internet.

---

## Local vs Remote Access

Local access:

```text
http://rasp4serv.local:<port>
```

Remote Tailscale access:

```text
http://rasp4serv:<port>
```

Examples:

```text
Local:
http://rasp4serv.local:8096

Remote:
http://rasp4serv:8096
```

The service behind both addresses is the same Jellyfin container.

Only the network path and name-resolution mechanism differ.

---

## Security Model

Administrative services are kept private.

The system avoids directly exposing services such as:

```text
Sonarr
Radarr
Prowlarr
qBittorrent
Portainer
Bazarr
```

to the public Internet.

Instead, Tailscale provides authenticated private connectivity.

This reduces the externally exposed attack surface while still allowing remote administration.

---

## Network Design Summary

The network design follows several principles:

1. **Ethernet is the primary server connection**
2. **mDNS provides convenient local naming**
3. **Tailscale provides private remote access**
4. **Docker DNS handles container-to-container communication**
5. **Only qBittorrent is routed through the VPN**
6. **Gluetun provides VPN isolation and kill-switch behavior**
7. **Administrative services are not directly exposed to the public Internet**

---

## Related Documentation

- [Architecture](architecture.md)
- [Storage](storage.md)
- [Troubleshooting](troubleshooting.md)
