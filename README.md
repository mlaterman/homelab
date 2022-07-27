# Homelab

homelab will install and configure machines to self host resources.

## Setup

Targets ubuntu/debian (Raspbian) OS'.

- Install ansible (under a python3 venv).
- Add public ssh key to `~/.ssh/authorized_keys` on target instances.
- Assign nodes static IP addresses through a router.

## ZFS

There is a role to install ZFS on debian operating systems.
Creation of pools should be done manually before the storage pool is needed.

Set ashift=12 to have 4k sectors (9=512 bytes is the default):
```sh
zpool create -o ashift=12 tank raidz1 $DISKS
```

Create the mountpoint and mount  a filesystem
```sh
zfs create -o mountpoint=/data tank/data
zfs create -o mountpoint=/srv/media tank/data/media # media library
zfs create -o mountpoint=/srv/data  tank/data/lxc  # lxc backups?
```

Enable zfs scrub weekly:
```sh
systemctl enable zfs-scrub-weekly@tank.timer --now
```

## LXC

Install LXC in order to host services in OS containers.

```
# Find an image
/usr/share/lxc/templates/lxc-download
/usr/share/lxc/templates/lxc-download | grep debian | grep amd64
```

```
# creation
lxc-create -t /usr/share/lxc/templates/lxc-download -n jellyfin2 -- -d debian -r bullseye -a amd64
# start
lxc-unpriv-start -n jellyfin
# attach
lxc-unpriv-attach -n jellyfin
```

Note: I'm not sure how to get a container an IP address that is provisioned by my network DHCP service.

## Hardware acceleration

Jellyfin is capable of using a GPU unit to transcode video.
Host machine must have the drivers installed in order to pass the device to an LXC instance.

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
