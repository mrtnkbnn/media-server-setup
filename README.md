# Media server setup

Docker Compose setup for a mini PC media server running Jellyfin, Seerr, Radarr,
Sonarr, Bazarr, Prowlarr and Deluge.

The current setup targets Ubuntu Server with an external media disk mounted at
`/mnt/media`. Deluge, Prowlarr, Radarr, Sonarr and Bazarr share a Mullvad VPN
connection through Gluetun. Jellyfin and nginx stay outside the VPN so LAN media
streaming and reverse proxy access remain simple.

## Services

| Service | Default host | Notes |
| --- | --- | --- |
| Jellyfin | `jellyfin.lan` | Direct network, Intel `/dev/dri` passthrough for hardware acceleration |
| Seerr | `seerr.lan` | Direct network, discovery and requests for Jellyfin/Radarr/Sonarr |
| Radarr | `radarr.lan` | Routed through Mullvad VPN |
| Sonarr | `sonarr.lan` | Routed through Mullvad VPN |
| Bazarr | `bazarr.lan` | Routed through Mullvad VPN |
| Prowlarr | `prowlarr.lan` | Routed through Mullvad VPN |
| Deluge | `deluge.lan` | Routed through Mullvad VPN |

## Directory layout

Create the shared data directory before starting the stack:

```bash
sudo mkdir -p /mnt/media/downloads/torrents/movies
sudo mkdir -p /mnt/media/downloads/torrents/tv
sudo mkdir -p /mnt/media/library/movies
sudo mkdir -p /mnt/media/library/tv
sudo chown -R "$USER:$USER" /mnt/media/downloads /mnt/media/library
```

All media containers mount the same host directory as `/data`. Keep this layout
inside the apps:

```text
/data/downloads/torrents/movies
/data/downloads/torrents/tv
/data/library/movies
/data/library/tv
```

Using one shared `/data` mount lets Radarr and Sonarr hardlink imported files
instead of copying them. Deluge can continue seeding from
`/data/downloads/torrents/...`, while Jellyfin sees the same file content under
`/data/library/...` without doubling disk usage.

## Setup

1. Install Docker and the Compose plugin on the Ubuntu server.

2. Copy the environment example:

   ```bash
   cp .env.example .env
   ```

3. Set `PUID` and `PGID` to your server user:

   ```bash
   id
   ```

4. Fill in the Mullvad WireGuard values in `.env`.

   Required values:

   ```env
   MULLVAD_WIREGUARD_PRIVATE_KEY=...
   MULLVAD_WIREGUARD_ADDRESSES=10.x.y.z/32
   MULLVAD_SERVER_COUNTRIES=Estonia
   ```

   Get these from a Mullvad WireGuard configuration. Do not commit `.env`.

5. Make the `.lan` names resolve on your LAN.

   For a quick single-machine test, add entries to your client machine's hosts
   file pointing at the mini PC IP:

   ```text
   192.168.1.10 jellyfin.lan seerr.lan radarr.lan sonarr.lan bazarr.lan prowlarr.lan deluge.lan
   ```

   For normal LAN use, add equivalent local DNS records in your router or DNS
   server.

6. Start the stack:

   ```bash
   docker compose up -d
   ```

7. Check the VPN container logs:

   ```bash
   docker logs gluetun
   ```

   Do not continue with torrenting until Gluetun reports a healthy VPN
   connection.

## First-run application setup

These app settings are intentionally not committed as prebuilt config files.
Radarr, Sonarr, Bazarr and Prowlarr store important state in app databases and
API keys, and those schemas can change between releases. Configure them once in
the web UIs, or later add an explicit API bootstrap script if repeatability
becomes worth the maintenance.

Networking detail: Radarr, Sonarr, Bazarr, Prowlarr and Deluge use
`network_mode: service:gluetun`, so they share Gluetun's network namespace.
Between those apps, use host `localhost` with the target app port. Do not use
Docker service names like `radarr` or `sonarr` for VPN-routed app-to-app
connections. Seerr is not behind the VPN, so it should connect to Jellyfin at
`http://jellyfin:8096` and to Radarr/Sonarr through Gluetun at
`http://gluetun:7878` and `http://gluetun:8989`.

### Deluge

Open `http://deluge.lan`.

Recommended settings:

- Enable the Label plugin.
- Set the `radarr` label download location to `/data/downloads/torrents/movies`.
- Set the `sonarr` label download location to `/data/downloads/torrents/tv`.
- Use categories/labels from Radarr and Sonarr if you enable the Label plugin.
- Keep seeding enabled according to your tracker rules.

Mullvad no longer provides port forwarding, so seeding can still work but may be
less connectable than with a VPN provider that supports forwarded torrent ports.

### Prowlarr

Open `http://prowlarr.lan`.

Recommended settings:

- Add indexers first.
- Add Deluge as a download client using host `localhost`, port `8112`, or manage
  download clients only inside Radarr/Sonarr.
- After Radarr and Sonarr have completed their initial setup, add them as
  Prowlarr apps using host `localhost`, ports `7878` and `8989`.
- Let Prowlarr sync indexers to Radarr and Sonarr instead of configuring the same
  indexers manually in each app.

### Radarr

Open `http://radarr.lan`.

Recommended settings:

- Root folder: `/data/library/movies`.
- Download client: Deluge at host `localhost`, port `8112`.
- Category: `radarr`.
- Enable completed download handling.
- Enable hardlinks instead of copy.
- Enable importing extra files and include subtitle extensions such as `srt`,
  `ass`, `ssa`, `sub` and `idx`.
- After initial setup, copy the Radarr API key into Prowlarr and let Prowlarr
  sync indexers.
- Start with a sane 1080p profile; add custom formats for `HEVC`, `x265`,
  `10bit`, `HDR` or language preferences once you know what your clients handle.

### Sonarr

Open `http://sonarr.lan`.

Recommended settings:

- Root folder: `/data/library/tv`.
- Download client: Deluge at host `localhost`, port `8112`.
- Category: `sonarr`.
- Enable completed download handling.
- Enable hardlinks instead of copy.
- Enable importing extra files and include subtitle extensions such as `srt`,
  `ass`, `ssa`, `sub` and `idx`.
- After initial setup, copy the Sonarr API key into Prowlarr and let Prowlarr
  sync indexers.
- Start with a 1080p profile; add release/custom format rules after the basic
  pipeline works.

### Bazarr

Open `http://bazarr.lan`.

Recommended settings:

- Connect Radarr using host `localhost`, port `7878`.
- Connect Sonarr using host `localhost`, port `8989`.
- Use `/data/library/movies` and `/data/library/tv` paths.
- Configure subtitle providers and language preferences manually.

### Seerr

Open `http://seerr.lan`.

Recommended settings:

- Connect Jellyfin using internal URL `http://jellyfin:8096`.
- Connect Radarr using internal URL `http://gluetun:7878`.
- Connect Sonarr using internal URL `http://gluetun:8989`.
- Use Seerr as the discovery/request UI, but keep quality profiles, custom
  formats and import behavior managed in Radarr and Sonarr.
- If only you use the server, approvals can stay disabled; if other Jellyfin
  users can log in, enable request approvals or restrict permissions.

### Jellyfin

Open `http://jellyfin.lan`.

Recommended settings:

- Movie library path: `/data/library/movies`.
- TV library path: `/data/library/tv`.
- Enable hardware acceleration with Intel VAAPI or Quick Sync.
- If Jellyfin does not see hardware acceleration, verify that `/dev/dri` exists
  on the host and inside the container.

The Intel N150 should handle HEVC much better than a Raspberry Pi 4 when
hardware acceleration is active. Software transcoding can still struggle with
high-bitrate files or subtitle burn-in, so prefer clients that can direct-play
your media and subtitles.

## Useful commands

```bash
docker compose ps
docker compose logs -f gluetun
docker compose logs -f jellyfin
docker compose pull
docker compose up -d
```

## Notes

- nginx uses `nginx.conf.template`; the official nginx image substitutes domain
  values from `.env` at container startup.
- Only HTTP port `80` is exposed by default for LAN use. Add TLS later if you
  expose anything outside your LAN.
- Keep the VPN-routed services behind Gluetun unless you have a specific reason
  to move Radarr, Sonarr or Bazarr back to direct networking.
