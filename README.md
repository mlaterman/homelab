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

## media server (old)

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

## Proxmox

Proxmox is used as the base platform to run services (excluding pihole).

A ZFS filesystem needs to be created and mounted to the containers. And some other packages need to be installed:

```sh
# create media server filesystem
zfs create tank/jellydata
# create directories
mkdir -p /tank/jellydata/media/tv
mkdir -p /tank/jellydata/media/movies
mkdir -p /tank/jellydata/download/torrent
# set jellydata owner
# you may have to set directory owners on proxmox after containers are running to get everything working as expected
# i.e.  download/torrent should be owned by qbittorrent
#       media/* must be owned by media group
chmod -R 100000:101000 /tank/jellydata

# create vtt data storage dir
zfs create tank/vtt
chown -R 100000:100000 /tank/vtt/

zfs create tank/city
chown -R 100000:100000 /tank/city/

# scrub zpool weekly
systemctl enable zfs-scrub-weekly@tank.timer --now

# Install wireguard on host to allow containers to install and use wireguard
apt install wireguard

# Install AMD GPU drivers on host to allow containers to install and use
apt install mesa-va-drivers vainfo
```

Note a domain registed through cloudflare is needed for the nginx container.

The proxmox playbook will run on LXC containers hosted on a proxmox host. The current list is:

- jellyfin - debian 11
  - Runs jellyfin with (AMD) GPU passthrough
- qbittorrent - ubuntu 22.04
  - Runs qbittorrent-nox with wireguard
- arr - ubuntu 20.04
  - sonarr - installed through official repo
  - radarr - manually installed
  - bazarr - installed through python3 virtual env
  - jackett - manually installed
- nginx - debian 11
  - nginx - installed as a package
  - certbot - installed through pip (python3)
  - DDNS script - cloudflare dynamic DNS script (bash)
- vtt/city - debian 11
  - FoundryVTT running on node 14 LTS

The following setup steps are to be done on the host OS in addition to running the `lxc-playbook.yml` targetting the containers:

### jellyfin setup

To mount a zfs filesystem as a bind-mount to an LXC container (ID 100):

```sh
pct set 100 -mp0 /tank/jellydata/media,mp=/jellydata/media
```

Then add the gpu device to the config. i.e.:
```sh
vi /etc/pve/lxc/100.conf
lxc.cgroup2.devices.allow: c 226:* rwm
lxc.mount.entry: /dev/dri/card0 dev/dri/card0 none bind,optional,create=file
lxc.mount.entry: /dev/dri/renderD128 dev/dri/renderD128 none bind,optional,create=file
```

In order to properly use a video card to transcode, find the render group ID in both the host and the container.
In the host:
```sh
root@proxmox:~# getent group | grep render
render:x:103:
```

Container:
```sh
root@jellyfin:~# getent group | grep render
render:x:105
```

Then on the host add the following to `/etc/subgid` to allow LXC containers to map the host's render group:
```
root:103:1
```

And add the following to the configuration for the container:
```sh
vi /etc/pve/lxc/100.conf
lxc.idmap: u 0 100000 65536
lxc.idmap: g 0 100000 105
lxc.idmap: g 105 103 1
lxc.idmap: g 106 100106 65430
```

This will map the host render group to the container render group.
This allows the container to use the video card for transcoding.

### qbittorrent setup

To mount a zfs filesystem as a bind-mount to an LXC container (ID 101):

```sh
pct set 101 -mp0 /tank/jellydata/download/torrent,mp=/jellydata/download/torrent
```

### arr setup

To mount a zfs filesystem as a bind-mount to an LXC container (ID 102):

```sh
pct set 102 -mp0 /tank/jellydata/,mp=/jellydata
```

### nginx setup

Get the zone for the cloudflare domain, and a token with permissions to edit the domain (zone).
forward TCP ports 80/443 to the nginx LXC.

### vtt/city setup

To mount a zfs filesystem as a bind-mount to an LXC container (ID 104, 105):

```sh
pct set 104 -mp0 /tank/vtt/,mp=/data
pct set 105 -mp0 /tank/city/,mp=/data
```
