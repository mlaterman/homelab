# Homelab

homelab will install and configure machines to self host resources.

## Setup

Targets ubuntu/debian (Raspbian) OS'.

- Install ansible (under a python3 venv).
- Add public ssh key to `~/.ssh/authorized_keys` on target instances.
- Assign nodes static IP addresses through a router.

## pihole

Pihole will run under docker compose on an instance.
Systemd is used to ensure it starts on system boot

After deploying pihole set the primary DNS setting (in DHCP options) on the router to pihole's IP.

## media server

The Media server runs jellyfin, qbittorrent, sonarr, and radarr in docker compose.

The drive that is used by the media server should have the following structure:
```
/apps/
  qbittorrent/
    wireguard/
      wg0.conf
  sonarr/
  radarr/
  jellyfin/
/data/
  media/
    movies/
    tvshows/
  torrent/
```

A wireguard dir and config file must be present in the qbittorrent directory to ensure that a vpn is used.
