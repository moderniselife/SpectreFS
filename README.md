<p align="center">
  <img src="https://raw.githubusercontent.com/yourusername/spectrefs/main/assets/logo.png" alt="SpectreFS Logo" width="180"/>
</p>

<h1 align="center">SpectreFS 👻</h1>
<p align="center">
  <em>The spirit of <a href="https://github.com/puttyman/streamarrfs">Streamarrfs</a> reborn.</em><br/>
  Stream torrents directly through Plex, Jellyfin, or any media server — as if the files already existed.<br/>
  Powered by ⚡️ <a href="https://github.com/webtorrent/webtorrent">WebTorrent</a>.
</p>

<p align="center">
  <a href="https://github.com/moderniselife/SpectreFS/releases"><img src="https://img.shields.io/github/v/release/moderniselife/SpectreFS?style=flat-square&logo=github" /></a>
  <a href="https://hub.docker.com/r/moderniselife/spectrefs"><img src="https://img.shields.io/docker/pulls/moderniselife/spectrefs?style=flat-square&logo=docker" /></a>
</p>

---

## 🪄 What is SpectreFS?

SpectreFS is a **torrent streaming filesystem** — a spectral layer between your torrent indexers and your media server.  
It **mounts torrents as real files** using [FUSE](https://github.com/libfuse/libfuse), allowing Plex or Jellyfin to play videos instantly without downloading them first.

Born from the original [Streamarrfs](https://github.com/puttyman/streamarrfs) (which went quiet over a year ago), SpectreFS carries forward its spirit — rewritten, modernized, and actively maintained.

---

## 🧠 How It Works

1. **Watches Plex Watch List** to find new torrents
2. **Finds torrents** via your favorite indexer (e.g. [Jackett](https://github.com/Jackett/Jackett)).  
3. **Indexes and caches** torrent metadata into SQLite.  
4. **Mounts** a virtual directory through FUSE, simulating the files locally.  
5. **Streams only what’s needed** — SpectreFS launches the torrent and reads the requested portion on demand.  
6. **Auto-cleans** inactive torrents, keeping your system lean and uncluttered.

---

## ✨ Features

- Stream torrents directly via **Plex**, **Jellyfin**, or even **Nginx**.
- Smart torrent lifecycle: **pause → stop → cleanup** based on activity.  
- **Seek** through videos mid-stream.  
- **Feed polling** at customizable intervals.  
- Handles **duplicates** across multiple feeds.  
- Minimal resource usage — no long-term storage needed.  

---

## 🧩 Supported Indexers
- [Jackett](https://github.com/Jackett/Jackett) (more coming soon)

---

## ⚠️ Caveats

Media scanners like Plex and Jellyfin may auto-start torrents during library indexing.  
SpectreFS mitigates this with a configurable torrent concurrency limit (`SPECTREFS_TORRENT_MAX_READY`).  
If you hit the cap, new streams will pause until others stop. You can manage this via the built-in [Web GUI](http://{HOST}:3000/web).

---

## 🐋 Quick Start (Docker)

### Prerequisites
- **FUSE v2**
- **Docker + Compose v2**
- **x86_64 host**
- Minimum **8 GB RAM** (due to [WebTorrent issue #1973](https://github.com/webtorrent/webtorrent/issues/1973))

### Example setup

```bash
# 1. Install dependencies
sudo apt update
sudo apt install fuse libfuse2 libfuse-dev docker.io -y

# 2. Confirm FUSE is installed
ls /dev/fuse

# 3. Enter root
sudo su

# 4. Prepare mount directory
mkdir -p /tmp/spectrefs-tmp

# 5. Create a Docker Compose folder
mkdir -p /opt/spectrefs && cd /opt/spectrefs

# 6. Download example compose file
curl -O https://raw.githubusercontent.com/yourusername/spectrefs/main/examples/plex/docker-compose.yml

# 7. Start SpectreFS
docker compose up -d

```

### 🗂️ Adding the SpectreFS Mount as a Media Library

#### ⚙️ Plex Setup

The following guide is based on the [Plex Docker Compose example](https://raw.githubusercontent.com/yourusername/spectrefs/main/examples/plex/docker-compose.yml).

1. **Open your Plex Web UI**

   Based on the Docker setup, Plex should be accessible at:
   ```bash
   open http://{YOUR_HOST_IP}:32400/web
   ```

2. **Run through the [Plex Basic Setup Wizard](https://support.plex.tv/articles/200288896-basic-setup-wizard/)**  
   If Plex is freshly installed, complete the wizard until the media library setup step.

3. **Add the SpectreFS Mount**

   When prompted to add a media library:
   - Choose the appropriate library type (Movies, TV Shows, etc.)
   - Add this folder path:
     ```
     /spectrefs
     ```
     This is where SpectreFS mounts your active torrent streams.

4. **Optimize Plex Scanning (Recommended)**

   To prevent Plex from unnecessarily starting multiple torrents during library indexing, disable the following advanced options:
   - “Prefer artwork based on library language”
   - “Include related external content”
   - “Use local assets”
   - “Generate video preview thumbnails”

   These options cause excessive reads, which can trigger torrents to start and occupy your concurrency limit.

5. **Save and Test**

   Once saved, Plex will automatically detect files within `/spectrefs`.  
   You can test playback using the free sample torrents included in SpectreFS.

6. **Next Step: Add Indexers**  
   Continue below to [Adding Jackett Indexers](#-adding-jackett-indexers).

---

#### 💠 Jellyfin Setup

Coming soon — but the process mirrors Plex closely.  
Simply point Jellyfin’s media library to your SpectreFS mount directory (e.g. `/spectrefs`) and ensure the Docker container has access to that path.

A Jellyfin example configuration will be available in the next release.

---

### 🧭 Adding Jackett Indexers

1. **Set up Jackett**

   Run your Jackett instance (example):
   ```
   http://{YOUR_HOST_IP}:9117
   ```
   If you’re new to Jackett, follow this guide:  
   👉 [How to Set Up Jackett](https://www.rapidseedbox.com/blog/guide-to-jackett)

2. **Copy the RSS Feed URL**

   From your Jackett web interface, locate and copy the RSS link for your chosen indexer (e.g. YTS, 1337x, etc.).

3. **Add the Feed to SpectreFS**

   In your `docker-compose.yml`, add the feed URL under the environment section:

   ```yaml
   environment:
     - NODE_ENV=production
     - SPECTREFS_JACKETT_FEED_URL_ITEM_YTS=http://HOST:9117/api/v2.0/indexers/yts/results/torznab/api?apikey=KEY&t=search&cat=&q=
   ```

   Replace:
   - `YTS` with a unique name for this feed.
   - `KEY` with your Jackett API key.

   You can also append query parameters to fine-tune results.

4. **Restart SpectreFS**

   ```bash
   docker compose restart
   ```

5. **Enjoy**

   SpectreFS will now automatically poll your feeds, index the latest torrents, and make them instantly available through your virtual mount.

---

### ⚙️ Environment Variables

> Some variables are experimental and may not yet be fully documented.

| Variable | Description |
| --- | --- |
| `SPECTREFS_JACKETT_FEED_URL_PREFIX` | The prefix used to discover Jackett feed URLs. Default: `SPECTREFS_JACKETT_FEED_URL_ITEM` |
| `SPECTREFS_JACKETT_FEED_URL_ITEM_*` | Individual feed URLs (replace `*` with any unique identifier) |
| `SPECTREFS_LOG_LEVEL` | Comma-separated log levels: `debug,error,log,warn,verbose`. Default: `error,log` |
| `SPECTREFS_ADD_FREE_TORRENTS` | Adds free sample torrents on startup. Default: `true` |
| `SPECTREFS_TORRENT_PAUSE_AFTER_MS` | Time (ms) of inactivity before pausing a torrent. |
| `SPECTREFS_TORRENT_STOP_AFTER_MS` | Time (ms) before a paused torrent is stopped. |
| `SPECTREFS_TORRENT_MAX_READY` | Maximum number of torrents allowed to stream simultaneously. Default: `1` |
| `SPECTREFS_TORRENT_START_TIMEOUT` | Timeout for torrent readiness. Default: `2m` |
| `SPECTREFS_TORRENT_INDEXER_CONCURRENCY` | Maximum concurrent torrents being indexed. Default: `1` |
| `SPECTREFS_WEBTORRENT_MAX_CONNS` | Maximum peer connections per torrent. Default: `1` |

---

SpectreFS seamlessly turns your torrent feeds into a living, mountable library —  
files that don’t exist until you look at them. 👻
