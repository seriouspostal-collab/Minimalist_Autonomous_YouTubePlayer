# Minimalist Autonomous YouTube Player for Low-End PCs 🚀

A lightweight, autonomous, and completely seamless player for streaming YouTube playlists (such as background music and music videos) on older, low-spec laptops and mini-PCs (tested on **Intel Celeron N4000C**, Windows 10/11).

The project is entirely free of heavy dependencies (no Python, no heavy GUI shells) and optimized for maximum RAM and CPU efficiency.

---

## 💡 Configuration Features

* **No AV1 and heavy codecs:** Forces resolution limits up to 480p using hardware acceleration (`hwdec=auto`) and lightweight codecs (`avc1` / `vp09`), since older processors struggle with software decoding modern streams.
* **Built-in SponsorBlock in pure Lua:** Automatically skips unwanted intros (`intro`), outros (`outro`), and non-music talking segments in music videos (`music_offtopic`) without Python—API requests are executed directly via system `curl`.
* **Instant transitions:** Utilizes playlist prefetching (`prefetch-playlist=yes`) and caching (`cache-secs=50`) to eliminate dead air between tracks.
* **Playlist auto-refresh:** The player automatically re-requests a fresh track list from YouTube when the laptop wakes up (after closing/opening the lid) or when pressing the `r` key.

---

## 🛠 What to Download and Why

1. **`mpv` (v0.39+ / Portable build):**
The portable version of the player is recommended. It doesn't clutter the Windows registry, configures easily via the `portable_config` folder, and consumes minimal RAM.
*(Example: `mpv-x86_64-20261111-git-bcde6b93cc.7z` available via [zhongfly/mpv-winbuild releases](https://github.com/zhongfly/mpv-winbuild/releases))*
2. **`yt-dlp.exe`:**
The core engine for parsing direct stream URLs and playlists from YouTube. Must be placed in the same directory as `mpv.exe`.
*(Available at [yt-dlp/yt-dlp](https://github.com/yt-dlp/yt-dlp))*
3. **`curl.exe`:**
Used by the lightweight SponsorBlock Lua script to send fast HTTP requests to the API without installing heavy runtimes.
*(Built into modern versions of Windows 10/11 by default)*

---

## 📁 Folder Structure

All settings and scripts belong inside the player's portable configuration directory:

```text
C:\mpv\
│   mpv.com
│   mpv.exe
│   run.bat
│   yt-dlp.exe
└───portable_config
    │   input.conf
    │   mpv.conf
    └───scripts
            autorefresh.lua
            sponsorblock_minimal.lua

```

---

## ⚙️ Configuration Files

### 1. `mpv.conf`

Core performance, decoding, and cache settings:

```text
# Video hardware acceleration (uses Intel Celeron graphics for H.264 / VP9)
hwdec=auto

# Quality capping via yt-dlp (limited to 480p to save resources and ensure smooth kitchen playback)
ytdl-format="bestvideo[vcodec^=avc1][height<=?480]/bestvideo[vcodec^=vp09][height<=?480]+bestaudio/best[height<=?480]"
aid=1

# Save playback position on exit and keep player open after the track ends
save-position-on-quit=yes
keep-open=yes
idle=yes

# Caching to minimize startup latency
cache=yes
demuxer-max-bytes=300M
cache-secs=50
tls-verify=no
fs=yes
terminal=no
prefetch-playlist=yes

```

### 2. `input.conf`

Custom keybindings:

```text
# Volume control via mouse wheel or arrow keys
UP add volume 5
DOWN add volume -5

# Pause / Play
SPACE cycle pause

# Next / Previous track in playlist
ENTER playlist-next
BS playlist-prev

# Manual refresh & Quit
r script-binding manual-refresh
q quit

```

---

## 📜 Lua Scripts

### 1. `scripts/autorefresh.lua`

Handles manual playlist refreshing via the `r` key:

```lua
-- autorefresh.lua for mpv
-- Allows refreshing the playlist on the fly via hotkey

local mp = require 'mp'

-- Playlist refresh function
function refresh_playlist()
    mp.msg.info("Refreshing playlist...")
    mp.command("loadfile ${path} append-or-play")
end

-- Register command for hotkey binding
mp.register_script_message("refresh-playlist", refresh_playlist)

-- Bind 'r' key for manual refresh
mp.add_key_binding("r", "manual-refresh", refresh_playlist)

```

### 2. `scripts/sponsorblock_minimal.lua`

Optimized script for automatically skipping unwanted segments (runs via system `curl` with zero external dependencies):

```lua
local mp = require('mp')
local utils = require('mp.utils')

-- CATEGORY SETTINGS (true = skip, false = ignore)
local categories_config = {
    intro = true,          -- Intros / introductory animations
    outro = true,          -- Outros / credits / endcards
    music_offtopic = true  -- Non-music talking segments in music videos
}

local current_segments = {}
local fetched_video_id = nil

-- Function to extract video ID
local function get_video_id()
    local path = mp.get_property("path", "")
    local purl = mp.get_property("metadata/by-key/PURL", "")
    local ytdl_id = mp.get_property("ytdl-metadata/id", "")
    
    if ytdl_id ~= "" then return ytdl_id end

    local urls = {
        "youtu%.be/([%w-_]+)",
        "v=([%w-_]+)",
        "embed/([%w-_]+)"
    }
    for _, u in ipairs(urls) do
        local id = path:match(u) or purl:match(u)
        if id then return id end
    end
    return nil
end

-- Check playback position (triggered only on time change)
local function skip_ads(name, pos)
    if not pos or #current_segments == 0 then return end
    for _, seg in ipairs(current_segments) do
        if pos >= seg.start and pos < seg.ending - 0.01 then
            mp.set_property("time-pos", seg.ending + 0.01)
            mp.osd_message(string.format("⏩ Skipped (%s)", seg.category), 2)
            break
        end
    end
end

-- Request segments from SponsorBlock API via direct curl call
local function fetch_segments(video_id)
    if not video_id or video_id == fetched_video_id then return end
    fetched_video_id = video_id
    current_segments = {}

    -- Collect enabled categories
    local active_cats = {}
    for cat, enabled in pairs(categories_config) do
        if enabled then table.insert(active_cats, cat) end
    end
    if #active_cats == 0 then return end

    local cats_json = utils.format_json(active_cats)

    -- Call curl directly (no PowerShell, instant and lightweight)
    local res = utils.subprocess({
        args = {
            "curl", "-s", "-G",
            "https://sponsor.ajay.app/api/skipSegments",
            "--data-urlencode", "videoID=" .. video_id,
            "--data-urlencode", "categories=" .. cats_json
        }
    })

    if res.status ~= 0 or not res.stdout then return end

    -- Reliable JSON parsing using built-in mpv utilities
    local json = utils.parse_json(res.stdout)
    if type(json) == "table" then
        for _, item in ipairs(json) do
            if item.segment and item.category and categories_config[item.category] then
                table.insert(current_segments, {
                    start = item.segment[1],
                    ending = item.segment[2],
                    category = item.category
                })
            end
        end
    end

    if #current_segments > 0 then
        mp.osd_message("SponsorBlock: segments loaded: " .. #current_segments, 2)
        -- Enable timeline observation only when there are segments to skip
        mp.observe_property("time-pos", "native", skip_ads)
    end
end

-- Reset and initialize on new file load
mp.register_event("file-loaded", function()
    fetched_video_id = nil
    current_segments = {}
    mp.unobserve_property(skip_ads)
    
    mp.add_timeout(1.0, function()
        local id = get_video_id()
        if id then
            fetch_segments(id)
        end
    end)
end)

```

---

## 🚀 Quick Launch (`run.bat`)

To easily launch your YouTube playlist in the background without lingering console windows, create a file named **`run.bat`** in the root `C:\mpv\` folder:

```bat
@echo off
REM Run mpv to launch your YouTube playlist or stream URL
start /min "" C:\mpv\mpv.com "https://www.youtube.com/playlist?list=PL2BlfuqKJ9uyGgbUQ_1OdX_4e_trfdzEe"
timeout /t 22
exit

```
