## Usenet Automation Stack

Fully automated Usenet media pipeline fronting the existing Plex library. Family request via Seerr → Sonarr/Radarr search NZBGeek through Prowlarr → SABnzbd downloads from Newshosting → finished file imported into the Plex library by an instant **atomic move**. Five new unprivileged LXCs (`shanlieve`, `doan`, `errigal` on pve; `conavalla`, `beara` on pve2). Transmission (`iveagh`) retained for manual torrents only.

| Version | File | Audience / store |
|---|---|---|
| Internal (full detail) | `usenet-automation-INTERNAL.md` | Encrypted/password-managed vault — real IPs/hostnames |
| Public (sanitised) | `usenet-automation-PUBLIC.md` | GitHub — RFC-1918 ranges, genericised router/VPN |
| Joplin master | `usenet-automation-JOPLIN.md` | Joplin notebook (RC COMMS / Infrastructure / Runbooks), synced via Nextcloud WebDAV |
| Word (TOC) | `usenet-automation-INTERNAL.docx` | Validated Word, navigable TOC — internal vault |
| Custom formats | `radarr-sonarr-custom-formats.md` | Importable Radarr/Sonarr custom-format JSON + per-profile scoring |

**Key facts:** complete downloads land in `tank/Media` (same dataset as the library → atomic moves); incomplete churn in `tank/scratch` (no snapshots/backup); all apps run as `media` (uid 1000 → host 101000); SAB egress via VPN Fusion (Surfshark UK); Seerr is the only family-facing service (Tailscale MagicDNS), arr admin UIs off the tailnet. Stack data is deliberately **outside** the off-site restic set (reconstructible). See the runbook §11 for the 13 reusable build gotchas (Tailscale-resolver inheritance, `pct set -nameserver` reboot requirement, uv-Python-under-`/root`, TUN device for in-container Tailscale, Jellyseerr→Seerr rename, false success banners, …).

**Related:** `proxmox-infrastructure-INTERNAL.md` (cluster context) · `backup-restore-runbook.md` (snapshot/backup scope) · `nextcloud-vm-INTERNAL.md` (sibling family-cloud service).
