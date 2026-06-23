# Radarr / Sonarr Custom Formats — Direct-Play & Atmos Strategy

**Goal:** 1080p optimised for remote direct-play (x264 + compatible audio);
2160p optimised for local quality (HEVC + HDR + Atmos/TrueHD). Storage is not
the constraint (40TB+); transcode-avoidance is.

Applies to **both** Radarr and Sonarr — the regexes and schema are identical.
Sonarr 4+ and all current Radarr support custom formats.

---

## How to import

1. Settings → **Custom Formats** → **+** (or the **Import** button).
2. Paste **one** JSON block below → Save.
3. Repeat for each format.
4. Then build the two Quality Profiles (below) and assign the scores from the table.

> Resolution is handled by the **Quality Profile** (a 1080p profile only allows
> 1080p qualities, a 2160p profile only 2160p). The custom formats are therefore
> resolution-agnostic and are **scored per profile** — the same format scores
> positive in one and negative in another. This is the whole trick.

---

## Scoring table

| Custom Format         | 1080p Remote | 2160p Quality |
|-----------------------|:------------:|:-------------:|
| x264 (AVC)            |     +25      |       0       |
| x265 (HEVC)           |     −15      |      +25      |
| Dolby Vision          |      0       |      +30      |
| HDR10+                |      0       |      +20      |
| HDR (any)             |      0       |      +10      |
| Dolby Atmos           |     +10      |      +40      |
| TrueHD                |     −5       |      +30      |
| DTS-HD MA / DTS:X     |     −5       |      +20      |
| DD+ / EAC3            |     +20      |      +15      |
| AC-3 (Dolby Digital)  |     +10      |       0       |
| AAC                   |     +10      |       0       |

**Profile build:**
- **1080p Remote** — allow WEBDL-1080p, Bluray-1080p (and WEBRip-1080p). Assign
  the left-column scores. Use for `/mnt/media/movies`, `/mnt/media/tv`,
  `/mnt/media/kids`, `/mnt/media/kidstv`, `/mnt/media/documentaries`.
- **2160p Quality** — allow WEBDL-2160p, Bluray-2160p, Remux-2160p. Assign the
  right-column scores. Use for `/mnt/media/4kmovies`, `/mnt/media/4ktv`.

Assign root folder + profile at add-time (or set Jellyseerr's per-server defaults).

> **DD+ / EAC3 is the sweet spot for your gear:** lossy (direct-plays remotely
> with no audio transcode) *and* carries Atmos metadata to the Sonos Arc Ultra.
> TrueHD is the lossless 4K-tier equivalent — reserved for 2160p because it
> transcodes when streamed remotely. The Sonos Ray (optical) auto-downmixes to
> Dolby Digital; nothing to configure for it.

---

## Custom format JSON (import each block)

### x264 (AVC)
```json
{
  "name": "x264 (AVC)",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x264 / h264 / AVC",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "(?:x|h)[ .]?264|\\bAVC\\b" }
    }
  ]
}
```

### x265 (HEVC)
```json
{
  "name": "x265 (HEVC)",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "x265 / h265 / HEVC",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "(?:x|h)[ .]?265|\\bHEVC\\b" }
    }
  ]
}
```

### Dolby Vision
```json
{
  "name": "Dolby Vision",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Dolby Vision",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\b(dolby[ .]?vision|do?vi)\\b|\\bDV\\b" }
    }
  ]
}
```

### HDR10+
```json
{
  "name": "HDR10+",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "HDR10+",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bHDR10(\\+|Plus|P)\\b" }
    }
  ]
}
```

### HDR (any)
```json
{
  "name": "HDR (any)",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "HDR",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bHDR(10)?\\b" }
    }
  ]
}
```

### Dolby Atmos
```json
{
  "name": "Dolby Atmos",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "Atmos",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\batmos\\b" }
    }
  ]
}
```

### TrueHD
```json
{
  "name": "TrueHD",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "TrueHD",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\btrue[ .]?hd\\b" }
    }
  ]
}
```

### DTS-HD MA / DTS:X
```json
{
  "name": "DTS-HD MA / DTS-X",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "DTS-HD MA / DTS-X",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bDTS[ .-]?HD[ .-]?MA\\b|\\bDTS[ .:-]?X\\b" }
    }
  ]
}
```

### DD+ / EAC3
```json
{
  "name": "DD+ / EAC3",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "DD+ / EAC3",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\b(E[ .-]?AC[ .-]?3|EAC3|DDP|DD\\+)\\b" }
    }
  ]
}
```

### AC-3 (Dolby Digital)
```json
{
  "name": "AC-3 (Dolby Digital)",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "AC-3",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bAC[ .-]?3\\b" }
    }
  ]
}
```

### AAC
```json
{
  "name": "AAC",
  "includeCustomFormatWhenRenaming": false,
  "specifications": [
    {
      "name": "AAC",
      "implementation": "ReleaseTitleSpecification",
      "negate": false,
      "required": false,
      "fields": { "value": "\\bAAC\\b" }
    }
  ]
}
```

---

## Tuning notes

- These regexes are solid starting points, not TRaSH-grade exhaustive. Refine
  scores after watching a few weeks of grabs in Activity → History.
- The `\bDV\b` token in Dolby Vision can occasionally false-positive on release
  names; if you see odd matches, drop that alternation and keep only the
  `dolby vision|dovi` branch.
- If a profile keeps grabbing nothing, your **min/max size in Quality
  Definitions** is likely too tight — with 40TB you can leave maxes generous.
- For the Quick Sync transcoding upgrade (MS-01 nodes), revisit x265 scoring on
  the 1080p profile — once the cluster handles HEVC cheaply, the −15 penalty can
  be relaxed and you'd save space without remote-transcode pain.
