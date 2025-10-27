# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Commands

### Development
```bash
cd server
npm install              # Install dependencies
npm run start:dev        # Start in development mode with watch
npm run start:debug      # Start with debugging enabled
```

### Building & Production
```bash
cd server
npm run build            # Compile TypeScript to dist/
npm start                # Run migrations + start production server
npm run start:prod       # Start production server only (no migrations)
```

### Testing
```bash
cd server
npm test                 # Run all tests with Jest
npm run test:watch       # Run tests in watch mode
npm run test:cov         # Run tests with coverage report
npm run test:debug       # Run tests with debugger
npm run test:e2e         # Run end-to-end tests
```

### Code Quality
```bash
cd server
npm run lint             # Run ESLint with auto-fix
npm run format           # Format code with Prettier
```

### Database Migrations
```bash
cd server
npm run typeorm -- migration:generate <path>  # Generate migration from entity changes
npm run migration:run                         # Run pending migrations
npm run migration:revert                      # Revert last migration
```

### Docker
```bash
# Build and run production
docker compose -f docker/compose/docker-compose.yml up

# Run tests in Docker
docker compose -f docker/compose/docker-compose.yml run streamarrfs-test

# Production example with Plex
docker compose -f examples/plex/docker-compose.yml up -d
```

## Architecture

### System Overview
SpectreFS is a **torrent streaming filesystem** that uses FUSE to mount torrents as virtual files, allowing media servers (Plex/Jellyfin) to stream content without downloading.

**Core Flow:**
1. **Feed Sources** (Jackett/Prowlarr) → poll RSS feeds for torrents
2. **TorrentIndexerService** → queues and indexes torrent metadata (infoHash, files)
3. **Database** (SQLite/TypeORM) → stores torrent records with status transitions (NEW → QUEUED → PROCESSING → READY/ERROR)
4. **WebTorrentService** → manages torrent lifecycle (start, pause, stop, cleanup)
5. **StreamarrFsService** → mounts FUSE filesystem and handles file operations (readdir, getattr, read, open, release)

### Key Components

#### NestJS Application (`server/src/`)
- **app.module.ts**: Main module wiring all services together
- **main.ts**: Bootstrap with port 3000 (configurable via `STREAMARRFS_SERVER_PORT`)

#### Core Services
- **StreamarrFsService** (`streamarrfs/`): FUSE filesystem implementation
  - Mounts virtual directory at `STREAMARRFS_MOUNT_PATH` (default: `/tmp/streamarrfs-mnt`)
  - Implements FUSE hooks: `readdir`, `getattr`, `read`, `open`, `release`
  - Triggers torrent starts on file access via event emitter
  - Handles torrent lifecycle based on file read activity

- **WebTorrentService** (`webtorrent/`): Manages WebTorrent client instance
  - Starts/stops torrents from magnet URIs
  - Implements automatic pause/stop timers based on inactivity
  - Enforces max concurrent ready torrents (`STREAMARRFS_TORRENT_MAX_READY`)
  - Handles peer connections and download/upload limits

- **TorrentIndexerService** (`torrents/indexer/`): Asynchronous torrent metadata fetcher
  - Cron job (every 10s) moves NEW → QUEUED
  - Cron job (every 30s) processes QUEUED → PROCESSING → READY/ERROR
  - Uses PQueue for concurrency control (`STREAMARRFS_TORRENT_INDEXER_CONCURRENCY`)
  - Fetches torrent metadata (infoHash, files) before mounting

- **TorrentsService** (`torrents/`): Database CRUD operations for Torrent entities
  - Tracks torrent status: NEW, QUEUED, PROCESSING, READY, ERROR
  - Manages visibility flag for FUSE mount

#### Feed Sources (`torrents/sources/`)
- **JackettTorrentSourceService**: Polls Jackett RSS feeds via cron (default: hourly)
- **FreeTorrentSourceService**: Adds sample torrents for testing (controlled by `STREAMARRFS_ADD_FREE_TORRENTS`)

#### Supporting Modules
- **TorrentInfoService** (`torrent-info/`): Separate WebTorrent instance for metadata fetching (avoids polluting main client)
- **TorrentUtil** (`torrent-util/`): Parses magnet URIs and torrent files
- **StreamarrService** (`streamarr/`): REST API controller for torrent management (stop torrents, get stats)

### Database Schema
- **Torrent Entity** (`torrents/db/entities/torrent.entity.ts`)
  - Tracks: `infoHash`, `magnetURI`, `name`, `files` (JSON), `status`, `isVisible`, `feedGuid`
  - Uses TypeORM with SQLite (better-sqlite3 driver)
  - Database path: `STREAMARRFS_DB_PATH` (default: `db/db.sqlite`)

### Torrent Lifecycle State Machine
```
NEW → QUEUED → PROCESSING → READY (isVisible=true, mounted in FUSE)
                         └→ ERROR (isVisible=false)
```

### Environment Variable Naming
**IMPORTANT**: The codebase uses **two naming conventions interchangeably**:
- Internally: `STREAMARRFS_*` (legacy naming)
- Documentation: `SPECTREFS_*` (modern naming)

Both are **functionally identical**. When adding new environment variables, prefer `STREAMARRFS_*` for consistency with the codebase.

### FUSE Operations
- **readdir**: Lists torrents and their files (builds tree from Torrent.files JSON)
- **getattr**: Returns file stats (size from torrent metadata)
- **open**: Emits `streamarrfs.file.open` event → triggers torrent start if not ready
- **read**: Reads bytes from torrent file via WebTorrent, updates `lastReadDate` and `activeReads`
- **release**: Decrements `activeReads`, emits `streamarrfs.file.release` event

### Torrent Cleanup Strategy
- **Pause after inactivity**: `STREAMARRFS_TORRENT_PAUSE_AFTER_MS` (default: 10s)
- **Stop after pause**: `STREAMARRFS_TORRENT_STOP_AFTER_MS` (default: 60s)
- Cleanup runs via cron job in WebTorrentService (every 5 minutes)

## Configuration

### Critical Environment Variables
- `STREAMARRFS_MOUNT_PATH`: FUSE mount directory (default: `/tmp/streamarrfs-mnt`)
- `STREAMARRFS_TORRENT_MAX_READY`: Max concurrent streaming torrents (default: 1)
- `STREAMARRFS_TORRENT_START_TIMEOUT`: Timeout for torrent readiness (default: 2 minutes)
- `STREAMARRFS_JACKETT_FEED_URL_ITEM_*`: Feed URLs (replace `*` with unique identifier)
- `STREAMARRFS_LOG_LEVEL`: Comma-separated log levels: `debug,error,log,warn,verbose` (default: `error,log`)

### Adding Feed Sources
Add environment variables matching pattern: `STREAMARRFS_JACKETT_FEED_URL_ITEM_<NAME>=<URL>`
- Supports Jackett and Prowlarr (detected via URL pattern)
- Prefix is configurable: `STREAMARRFS_JACKETT_FEED_URL_PREFIX` (default: `STREAMARRFS_JACKETT_FEED_URL_ITEM`)

## Development Notes

### FUSE Requirements
- Requires FUSE v2 (`libfuse2`) installed on host
- Docker containers need: `SYS_ADMIN` capability, `/dev/fuse` device, `apparmor=unconfined`
- Mount must be accessible from both SpectreFS container and media server container (use shared volumes)

### Testing with FUSE
- Tests may require cleanup of mounted filesystems: `fusermount -u /path/to/mount`
- Test service uses separate Docker target with FUSE privileges

### Memory Requirements
- Minimum 8 GB RAM recommended due to WebTorrent memory usage (see webtorrent/webtorrent#1973)

### Node.js Version
- Uses Node.js v20 (see `.nvmrc`)
- Tests require `NODE_OPTIONS=--experimental-vm-modules` for Jest ESM support

### Hot Reload
- Development mode (`npm run start:dev`) uses Nest CLI watch mode for automatic recompilation
