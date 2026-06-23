# Usenet Media Automation Stack — Build & Operations Runbook

> **🌐 PUBLIC / SANITISED VERSION**
> Hostnames are retained for readability; IP addressing has been mapped to documentation ranges (RFC 1918), and identifying router/VPN/account details have been genericised. No secrets appear in this document. Substitute your own values throughout.

| | |
|---|---|
| **Maintainer** | Richard Carragher (RC COMMS) |
| **Last updated** | 22 June 2026 |
| **Document version** | 1.0 |
| **Proxmox VE version** | 9.2.3 (kernel 7.0.6-2-pve) |
| **Cluster** | `ulster` (nodes: pve, pve2) |
| **Stack** | SABnzbd · Sonarr · Radarr · Prowlarr · Seerr · Plex |
| **Classification** | Public |

---

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Storage & the atomic-move design](#3-storage--the-atomic-move-design)
4. [Ownership & UID mapping](#4-ownership--uid-mapping)
5. [Build procedure](#5-build-procedure)
6. [Provider, indexer & Prowlarr wiring](#6-provider-indexer--prowlarr-wiring)
7. [VPN routing (per-device policy)](#7-vpn-routing-per-device-policy)
8. [Family access design](#8-family-access-design)
9. [Quality profiles & custom formats](#9-quality-profiles--custom-formats)
10. [Known quirks and gotchas](#10-known-quirks-and-gotchas)
11. [Operational runbook](#11-operational-runbook)
12. [Troubleshooting](#12-troubleshooting)
13. [Family usage](#13-family-usage)
14. [Explain like I'm 5](#14-explain-like-im-5)
15. [References](#15-references)
16. [Appendices](#16-appendices)

---

## 1. Overview

A fully automated Usenet media pipeline bolted onto the existing Plex library. A family member requests a film or show through a clean web portal (Seerr); the request is handed to Sonarr/Radarr, which search the indexer (NZBGeek) via Prowlarr, send the chosen NZB to the downloader (SABnzbd), which pulls and unpacks it from the Usenet provider (Newshosting); the finished file is moved into the Plex library, where it appears for playback. No human touches a download client after the request.

The automation runs as **five new unprivileged LXCs** on the `ulster` cluster, joining the pre-existing Plex (`slemish`), NAS (`oriel`) and Transmission (`iveagh`) containers. Transmission is retained for **manual torrent downloads only** — Usenet is the automated path.

### Why this design?

1. **One service per LXC, not a combined box.** Matches the existing estate (every other service is one-per-container). Gives independent restart/update cadence, per-container resource caps, and failure isolation — SABnzbd is the only CPU/I-O-hungry member (par2 repair, unpack) and is isolated so it cannot starve the others.
2. **Downloads land *inside* the `tank/Media` dataset.** The single most important decision in the build. ZFS atomic moves (instant `rename()`) only work *within one dataset*; across datasets the import is a slow copy+delete. Putting SAB's *complete* folder under `tank/Media` (same dataset as the library) makes every import instant. See §3.
3. **Incomplete/temp churn lives on a separate throwaway dataset (`tank/scratch`).** Partials, par2 blocks and unpack scratch are worthless and high-churn; they sit on a dataset with no snapshots and no backup so they never pollute `tank/Media` snapshots. See §3.
4. **Family reach a request portal, never the admin UIs.** Seerr is the only service exposed to family (over Tailscale MagicDNS). The arr apps stay admin-only. Family get a request box with per-user quotas, not Radarr's delete button. See §8.
5. **All containers run as a single `media` user (uid/gid 1000 → host 101000)** so the Samba view (oriel), Plex and the arr-managed files share one consistent owner. See §4.

---

## 2. Architecture

### 2.1 Components

| Component | Host | IP:Port | Node | Notes |
|---|---|---|---|---|
| SABnzbd | CT 114 `shanlieve` | 10.0.0.35:7777 | pve | Usenet downloader; par2cmdline-turbo 1.4.0; runs as `media` |
| Sonarr 4.0.17 | CT 115 `doan` | 10.0.0.36:8989 | pve | TV automation; runs as `media` |
| Radarr 6.2.1 | CT 116 `errigal` | 10.0.0.37:7878 | pve | Movie automation; runs as `media` |
| Prowlarr 2.4.0 | CT 117 `conavalla` | 10.0.0.38:9696 | **pve2** | Indexer manager; no bind-mounts (network only) |
| Seerr v3.3.0 | CT 118 `beara` | 10.0.0.39:5055 | **pve2** | Family request portal (unified Jellyseerr/Overseerr); on Tailscale |
| Plex | CT 102 `slemish` (existing) | 10.0.0.33 | pve | Reads `/mnt/media`; transcode |
| NAS / Samba | CT 101 `oriel` (existing) | 10.0.0.32 | pve | Owns `tank/Media`/`tank/scratch` shares |
| Transmission | CT 103 `iveagh` (existing) | 10.0.0.34 | pve | **Manual torrents only** — outside the automation |
| Pi-hole (pri.) | CT 106 `commedagh` | 10.0.0.10 | pve | DNS resolver for the stack |
| Pi-hole (sec.) | CT 104 `donard` | 10.0.0.11 | pve2 | DNS resolver (redundancy) |
| Router / VPN | Consumer router (VPN-policy-routing capable) | 10.0.0.1 | — | Per-device VPN policy routing; firmware redacted |

All five new containers are **unprivileged**, Debian-based, built via the Community Helper Scripts. `tank` is **pve-only** — that is why SAB/Sonarr/Radarr (which bind-mount `tank`) must run on **pve**, while Prowlarr/Seerr (no `tank` mounts) sit on **pve2** to balance load.

### 2.2 Data flow

```
Family member (anywhere)
  └─ http://beara.<tailnet>.ts.net:5055   (Seerr — request portal, Tailscale)
       └─ approved request → Sonarr (10.0.0.36) / Radarr (10.0.0.37)
            └─ indexer search via Prowlarr (10.0.0.38) → NZBGeek (api.nzbgeek.info)
                 └─ NZB sent to SABnzbd (10.0.0.35:7777)
                      └─ download from Newshosting (news.newshosting.com:563, TLS)
                         · incomplete  → /mnt/scratch/usenet/incomplete   (tank/scratch — throwaway)
                         · complete    → /mnt/media/usenet/complete        (tank/Media)
                           └─ Sonarr/Radarr ATOMIC MOVE → /mnt/media/tv | /mnt/media/movies
                                └─ Plex (slemish) sees the new file → playback
```

SAB's egress (only) is policy-routed through the commercial VPN (UK exit) at the router (per-device policy, by device IP). Everything else uses the LAN gateway directly.

### 2.3 Bind-mounts

| CT | Mount | Container path | Purpose |
|---|---|---|---|
| 114 SAB | `/tank/Media` | `/mnt/media` | complete dir + library visibility |
| 114 SAB | `/tank/scratch` | `/mnt/scratch` | incomplete/temp |
| 115 Sonarr | `/tank/Media` | `/mnt/media` | imports *from* complete, *to* `/mnt/media/tv` |
| 116 Radarr | `/tank/Media` | `/mnt/media` | imports *to* `/mnt/media/movies` |
| 117 Prowlarr | — | — | none needed |
| 118 Seerr | — | — | none needed |

All containers see `/mnt/media` at the **identical path**, so no remote-path mapping is required in the arr apps.

---

## 3. Storage & the atomic-move design

This is the architectural heart of the build. Get the dataset boundary wrong and every import becomes a slow, space-doubling copy.

### 3.1 The rule

> A ZFS **atomic move** (instant `rename()`) — and a **hardlink** — only work **within a single dataset**. A move *across* datasets is a physical copy + delete. Each ZFS dataset (including a *child* dataset) is its own filesystem.

Usenet does not seed, so hardlinks are not needed (unlike torrents). But atomic moves still matter: without them, importing a 60 GB season pack copies the whole file and briefly doubles its space, on every import.

### 3.2 Layout

```
tank/scratch                     (NO snapshots, NO backup — throwaway)
  └─ usenet/incomplete           ← SAB temp dir (partials, par2, unpack scratch)

tank/Media                       (atomic-move zone — OUTSIDE restic by sibling naming)
  ├─ usenet/complete             ← SAB complete dir  ← Sonarr/Radarr import FROM here
  ├─ tv        4ktv    kidstv    documentaries        ← Sonarr root folders
  ├─ movies    4kmovies   kids                        ← Radarr root folders
  └─ (existing genre/curation folders: horror, christmas → folded into movies)
```

Critical points:

- SAB's **complete** dir is inside `tank/Media` → import to `tank/Media/tv` is an **atomic move**. Do **not** put the complete dir on `tank/scratch` — that re-introduces the cross-dataset copy and defeats the whole design.
- SAB's **incomplete/temp** dir is on `tank/scratch` → the churny worthless data never enters `tank/Media` snapshots.
- A ZFS **child** dataset (e.g. `tank/Media/downloads`) would *not* help — it is a separate filesystem. The complete/incomplete dirs are plain **directories** inside the datasets, not nested datasets.

### 3.3 Backup & snapshot scope (deliberate control)

- `tank/Media` is **outside restic** by sibling naming (the restic target is `tank/storage`; siblings are automatically excluded — an architectural control, not an oversight). Media is reconstructible, so folding downloads in keeps them correctly out of the off-site set.
- `tank/scratch` is likewise outside restic, **and** explicitly outside sanoid (no snapshot policy / `autosnap=no`). Snapshotting a download scratch area only pins deleted churn.
- Trade-off accepted: `tank/Media/usenet/complete` *is* inside `tank/Media`'s sanoid scope, so transient post-download cruft (`_UNPACK_…`, `.1` dedupe remnants) can be briefly captured. Mitigated by SAB's history/queue cleanup (see §11) and a light `tank/Media` snapshot policy.

### 3.4 Host-side creation (on pve)

```bash
mkdir -p /tank/scratch/usenet/incomplete
mkdir -p /tank/Media/usenet/complete
chown -R 101000:101000 /tank/scratch/usenet /tank/Media/usenet
chmod -R 2775 /tank/scratch/usenet /tank/Media/usenet   # setgid: new files inherit gid 1000
```

The **setgid** bit (`2775`) is what keeps SAB's writes and Sonarr's moves landing as group `1000` automatically, so ownership never drifts.

### 3.5 Future: scratch on NVMe

par2 repair and unpack are the I/O-heavy phase. Moving `tank/scratch` onto NVMe (rather than the spinning `tank` vdevs) markedly speeds repairs/extractions — a natural thing to revisit when the MS-01 cluster-refresh nodes land.

---

## 4. Ownership & UID mapping

### 4.1 Model

All containers are unprivileged with the standard **offset 100000**. Each app runs as an in-container `media` user, **uid/gid 1000**, which maps to **host uid/gid 101000**. The `tank/Media` tree is already owned `101000:101000` on the host (confirmed via `ls -n`).

| Layer | uid:gid |
|---|---|
| In-container `media` user | 1000:1000 |
| On the host (mapped) | 101000:101000 |

### 4.2 Per-container alignment (115/116/114)

The helper scripts install the arr apps to run as **root**. Left as root, files written into `/mnt/media` land as host `100000` and split ownership against the `101000` library. Fix per container:

```bash
pct exec <id> -- bash -c '
groupadd -g 1000 media 2>/dev/null
getent passwd media >/dev/null || useradd -u 1000 -g 1000 -M -s /usr/sbin/nologin media
'
```

Then a systemd drop-in to run as `media` (Sonarr/Radarr example):

```bash
pct exec 115 -- bash -c '
systemctl stop sonarr
chown -R 1000:1000 /var/lib/sonarr /opt/Sonarr
install -d /etc/systemd/system/sonarr.service.d
printf "[Service]\nUser=media\nGroup=media\n" > /etc/systemd/system/sonarr.service.d/override.conf
systemctl daemon-reload && systemctl start sonarr'
```

> **Servarr note:** `chown` the **install dir** (`/opt/Sonarr`, `/opt/Radarr`) as well as the data dir. The built-in updater replaces files *in the install directory*, so the run-as user must own it or in-app updates fail. Standard Servarr practice for non-root installs.

### 4.3 SAB specifics (see also §5.4)

SAB has no `-f` flag by default → as root it stores config in `/root/.sabnzbd`, which a non-root user cannot use. The override pins config to `/opt/sabnzbd/config` and sets `HOME`:

```
[Service]
User=media
Group=media
Environment=HOME=/opt/sabnzbd/config
ExecStart=
ExecStart=/opt/sabnzbd/venv/bin/python SABnzbd.py -f /opt/sabnzbd/config -s 0.0.0.0:7777
```

The blank `ExecStart=` before the new one is **mandatory** — systemd *appends* to ExecStart otherwise, producing two start commands and a failed unit.

### 4.4 Verification

After any grab, confirm ownership held end-to-end (ideal = no output):

```bash
ls -lnR "/tank/Media/tv/<Title>" | grep -v '101000 101000' | grep -E '^[-d]'
```

Anything printed is owned by something other than `101000:101000` and indicates the override didn't take.

---

## 5. Build procedure

### 5.1 Create the five LXCs (Community Helper Scripts)

Verify the current one-liner at `community-scripts.github.io/ProxmoxVE` per app (they occasionally re-path). **SAB/Sonarr/Radarr on pve; Prowlarr/Seerr on pve2.** Advanced install, unprivileged, Debian. Sizing: SAB 4 GB/4 cores; the rest 1–2 GB/1–2 cores. **Let each script run start-to-finish untouched** — a mid-run interruption produces a false "success" banner with nothing installed (Gotcha 7).

```bash
# on pve:
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/sabnzbd.sh)"
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/sonarr.sh)"
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/radarr.sh)"
# on pve2:
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/prowlarr.sh)"
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/seerr.sh)"   # NOT jellyseerr.sh (Gotcha 6)
```

> Installer ports differ from defaults: **SAB = 7777**, **Prowlarr = 9696** (Sonarr 8989 / Radarr 7878 as expected). Seerr = 5055.

### 5.2 DNS — fix on every container (Gotcha 1 & 2)

New CTs inherit the host's Tailscale resolver `100.100.100.100` (MagicDNS), which cannot resolve public DNS — series/movie lookups and indexer tests fail silently. Set the Pi-hole resolver in the container config **and reboot** (a `pct set -nameserver` only regenerates `resolv.conf` on container *start* — an app restart is not enough):

```bash
pct set <id> -nameserver 10.0.0.10
pct reboot <id>
pct exec <id> -- getent hosts thetvdb.com    # confirm an IP comes back
```

### 5.3 Storage & ownership

Run §3.4 (host dirs + setgid) and §4.2 (media user + overrides). Bind-mounts:

```bash
pct set 114 -mp0 /tank/Media,mp=/mnt/media -mp1 /tank/scratch,mp=/mnt/scratch   # SAB
pct set 115 -mp0 /tank/Media,mp=/mnt/media                                       # Sonarr
pct set 116 -mp0 /tank/Media,mp=/mnt/media                                       # Radarr
pct reboot 114 115 116
```

### 5.4 SAB — uv Python relocation (Gotcha 3)

The SAB helper builds its venv against a **uv-managed Python under `/root/.local`** (mode `700`). A non-root service user cannot traverse `/root`, so `tailscaled`-style `203/EXEC Permission denied` occurs on the interpreter. Relocate the interpreter out of `/root` (follow symlinks with `-L`):

```bash
pct exec 114 -- bash -c '
systemctl stop sabnzbd
rm -rf /opt/uv-python; mkdir -p /opt/uv-python
cp -rL /root/.local/share/uv/python/* /opt/uv-python/
chown -R 1000:1000 /opt/uv-python
ln -sfn /opt/uv-python/cpython-3.13.14-linux-x86_64-gnu/bin/python3.13 /opt/sabnzbd/venv/bin/python
systemctl reset-failed sabnzbd && systemctl restart sabnzbd
sleep 3; systemctl is-active sabnzbd; ss -ltnp 2>/dev/null | grep 7777'
```

> `cp -rL` (follow links → copy real files), **not** `cp -a` (preserves the symlink, which still points back into `/root`). Point the venv `python` symlink at the **versioned** real dir.

### 5.5 SAB folders & categories (UI, `:7777`)

- **Temporary Download Folder:** `/mnt/scratch/usenet/incomplete`
- **Completed Download Folder:** `/mnt/media/usenet/complete`
- **Categories:** `tv` and `movies` — leave **Folder/Path BLANK** (Gotcha 11; a path here creates an orphaned second copy).
- Copy SAB's **API Key** (Config → General) for Prowlarr/Sonarr/Radarr.
- Connections: 30. **Server: see §6** (left empty until the provider is configured).
- Enable history/queue cleanup (Config → Switches) so `complete/` is tidied after import.

### 5.6 Verify

```bash
pct exec 114 -- bash -c 'systemctl is-active sabnzbd; ss -ltnp | grep 7777'
pct exec 115 -- bash -c 'systemctl is-active sonarr;  ss -ltnp | grep 8989'
pct exec 116 -- bash -c 'systemctl is-active radarr;  ss -ltnp | grep 7878'
ssh pve2 "pct exec 117 -- bash -c 'systemctl is-active prowlarr; ss -ltnp | grep 9696'"
ssh pve2 "pct exec 118 -- bash -c 'systemctl is-active seerr;    ss -ltnp | grep 5055'"
```

> `pct exec` is **node-local** (Gotcha 13): containers 117/118 live on pve2, so query them from pve2 (or `ssh pve2 …`). Running `pct config 118` on pve returns *"Configuration file does not exist"*.

---

## 6. Provider, indexer & Prowlarr wiring

### 6.1 The provider vs indexer distinction

- **Provider (news server):** Newshosting — `news.newshosting.com`, port **563** (SSL/TLS). This is where the binaries are actually pulled from. Account: a paid Usenet provider subscription (commercial term).
- **Indexer (search catalogue):** NZBGeek — `api.nzbgeek.info`. Finds the NZBs; does **not** serve content. The **API requires an active VIP membership** on NZBGeek's side — an indexer Test that fails with an auth error despite a correct key is usually a lapsed VIP, not a config fault.

> A common trap: NZBGeek alone downloads nothing — without a provider, every grab fails at "connecting to server."

### 6.2 SAB server entry

In SAB → Config → Servers, **Host and Port are separate fields**. Entering `news.newshosting.com:563` in the Host box yields *"Server address … is not valid"* (colon rejected). Set:

- **Host:** `news.newshosting.com`
- **Port:** `563`
- **SSL:** ticked
- **Username / Password:** Newshosting login
- **Connections:** 30

### 6.3 Prowlarr (the hub) — wiring order: SAB → Apps → Indexer

1. **Authentication:** Forms (Login), username + password. Do not run authless even on the LAN.
2. **Download Clients → + → SABnzbd:** Host `10.0.0.35`, Port `7777`, SAB API key. Test → green.
3. **Apps → +:** add Sonarr (`http://10.0.0.36:8989`, Sonarr's own API key, **Full Sync**) and Radarr (`http://10.0.0.37:7878`, Radarr's key, Full Sync). The Prowlarr Server URL must be the `.38` address (not localhost — the arr apps connect *back* to it).
4. **Indexers → Add Indexer → NZBGeek:** select from the built-in list (URL is known), paste the NZBGeek API key, Test → Save. On Save, Full Sync pushes the indexer **and** the SAB client out to Sonarr and Radarr automatically.

> If Sonarr shows *"No download client is available,"* the client sync hasn't propagated — either re-run **Apps → Sync App Indexers**, or add SABnzbd directly in Sonarr/Radarr (Settings → Download Clients → SABnzbd, Host `.35`, Port `7777`, category `tv`/`movies`). Adding it directly unblocks immediately; Prowlarr's client sync is convenience, not a requirement.

### 6.4 Sonarr/Radarr root folders

- Sonarr → Media Management → Root Folders: `/mnt/media/tv`, `/mnt/media/4ktv`, `/mnt/media/kidstv`, `/mnt/media/documentaries`
- Radarr → Root Folders: `/mnt/media/movies`, `/mnt/media/4kmovies`, `/mnt/media/kids`

Genre folders (`horror`, `christmas`) are **not** arr routing — they are folded into `/mnt/media/movies` and surfaced as **Plex Collections** (see §13).

---

## 7. VPN routing (per-device policy)

SAB's egress is routed through a commercial VPN **at the router**, not via a VPN client inside the container.

- **Router:** consumer router with per-device VPN policy routing (vendor feature). **Commercial VPN provider, UK exit node**.
- **Binding:** policy-routed by **device IP** — `10.0.0.35` (SAB) is bound to the tunnel; the rest of the LAN uses the normal WAN.
- The container's own routing table shows plain `eth0 → 10.0.0.1`; the router handles the tunnel transparently downstream of that hop.

### 7.1 Is the VPN even necessary?

For Usenet, usually **no** — the connection to Newshosting is already TLS on 563, so the ISP sees an encrypted session to a news host, not content. The VPN here is a deliberate extra layer; it adds latency and a failure point. It is **not** required for the pipeline to function.

### 7.2 Fragility — important

If the per-device VPN binding ever drops (router firmware update, DHCP drift), **SAB silently falls back to the bare WAN with no kill-switch** — the container has no idea it is no longer tunnelled. Mitigations:

1. **`.35` must be a DHCP reservation / static** so the binding cannot migrate to another host.
2. **After any router firmware update, eyeball the router VPN policy page** to confirm the SAB binding survived.
3. Only `.35` should be bound — verify no unintended devices (e.g. `.41` eurooffice was briefly mis-bound and corrected) share the UK VPN exit route.

---

## 8. Family access design

The goal: let a family member open a link and request a movie/show themselves, **without** any indication they have access to the wider network, and **without** handing them an admin UI.

### 8.1 Final design — Seerr request portal

- **Seerr (beara)** is the only service family touch. It is a request portal (unified fork of Jellyseerr/Overseerr) connected to Plex (imports libraries + users) and to Sonarr/Radarr (sends approved requests straight in).
- Family **never see Radarr/Sonarr**. They browse a clean UI and click "Request." Per-user **request quotas** cap volume; children are on **manual approve** (richard/clare auto-approve).
- Exposed via **Tailscale MagicDNS**: `http://beara.<tailnet>.ts.net:5055`.

### 8.2 Why per-service Tailscale, not a subnet router

The earlier option was per-container Tailscale on doan/errigal so a family member could be sent a direct link to Sonarr/Radarr. The **deciding rationale**: a subnet router advertising `10.0.0.0/24` would expose the *entire* subnet to family. Per-service Tailscale (MagicDNS to one container) puts only that service on the tailnet — **no subnet visibility**. Seerr then improves on this further: family reach only the request portal, and the admin UIs leave the tailnet entirely (Tailscale removed from doan/errigal; their nodes deleted from the admin console).

### 8.3 TUN device for in-container Tailscale (Gotcha 5)

An unprivileged LXC cannot create `/dev/net/tun` itself; `tailscaled` crash-loops (`status=1/FAILURE`, then "start request repeated too quickly"). Add the device to the container config on the host:

```bash
cat >> /etc/pve/lxc/<id>.conf <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
pct reboot <id>
pct exec <id> -- bash -c 'tailscale up'
```

### 8.4 Seerr 4K servers

To offer family a "Request in 4K" option, add a **second** Radarr/Sonarr server entry in Seerr pointing at the same box but defaulting to the `2160p Quality` profile + `/mnt/media/4kmovies` / `/mnt/media/4ktv`, and set the explicit **"Enable 4K"** toggle. Without it, everything routes to the 1080p server.

### 8.5 Tailnet hygiene

Failed/destroyed container builds can leave **stale tailnet nodes** (an offline `beara` ghost held the clean name, forcing the live node to register as `beara-1`). Delete stale/offline nodes from the admin console, then rename the live node to the clean name. Keep the tailnet machine list reflecting reality — it *is* the family access map.

---

## 9. Quality profiles & custom formats

**Strategy:** storage is not the constraint (40 TB+). Optimise for **direct-play** (avoid transcode), not for size.

- **1080p Remote** (the default for family streaming): prefer **x264** for universal device compatibility; prefer compatible lossy audio (DD+/EAC3, AC-3, AAC). Used for `/mnt/media/movies`, `tv`, `kids`, `kidstv`, `documentaries`.
- **2160p Quality** (local living-room): prefer **x265/HEVC** (4K is HEVC), **HDR** (Dolby Vision / HDR10+), and **Atmos/TrueHD** for the Sonos Arc Ultra. Used for `/mnt/media/4kmovies`, `4ktv`.

### 9.1 The mechanism

Custom formats are **global definitions**; the **scores are assigned per Quality Profile**. The same eleven formats are scored differently in each profile — that is the whole trick. Resolution is handled by the profile (a 1080p profile only allows 1080p qualities), so the codec/audio/HDR formats are resolution-agnostic.

### 9.2 Scoring table

| Custom Format | 1080p Remote | 2160p Quality |
|---|---:|---:|
| x264 (AVC) | +25 | 0 |
| x265 (HEVC) | −15 | +25 |
| Dolby Vision | 0 | +30 |
| HDR10+ | 0 | +20 |
| HDR (any) | 0 | +10 |
| Dolby Atmos | +10 | +40 |
| TrueHD | −5 | +30 |
| DTS-HD MA / DTS:X | −5 | +20 |
| DD+ / EAC3 | +20 | +15 |
| AC-3 (Dolby Digital) | +10 | 0 |
| AAC | +10 | 0 |

> **DD+/EAC3 is the sweet spot for the Sonos Arc Ultra:** lossy (direct-plays remotely with no audio transcode) **and** carries Atmos metadata to the soundbar. TrueHD is the lossless 4K-tier equivalent — reserved for 2160p because it transcodes when streamed remotely. The Sonos Ray (optical) auto-downmixes to Dolby Digital; nothing to configure.

The eleven importable custom-format JSON blocks are in **Appendix A**, and shipped as the companion file `radarr-sonarr-custom-formats.md`.

### 9.3 Quality Definitions

The real lean/size lever is **Settings → Quality → Quality Definitions** (min/max MB-per-minute per tier). With 40 TB the maxes can stay generous; if a profile keeps grabbing nothing, the min/max is too tight.

---

## 10. Known quirks and gotchas

The reusable lessons surfaced by this build — worth carrying to every future container.

| # | Gotcha | Detail / fix |
|---|---|---|
| 1 | **New CTs inherit the host Tailscale resolver** | Fresh containers get `nameserver 100.100.100.100` (MagicDNS), which only resolves tailnet names → public lookups fail silently. Set `pct set <id> -nameserver 10.0.0.10` at create time. |
| 2 | **`pct set -nameserver` needs a container reboot** | It rewrites the *config*; `/etc/resolv.conf` is only regenerated on container **start**. An app restart is not enough — `pct reboot <id>`. |
| 3 | **Helper-script uv venvs put Python under `/root`** | uv-managed interpreter lives in `/root/.local/...` (mode `700`); a non-root service user cannot traverse `/root` → `203/EXEC Permission denied`. Relocate with `cp -rL` to `/opt`, re-point the venv symlink at the versioned dir. |
| 4 | **Recursive chown / non-root owner exposes binary perms** | `chown -R` on a venv, or simply running a binary that lives under a `700` dir as non-root, surfaces as a permission/exec failure. Diagnose the full chain with `namei -l <path>`. |
| 5 | **In-container Tailscale needs a TUN device** | Unprivileged LXC can't create `/dev/net/tun`; `tailscaled` crash-loops. Add `lxc.cgroup2.devices.allow: c 10:200 rwm` + the `/dev/net/tun` bind to the CT config, reboot. |
| 6 | **Jellyseerr is now Seerr** | The community-scripts `jellyseerr.sh` is a deprecated stub that 404s mid-install (redirects to the merged **Seerr** project). Use `ct/seerr.sh`; the service is **`seerr`** (not `jellyseerr`); config at `/etc/seerr/seerr.conf`; defaults Debian 13, 4 cores/4 GB. |
| 7 | **Truncated helper-script run = false success banner** | An interrupted script (or upstream 404) can still print "✔️ Completed successfully" with **nothing installed**. Always verify the service exists: `systemctl is-active <svc>`; an empty `/opt` confirms a failed install. |
| 8 | **Installer ports differ from docs** | SAB = **7777**, Prowlarr = **9696** (not 8080). Check the script's final URL banner. |
| 9 | **ZFS atomic move only within one dataset** | Cross-dataset import = copy+delete. Downloads' *complete* dir must share the library's dataset (`tank/Media`). A *child* dataset does not count — use plain directories. |
| 10 | **systemd ExecStart override** | A drop-in `ExecStart=` *appends*; precede it with a blank `ExecStart=` to clear the original, else the unit fails with two start commands. |
| 11 | **SAB category Folder/Path** | Leave blank. A path here makes SAB nest by category and can orphan a second copy; the arr apps import by API regardless. |
| 12 | **Servarr non-root install dir ownership** | `chown` `/opt/Sonarr` and `/opt/Radarr` (the install dir) too, not just the data dir — the in-app updater rewrites the install dir. |
| 13 | **`pct exec` is node-local** | Containers on pve2 (117 Prowlarr, 118 Seerr) are invisible to `pct …` run on pve. Use `ssh pve2 "pct …"` or run on the owning node. |

---

## 11. Operational runbook

### 11.1 Health check

```bash
# pve:
for pair in "114 sabnzbd 7777" "115 sonarr 8989" "116 radarr 7878"; do
  set -- $pair; echo "=== $2 ==="; pct exec "$1" -- bash -c "systemctl is-active $2; ss -ltnp | grep $3"
done
# pve2:
ssh pve2 'for pair in "117 prowlarr 9696" "118 seerr 5055"; do set -- $pair; echo "=== $2 ==="; pct exec "$1" -- bash -c "systemctl is-active $2; ss -ltnp | grep $3"; done'
```

### 11.2 Download-area hygiene

`/mnt/media/usenet/complete` should be empty between jobs. Stray `_UNPACK_…` (stalled extraction) or `…​.N` (re-download dedupe) folders indicate failed/aborted grabs:

```bash
ls -la /tank/Media/usenet/complete/        # should be ~empty
# if genuinely stalled (check Sonarr/Radarr Activity → Queue/History first):
rm -rf "/tank/Media/usenet/complete/<stuck-folder>"
```

Ensure SAB's "delete completed / history" switches are on so this self-tidies.

### 11.3 Updates

- **arr apps:** in-app updater (works because the install dir is owned by `media` — §4.2).
- **SAB / Seerr / Prowlarr containers:** community-scripts `update` command *inside* the container (it is a shell function the script installs, not a standalone binary — `pct exec <id> -- update` will not find it; enter the container).
- **After any container update:** re-verify DNS (`getter hosts thetvdb.com`) and, for SAB, that the uv-Python symlink survived.

### 11.4 Post-router-firmware checklist

1. Confirm the per-device VPN still binds `10.0.0.35` to the commercial VPN (UK exit).
2. Confirm `.35` DHCP reservation intact.
3. SAB → Test Server (Newshosting) green.

### 11.5 Adding a title (manual path)

Sonarr/Radarr → Add → pick root folder + **profile** (`1080p Remote` or `2160p Quality`) → interactive search → grab. Or, for family, via Seerr (auto-routes by the per-server defaults).

---

## 12. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Sonarr/Radarr "Couldn't find any results" at **Add** | Container DNS on tailnet resolver (Gotcha 1/2) | `pct set <id> -nameserver 10.0.0.10`; `pct reboot`; confirm `getent hosts thetvdb.com` |
| "No download client is available" (arr System health) | SAB client not synced from Prowlarr | Prowlarr → Apps → Sync App Indexers; or add SABnzbd directly in the arr app |
| SAB server Test: *"address … is not valid"* | host:port in the Host field | Split — Host `news.newshosting.com`, Port `563`, SSL ticked |
| SAB can't connect / "connecting to server" forever | No provider configured, or DNS dead in CT | Add Newshosting; check `getent hosts news.newshosting.com` |
| SAB service `failed`, `203/EXEC Permission denied` | uv Python under `/root` (Gotcha 3) | Relocate interpreter to `/opt` with `cp -rL`; re-point symlink |
| `tailscaled` crash-loops | No TUN device (Gotcha 5) | Add `/dev/net/tun` to CT config; reboot |
| Seerr/Jellyseerr install 404 / empty `/opt` | `jellyseerr.sh` deprecated (Gotcha 6) | Destroy CT; re-run `ct/seerr.sh`; service is `seerr` |
| `pct config <id>` "does not exist" | Container on the other node (Gotcha 13) | Run on owning node / `ssh pve2` |
| Imported file owned `100000` not `101000` | arr running as root, override didn't take | Re-apply media user + `User=media` drop-in (§4.2); reboot |
| Browser "refused to connect" to a service | Wrong port (e.g. `:80` instead of `:5055`/`:7777`/`:9696`) | Use the correct port; "refused" (RST) = reachable but nothing on that port |
| Imports slow / space briefly doubles | complete dir on a different dataset than library | Move complete dir under `tank/Media` (§3) |

---

## 13. Family usage

1. **Request, don't download.** Open `http://beara.<tailnet>.ts.net:5055`, sign in (Plex account), search, click **Request**. Adults' requests auto-approve; children's go to a queue for approval.
2. **4K vs HD.** Where offered, "Request in 4K" pulls the 2160p/HDR/Atmos copy to the living-room library; the default is the HD copy optimised for streaming anywhere.
3. **It just appears in Plex.** Once granted and downloaded, the title shows up in the relevant Plex library automatically.
4. **Genre shelves (Plex Collections).** Horror and Christmas are **Plex Collections**, not separate libraries — one physical file, surfaced as a Home-screen row:
   - *Horror* — Smart Collection, rule `Genre is Horror`; auto-populates new horror grabs.
   - *Christmas* — Smart Collection, rule `Label is Christmas` (Plex has no Christmas genre); add the `Christmas` label to new titles.
   - Pin to Home via the collection's **Visibility / Pin to Home**. A Collection is a Home row, *not* a left-sidebar library — that is the trade for one-file-many-shelves.
   - After folding genre folders into Movies: scan the Movies library, then delete the now-redundant old libraries (deleting a *library* does not delete files).

---

## 14. Explain like I'm 5

There's a robot butler for films and telly. When someone in the family wants to watch something, they ask the **butler at the front desk** (Seerr) — that's the only person they ever meet. The butler tells the **TV-finder** (Sonarr) or **film-finder** (Radarr) to go look. Those two ask the **librarian** (Prowlarr) which knows every catalogue, finds the right parcel, and hands it to the **postman** (SABnzbd), who fetches it from the big warehouse in the sky (Usenet, via Newshosting). The postman opens the parcel in a **messy back room** (the scratch disk) so the mess never touches the nice shelves, then slides the finished film onto the **right shelf** (the Plex library) so quickly it's like magic — because the back room and the shelves are in the same building (one ZFS dataset), the butler doesn't have to carry it across the street. Everyone in the house can only talk to the front-desk butler; they can't wander into the back rooms.

---

## 15. References

### The automation apps (Servarr / SAB / Seerr)
- Sonarr — https://wiki.servarr.com/sonarr
- Radarr — https://wiki.servarr.com/radarr
- Prowlarr — https://wiki.servarr.com/prowlarr
- SABnzbd documentation — https://sabnzbd.org/wiki/
- Seerr (unified Jellyseerr/Overseerr) — https://docs.seerr.dev/ · GitHub: https://github.com/seerr (release notes / migration: https://docs.seerr.dev/blog/seerr-release)

### Quality profiles & custom formats
- TRaSH Guides (custom formats, quality definitions, folder structure) — https://trash-guides.info/
- TRaSH — Radarr/Sonarr custom formats — https://trash-guides.info/Radarr/Radarr-collection-of-custom-formats/
- TRaSH — recommended folder structure (atomic moves / hardlinks) — https://trash-guides.info/File-and-Folder-Structure/

### Proxmox / containers / Tailscale
- Community Helper Scripts — https://community-scripts.github.io/ProxmoxVE/
- Jellyseerr→Seerr migration issues (context for Gotcha 6) — https://github.com/community-scripts/ProxmoxVE/issues/11961 · https://github.com/community-scripts/ProxmoxVE/issues/12529
- PVE Unprivileged LXC — https://pve.proxmox.com/wiki/Unprivileged_LXC_containers
- PVE Linux Container (bind mounts) — https://pve.proxmox.com/wiki/Linux_Container
- Tailscale — subnet routers & MagicDNS — https://tailscale.com/kb/1019/subnets · https://tailscale.com/kb/1081/magicdns
- Tailscale in LXC / TUN device — https://tailscale.com/kb/1130/lxc-unprivileged

### Provider / indexer
- Newshosting — https://www.newshosting.com/
- NZBGeek — https://nzbgeek.info/

### ZFS
- OpenZFS documentation — https://openzfs.github.io/openzfs-docs/

### Companion documents (RC COMMS suite)
- `proxmox-infrastructure-INTERNAL.md` — ulster cluster master documentation
- `nextcloud-vm-INTERNAL.md`, `eurooffice-nextcloud-INTERNAL.md`, `hybrid-llm-runbook-TECHNICAL.md`
- `radarr-sonarr-custom-formats.md` — importable custom-format JSON (companion to §9 / Appendix A)

---

## 16. Appendices

### Appendix A — Custom format JSON (import each block; Radarr & Sonarr)

Import via Settings → Custom Formats → **+** / Import (one object per paste). Full file: `radarr-sonarr-custom-formats.md`.

```json
{ "name": "x264 (AVC)", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "x264 / h264 / AVC", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "(?:x|h)[ .]?264|\\bAVC\\b" } } ] }
```
```json
{ "name": "x265 (HEVC)", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "x265 / h265 / HEVC", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "(?:x|h)[ .]?265|\\bHEVC\\b" } } ] }
```
```json
{ "name": "Dolby Vision", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "Dolby Vision", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\b(dolby[ .]?vision|do?vi)\\b|\\bDV\\b" } } ] }
```
```json
{ "name": "HDR10+", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "HDR10+", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\bHDR10(\\+|Plus|P)\\b" } } ] }
```
```json
{ "name": "HDR (any)", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "HDR", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\bHDR(10)?\\b" } } ] }
```
```json
{ "name": "Dolby Atmos", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "Atmos", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\batmos\\b" } } ] }
```
```json
{ "name": "TrueHD", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "TrueHD", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\btrue[ .]?hd\\b" } } ] }
```
```json
{ "name": "DTS-HD MA / DTS-X", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "DTS-HD MA / DTS-X", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\bDTS[ .-]?HD[ .-]?MA\\b|\\bDTS[ .:-]?X\\b" } } ] }
```
```json
{ "name": "DD+ / EAC3", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "DD+ / EAC3", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\b(E[ .-]?AC[ .-]?3|EAC3|DDP|DD\\+)\\b" } } ] }
```
```json
{ "name": "AC-3 (Dolby Digital)", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "AC-3", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\bAC[ .-]?3\\b" } } ] }
```
```json
{ "name": "AAC", "includeCustomFormatWhenRenaming": false,
  "specifications": [ { "name": "AAC", "implementation": "ReleaseTitleSpecification",
    "negate": false, "required": false, "fields": { "value": "\\bAAC\\b" } } ] }
```

### Appendix B — CLI quick reference

| Task | Command |
|---|---|
| Container health (active + port) | `pct exec <id> -- bash -c 'systemctl is-active <svc>; ss -ltnp \| grep <port>'` |
| Set CT DNS (then reboot) | `pct set <id> -nameserver 10.0.0.10 && pct reboot <id>` |
| Add bind-mount | `pct set <id> -mpN /tank/Media,mp=/mnt/media` |
| Walk symlink/permission chain | `namei -l /opt/sabnzbd/venv/bin/python` |
| Confirm import ownership | `ls -lnR "/tank/Media/tv/<Title>" \| grep -v '101000 101000'` |
| Query a pve2 container from pve | `ssh pve2 "pct exec <id> -- <cmd>"` |
| Add TUN device | append `lxc.mount.entry: /dev/net/tun …` to `/etc/pve/lxc/<id>.conf`; reboot |

### Appendix C — Container quick map

| CTID | Host | IP | Port | Node | Service | uid |
|---|---|---|---|---|---|---|
| 114 | shanlieve | 10.0.0.35 | 7777 | pve | SABnzbd | media/1000 |
| 115 | doan | 10.0.0.36 | 8989 | pve | Sonarr | media/1000 |
| 116 | errigal | 10.0.0.37 | 7878 | pve | Radarr | media/1000 |
| 117 | conavalla | 10.0.0.38 | 9696 | pve2 | Prowlarr | (default) |
| 118 | beara | 10.0.0.39 | 5055 | pve2 | Seerr | (default) |

### Appendix D — Backlog (open items at time of writing)

- Build the two Quality Profiles (`1080p Remote`, `2160p Quality`) and assign the Appendix A scores; set Seerr per-server defaults + 4K flags.
- Add the four new secrets to **Keeper**: SAB API key, NZBGeek API key, Newshosting password, arr API keys.
- Set **`.35` DHCP reservation** on the router (VPN binding stability).
- Investigate the separate `tank/replica/immich-library` sanoid CRIT (no daily/monthly snapshots — possible stalled replication; unrelated to this build).
- Optional: relocate `tank/scratch` to NVMe when MS-01 nodes land.

### Appendix E — Change log

| Date | Version | Change |
|---|---|---|
| 22 Jun 2026 | 1.0 | Initial build & documentation. Five LXCs created (114 SAB, 115 Sonarr, 116 Radarr on pve; 117 Prowlarr, 118 Seerr on pve2). Atomic-move storage design (complete in `tank/Media`, incomplete in `tank/scratch`). `media`/101000 ownership model. Newshosting provider + NZBGeek indexer wired via Prowlarr. Per-device VPN (UK exit) binding for SAB. Seerr family portal on Tailscale; per-container Tailscale removed from doan/errigal. Custom-format strategy (x264-remote / Atmos-2160p). Gotchas 1–13 captured. End-to-end grab verified (The Good Place → atomic move → Plex; ownership held at 101000). |

### Appendix F — Glossary

| Term | Meaning |
|---|---|
| **NZB** | Index file describing where binary parts live on Usenet (the "what to fetch") |
| **Provider / news server** | Where binaries are actually downloaded from (Newshosting); SSL on 563 |
| **Indexer** | Searchable NZB catalogue (NZBGeek); finds, does not serve |
| **Prowlarr** | Indexer manager — one place for indexers + download clients, syncs to the arr apps |
| **Servarr** | Collective name for Sonarr/Radarr/Prowlarr/etc. |
| **Seerr** | Unified Jellyseerr/Overseerr request portal |
| **Atomic move** | Instant `rename()` within one filesystem; no copy. ZFS: only within one dataset |
| **Hardlink** | Second directory entry for one file; only within one dataset (not needed for Usenet) |
| **par2** | Parity recovery data; repairs incomplete/corrupt Usenet downloads |
| **setgid (2775)** | Dir bit making new files inherit the dir's group — keeps ownership consistent |
| **UID remap / offset 100000** | Unprivileged LXC namespace: container uid 1000 = host uid 101000 |
| **TUN** | Virtual network device Tailscale needs; must be granted to unprivileged LXCs |
| **Per-device VPN policy** | Router feature: route specific devices through a VPN tunnel by IP |
| **MagicDNS** | Tailscale's tailnet name resolution (`<host>.<tailnet>.ts.net`) |

---

*End of document. Maintain via PR / commit; tag releases by date and stack version.*
