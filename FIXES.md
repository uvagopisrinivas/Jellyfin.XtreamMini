# Jellyfin Xtream Plugin - Key Fixes & Features Reference

## Table of Contents

- [Empty Seasons/Episodes on TV Apps (Lazy Evaluation Fix)](#empty-seasonsepisodes-on-tv-apps-lazy-evaluation-fix)
- [Seasons Disappearing After Watching Episodes](#seasons-disappearing-after-watching-episodes)
- [Audio Track Selection for VOD (Movies)](#audio-track-selection-for-vod-movies)
- [VOD Duration & Premature Watched Status](#vod-duration--premature-watched-status)
- [Deployment & Release Process](#deployment--release-process)
- [OMV Deploy Script](#omv-deploy-script)

---

## Empty Seasons/Episodes on TV Apps (Lazy Evaluation Fix)

**Problem:** TV app clients (Android TV, Apple TV, etc.) show empty folders when browsing Series → Seasons → Episodes. The folders exist but contain no items.

**Root Cause:** LINQ `.Select()` returns a lazy `IEnumerable`. By the time TV app clients enumerate the results, the underlying data context (API response) is no longer available, so the enumeration yields nothing.

**Affected Methods in `Service/StreamService.cs`:**

### `GetSeasons()`

The season list returned from both code paths must be materialized immediately:

```csharp
// Use Seasons list as the source of truth instead of Episodes dictionary keys
if (series.Seasons != null && series.Seasons.Count > 0)
{
    // Convert to list immediately to avoid lazy evaluation issues with TV apps
    return series.Seasons.Select((Season season) => new Tuple<SeriesStreamInfo, int>(series, season.SeasonNumber)).ToList();
}

// Fallback to Episodes dictionary keys if Seasons list is empty
return series.Episodes.Keys.Select((int seasonId) => new Tuple<SeriesStreamInfo, int>(series, seasonId)).ToList();
```

### `GetEpisodes()`

Same pattern — materialize before returning:

```csharp
// Convert to list immediately to avoid lazy evaluation issues with TV apps
return episodes.Select((Episode episode) => new Tuple<SeriesStreamInfo, Season?, Episode>(series, season, episode)).ToList();
```

**Fix:** Append `.ToList()` to every `.Select()` call that returns channel items to Jellyfin's channel infrastructure. This forces immediate evaluation so the data is captured before the response context is disposed.

**Applies to:** Any method in `StreamService` that returns `IEnumerable<Tuple<...>>` to be consumed by `SeriesChannel.GetChannelItems()`.

---

## Seasons Disappearing After Watching Episodes

**Problem:** After watching all episodes in a season, that season disappears from the series folder. Single-season series vanish entirely after watching one episode.

**Root Cause:** The original code used `series.Episodes.Keys` to determine which seasons exist. When the Xtream API filters out watched episodes, the Episodes dictionary no longer contains that season's key, so the season disappears.

**Fix in `GetSeasons()`:**

Use `series.Seasons` list (which always contains all seasons regardless of watch state) as the primary source. Fall back to `Episodes.Keys` only when `Seasons` is null/empty:

```csharp
// Use Seasons list as the source of truth instead of Episodes dictionary keys
// This prevents seasons from disappearing when the API filters watched episodes
if (series.Seasons != null && series.Seasons.Count > 0)
{
    return series.Seasons.Select(...).ToList();
}

// Fallback to Episodes dictionary keys if Seasons list is empty
return series.Episodes.Keys.Select(...).ToList();
```

**Fix in `GetEpisodes()`:**

Add defensive check before accessing the Episodes dictionary to prevent crashes when a season exists in the Seasons list but has no episodes:

```csharp
// Check if the season exists in the Episodes dictionary before accessing
if (!series.Episodes.TryGetValue(seasonId, out ICollection<Episode>? episodes) || episodes == null || episodes.Count == 0)
{
    // Return empty list if season not found instead of crashing
    return new List<Tuple<SeriesStreamInfo, Season?, Episode>>();
}
```

---

## Summary Checklist

When implementing series channel browsing with Xtream API:

1. **Always `.ToList()` before returning** any LINQ projection that will be consumed by Jellyfin channel infrastructure
2. **Use `Seasons` list as source of truth** for available seasons, not `Episodes.Keys`
3. **Defensive `TryGetValue`** when accessing `Episodes[seasonId]` — the key may not exist
4. **Check for null/empty** on both `Seasons` list and episode collections


---

## Audio Track Selection for VOD (Movies)

**Goal:** Let users see and select between audio tracks in the Jellyfin player — using the real tracks embedded in the stream, no guessing from titles.

### Implementation

The simplest approach: let Jellyfin probe the stream and discover the actual audio tracks.

**1. `VodChannel` must NOT implement `IDisableMediaSourceDisplay`**

```csharp
// Just IChannel — no IDisableMediaSourceDisplay
public class VodChannel(...) : IChannel
```

Without `IDisableMediaSourceDisplay`, Jellyfin will probe the stream URL and discover all real audio/video/subtitle tracks automatically.

**2. `GetMediaSourceInfo()` — keep it minimal, enable probing**

```csharp
return new MediaSourceInfo()
{
    Container = extension,
    EncoderProtocol = MediaProtocol.Http,
    Id = ...,
    IsInfiniteStream = false,
    IsRemote = true,
    MediaStreams = [],  // empty — Jellyfin fills via probing
    Name = name ?? "default",
    Path = uri,
    Protocol = MediaProtocol.Http,
    SupportsDirectPlay = true,
    SupportsDirectStream = true,
    SupportsProbing = true,  // Jellyfin discovers real tracks
};
```

**3. No language parsing, no synthetic tracks, no PreferredAudioLanguage config**

Jellyfin probes the actual file and populates:
- All audio tracks with real codec, channels, language tags (if embedded in the file)
- Video stream info
- Subtitle tracks
- Duration

The player UI shows all discovered tracks and the user picks whichever they want.

### Key Points

- No title parsing needed — real tracks come from the file itself
- No `IDisableMediaSourceDisplay` — that interface prevents probing
- `SupportsProbing = true` is essential
- `MediaStreams` can be empty — Jellyfin fills it
- Duration, watched status, and progress all work automatically via probing
- If the source file has language metadata on its audio tracks, Jellyfin shows proper language labels in the player


---

## VOD Duration & Premature Watched Status

**Problem:** VOD movies show no duration and get marked as "watched" after a few seconds of playback.

**Root Cause (in this repo):** When `IDisableMediaSourceDisplay` is used with `SupportsProbing = false`, Jellyfin relies entirely on the plugin-provided `RunTimeTicks`. If that's not set (because `GetVodInfoAsync` wasn't called yet), Jellyfin sees duration as 0 → thinks the movie is complete immediately.

**In the new repo — no separate fix needed.**

Since we're using:
- No `IDisableMediaSourceDisplay`
- `SupportsProbing = true`
- Empty `MediaStreams` (let Jellyfin discover)

Jellyfin probes the actual stream URL and gets the real duration from the file headers. Duration shows correctly, progress tracking works, and watched status only triggers at the proper threshold (~90% watched).

**No `XtreamVodProvider` needed for duration.** The metadata provider is only useful if you want TMDB enrichment (posters, plot, genres). Duration and watched status are handled entirely by probing.


---

## Deployment & Release Process

### Building

```bash
dotnet build -c Release Jellyfin.Xtream/Jellyfin.Xtream.csproj
# Output: Jellyfin.Xtream/bin/Release/net9.0/Jellyfin.Xtream.dll
```

### Version Numbers — Update in 3 Places

1. `Jellyfin.Xtream/Jellyfin.Xtream.csproj` — `<AssemblyVersion>` and `<FileVersion>`
2. `build.yaml` — `version: "X.Y.Z.0"`
3. `README.md` — deploy script `VERSION=` and Version History section

### Release Steps

```bash
# 1. Build
dotnet build -c Release Jellyfin.Xtream/Jellyfin.Xtream.csproj

# 2. Commit & tag
git add -A
git commit -m "vX.Y.Z - Description"
git tag vX.Y.Z
git push origin master --tags

# 3. Create GitHub Release (tag alone is NOT enough)
zip -j /tmp/Jellyfin.Xtream-vX.Y.Z.zip Jellyfin.Xtream/bin/Release/net9.0/Jellyfin.Xtream.dll
gh release create vX.Y.Z /tmp/Jellyfin.Xtream-vX.Y.Z.zip \
  Jellyfin.Xtream/bin/Release/net9.0/Jellyfin.Xtream.dll \
  --title "vX.Y.Z" --notes "Description" --latest

# 4. Update repository.json on gh-pages
md5 /tmp/Jellyfin.Xtream-vX.Y.Z.zip  # Jellyfin uses MD5 for checksum verification, NOT SHA256
git stash
git checkout gh-pages
# Edit repository.json — add new entry at TOP of versions array:
# {
#   "version": "X.Y.Z.0",
#   "changelog": "...",
#   "targetAbi": "10.11.0.0",
#   "sourceUrl": "https://github.com/uvagopisrinivas/Jellyfin.XtreamMini/releases/download/vX.Y.Z/Jellyfin.Xtream-vX.Y.Z.zip",
#   "checksum": "<md5>",
#   "timestamp": "2026-XX-XXTXX:XX:XXZ"
# }
git add repository.json
git commit -m "Update repository.json for vX.Y.Z"
git push origin gh-pages
git checkout master
git stash pop
```

### Key Notes

- The `meta.json` file is required — Jellyfin won't load the plugin without it
- The GUID `5d774c35-8567-46d3-a950-9bb8227a0c5d` must match what's in `Plugin.cs`
- The zip file in the GitHub Release is required for Jellyfin's auto-update mechanism
- `repository.json` on `gh-pages` branch is what Jellyfin's plugin catalog reads
- Newest version must be first in the `versions` array
- On OMV NUC: config is at `/root/compose/jellyfin/config`
- First deploy: remove old catalog plugin folder first: `rm -rf "/root/compose/jellyfin/config/plugins/Jellyfin Xtream_0.8.1.0"`
- Also remove the old repo URL from Jellyfin (Dashboard → Plugins → Repositories) so it doesn't reinstall the old version

---

## OMV Deploy Script

Run this separately on the OMV GUI using the Scripts feature after a release is published on GitHub.

```bash
#!/bin/bash
set -e

VERSION="0.8.2"
PLUGIN_DIR="/root/compose/jellyfin/config/plugins/Jellyfin.Xtream_5d774c35-8567-46d3-a950-9bb8227a0c5d"

echo "=== Deploying Jellyfin Xtream v${VERSION} ==="

echo "Downloading DLL..."
rm -f /tmp/Jellyfin.Xtream.dll
wget -q -L -O /tmp/Jellyfin.Xtream.dll "https://github.com/uvagopisrinivas/Jellyfin.XtreamMini/releases/download/v${VERSION}/Jellyfin.Xtream.dll"

if [ ! -s /tmp/Jellyfin.Xtream.dll ]; then
  echo "ERROR: Download failed or file is empty"
  exit 1
fi
echo "Download OK ($(stat -c%s /tmp/Jellyfin.Xtream.dll) bytes)"

echo "Stopping Jellyfin..."
docker stop jellyfin

echo "Cleaning plugin directory..."
rm -rf "$PLUGIN_DIR"
mkdir -p "$PLUGIN_DIR"

echo "Installing plugin..."
cp /tmp/Jellyfin.Xtream.dll "$PLUGIN_DIR/"

cat > "$PLUGIN_DIR/meta.json" << EOF
{"Name": "Jellyfin Xtream","Guid": "5d774c35-8567-46d3-a950-9bb8227a0c5d","Version": "${VERSION}.0","TargetAbi": "10.11.0.0","Framework": "net9.0","Overview": "Stream content from an Xtream-compatible server.","Description": "Stream Live IPTV, Video On-Demand, and Series from an Xtream-compatible server using this plugin.","Category": "LiveTV","Owner": "uvagopisrinivas"}
EOF

chown -R 1000:100 "$PLUGIN_DIR"

echo "Starting Jellyfin..."
docker start jellyfin

echo "Waiting for startup..."
sleep 20

echo "=== Plugin status ==="
docker logs jellyfin --tail 100 2>&1 | grep -i "xtream\|plugin.*disabled\|error.*plugin" || echo "No Xtream-related log entries found"

echo "=== Done ==="
```

Update `VERSION` at the top before running. First time only: also remove the old plugin folder:

```bash
rm -rf "/root/compose/jellyfin/config/plugins/Jellyfin Xtream_0.8.1.0"
```
