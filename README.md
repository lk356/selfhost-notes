# Homelab: Self-Hosting Behind CGNAT

Notes on how I self-host [Immich](https://immich.app) (photos) from a headless laptop on university ethernet, made publicly reachable through a cheap VPS relay. Written as a runbook for myself and a record of what I learned getting here.

> All IPs, usernames, hostnames, and domains below are placeholders (`<VPS_IP>`, `<LAPTOP_TS_IP>`, `<user>`, `example.com`).

## Architecture

```
phone / browser
      |
      v
Cloudflare DNS  (photos.example.com -> <VPS_IP>, DNS only)
      |
      v
VPS: Caddy (HTTPS, Let's Encrypt)
      |
      |  Tailscale tunnel
      v
Laptop server: Immich in Docker, port 2283
      ^
      |  nightly rsync pull (04:00)
Raspberry Pi: offsite backup at home
```

No client device needs Tailscale just to view photos. The laptop is the only place the live library lives; the Pi pulls a copy each night.

| Machine | Role | Notes |
| --- | --- | --- |
| Laptop server | Runs Immich (Docker) | Old Dell with a broken screen, Debian, headless, 1 Gbps symmetric campus ethernet via USB adapter |
| External 2 TB SSD | Photo library | ext4, mounted at `/mnt/t7` |
| Cheap VPS | Public front door | 1 GB KVM, Ubuntu 24.04, ~$22/yr, 3 TB/mo; runs Caddy + Tailscale + ufw |
| Raspberry Pi 4B | Offsite backup | DietPi, pulls the library nightly, never deletes |
| Domain | Public name | Bought via Cloudflare, A record in DNS-only mode |

## How it got here

1. **Raspberry Pi 4B** started as a Home Assistant experiment, then became a general server. The SD card corrupted, so I moved to DietPi on a USB hard drive.
2. Over time it hosted Nextcloud, Immich, Navidrome, Tailscale, and an Xray VPN (backup access in case a network blocked WireGuard), all headless.
3. **CGNAT** meant no port forwarding, so Tailscale was the main way in. A Cloudflare Tunnel for Immich/Navidrome worked but was unreliable.
4. In the dorm I set up an old laptop with Debian. It first ran a Minecraft server (tunneled with playit.gg; getting Simple Voice Chat working was painful).
5. When nobody used it, I moved Immich there. I didn't want every device to need Tailscale, so I bought a VPS and a domain to act as a public relay.
6. Migrated the library and database from the Pi to the laptop with rsync and `pg_dumpall`.

## Key design decisions

- **VPS relay instead of Cloudflare Tunnel or proxy.** DNS points straight at the VPS. Cloudflare's free proxy caps uploads at 100 MB, which breaks large video backups, and the tunnel was unreliable.
- **Why a domain at all.** A bare IP means plain HTTP or a self-signed cert. A domain gives real HTTPS via Caddy with automatic renewal.
- **Tailscale only between the VPS and the laptop.** Keeps the laptop off the public internet and works through campus networking.
- **Database on the internal SSD, photos on the external drive.** A USB hiccup is far less damaging to image files than to a database mid-write.
- **Docker waits for the drive.** A systemd override (`RequiresMountsFor=/mnt/t7`) stops Docker from starting without the library mounted.
- **Backups are pulled, not pushed.** The Pi pulls from the laptop, so a compromised laptop can't wipe the backup. No `--delete`, so deleted photos survive on the Pi.
- **Pinned Immich version.** `IMMICH_VERSION` is set to a specific release, never `release`, so updates are deliberate.

## Setup summary

### Laptop: never sleep

```ini
# /etc/systemd/logind.conf
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
```

```bash
sudo systemctl restart systemd-logind
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

### Laptop: drive and Docker

- Format the external SSD as GPT + ext4 (not exFAT/NTFS; Docker and Postgres need Linux permissions).
- Mount via `/etc/fstab` with `nofail` so a missing drive doesn't hang boot:

```
UUID=<uuid>  /mnt/t7  ext4  defaults,nofail,x-systemd.device-timeout=10  0  2
```

- Install Docker from Docker's official repo (not `docker.io`).
- `sudo systemctl edit docker`:

```ini
[Unit]
RequiresMountsFor=/mnt/t7
```

- Immich `.env` values that matter:

```
UPLOAD_LOCATION=/mnt/t7/immich/library
DB_DATA_LOCATION=/opt/immich/postgres
IMMICH_VERSION=v<pinned version>
```

### VPS: relay

```bash
apt update && apt upgrade -y
apt install -y ufw caddy
ufw allow OpenSSH && ufw allow 80,443/tcp && ufw enable
curl -fsSL https://tailscale.com/install.sh | sh && tailscale up
```

`/etc/caddy/Caddyfile`:

```
photos.example.com {
    reverse_proxy <LAPTOP_TS_IP>:2283
}
```

Then `systemctl reload caddy`. Caddy handles certificates itself but needs port 80 reachable.

SSH hardening goes in `/etc/ssh/sshd_config.d/00-hardening.conf` (key-only, no password login). The `00-` prefix matters because sshd uses the first value it reads. Always test a new login in a second terminal before closing the working one.

### DNS

A record `photos` -> `<VPS_IP>`, **DNS only (grey cloud)**. Verify with `dig +short photos.example.com`.

### Adding another service

One subdomain per service: add an A record and another Caddy block pointing at that service's port.

## Migration from the Pi (short version)

1. On the Pi: stop the app, keep Postgres running, dump with `pg_dumpall --clean --if-exists | gzip`.
2. **Verify the dump** with `gzip -t` and check the tail says "PostgreSQL database cluster dump complete".
3. Copy the library and dump with `rsync -aH` **inside tmux**, so a dropped SSH session doesn't kill it.
4. On the laptop: fresh Postgres data dir, restore the dump on the **same Immich version**, then bring the stack up.
5. Set the external domain in Immich settings and clear the phone app's data before logging in again.

The ~234 GB library took about 14 hours over home upload speeds.

## Backups

- Immich writes its own DB backup nightly into `library/backups`.
- The Pi pulls the library at 04:00 via cron (rsync over SSH key, excluding regenerable `thumbs/` and `encoded-video/`).
- Verify with a dry run: `rsync -aHn --stats ...` should report ~0 files to transfer.

```
0 4 * * * rsync -aH --exclude 'thumbs/' --exclude 'encoded-video/' root@<LAPTOP_TS_IP>:/mnt/t7/immich/library/ /root/immich-app/library/ >> /root/immich-backup.log 2>&1
```

## Updating and rollback

1. Read the release notes, especially breaking changes.
2. Take a DB dump (`pg_dumpall | gzip`), verify it, copy it to the Pi.
3. Bump `IMMICH_VERSION`, then `docker compose pull && docker compose up -d`, and watch the logs.
4. **Rollback:** pin the old version and do a clean restore from the pre-update dump.

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Immich crash-loops, Postgres error 42P10 after restore | Truncated DB dump (Ctrl+C during a multi-line paste) | Redo the dump, verify with `gzip -t` and the "dump complete" line, wipe the DB dir, restore again |
| Backup log contains `0: command not found` | Cron line pasted into the shell instead of `crontab -e` | Add it through `crontab -e`; confirm with `crontab -l` |
| rsync dies when the SSH session drops | Not running inside tmux | `tmux new -s copy`, rerun (rsync resumes) |
| `Failed to exec ssh` on the Pi | DietPi ships Dropbear with no `ssh` client | `apt install openssh-client tmux` |
| `rsync: command not found` on the remote end | rsync must exist on both machines | Install it on the receiver |
| Immich down after reboot | Drive didn't mount, so Docker didn't start | `df -h /mnt/t7`, fix the mount, `systemctl start docker` |
| Drive disconnects or is slow | USB 2.0 port or bad cable, or USB power saving | `lsusb -t` (want 5000M+), `dmesg`, swap port or cable |
| Site slow | Tailscale falling back to a DERP relay | `tailscale status` should show "direct" |
| HTTPS broken | Cert renewal failing | `journalctl -u caddy`, check port 80 is open and DNS is still grey-cloud |
| `502` from the VPS | Laptop or Immich is down | Check the laptop's containers and Tailscale connectivity |
| Phone app won't load after the move | Cached old server details | Clear the app's data, log in again |
| `docker compose` says no such file | Logged in as root, so `~` is `/root` | `cd` to the real compose folder first |

## Weekly checks

- [ ] Backup log is empty and the newest DB backup on the Pi is recent
- [ ] Disk space on the Pi, external SSD, and laptop internal drive
- [ ] Read Immich release notes before updating
- [ ] `apt upgrade` on the VPS, laptop, and Pi

## Security notes

- Login page is public, so every account needs a long, unique password. Accounts are admin-created only.
- Share links get expiry dates or passwords.
- Possible hardening not yet done: Tailscale ACL limiting the VPS to the laptop's Immich port; binding Immich's port to the Tailscale IP only so the campus LAN can't reach it.

## Lessons learned

- Paste and run commands **one at a time** and verify output. The two real breakages (truncated dump, cron that never installed) both came from pasting multi-line blocks.
- Anything long-running goes in tmux.
- Verify backups with a dry run rather than assuming they work.
- A flaky free tunnel is worse than a cheap VPS you control.

## Future ideas

- restic for versioned, deduplicated backups
- Move the library to the laptop's empty internal SATA bay
- More services (Navidrome, etc.) on their own subdomains
- 2FA via an OAuth provider (Pocket ID or Authelia)
