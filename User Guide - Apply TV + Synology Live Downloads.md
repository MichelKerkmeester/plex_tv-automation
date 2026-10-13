# Apple TV + Synology Plex Automatic Download Setup

## Goal

Create this experience:

**Apple TV → Plex → Add to Watchlist → automatic download → appears in Plex**

After setup, normal usage requires no interaction with qBittorrent, Radarr, Sonarr, or the Synology interface.

The system works like this:

```text
Apple TV
   ↓
Plex
   ↓
Add to Watchlist
   ↓
Seerr
   ↓
Radarr / Sonarr
   ↓
qBittorrent
   ↓
Synology storage
   ↓
Plex Library
   ↓
Apple TV
```

Use torrent/indexer sources only for content you are legally entitled to download.

---

# 1. Components

You already have:

- Synology NAS
- Plex Media Server
- qBittorrent
- Apple TV
- Plex Apple TV app

Add:

### Seerr
The request-management layer.

It integrates with Plex and can automatically turn additions to your Plex Watchlist into requests.

### Radarr
Handles movies.

```text
Movie requested
→ searches configured sources
→ sends download to qBittorrent
→ imports finished file
→ renames it
→ puts it into Plex Movies
```

### Sonarr
Same system for TV shows.

### Prowlarr
Optional but recommended.

It centralizes compatible indexer/search-source configuration and connects those sources to Radarr and Sonarr.

---

# 2. Recommended folder structure

The important part is having downloads and finished media underneath one common filesystem path.

For example:

```text
/volume1/data/
├── torrents/
│   ├── movies/
│   └── tv/
│
└── media/
    ├── movies/
    └── tv/
```

Plex reads:

```text
/volume1/data/media/movies
/volume1/data/media/tv
```

qBittorrent downloads into:

```text
/volume1/data/torrents
```

Radarr imports movies into:

```text
/volume1/data/media/movies
```

Sonarr imports shows into:

```text
/volume1/data/media/tv
```

Keeping everything below a common `/data` mount is preferable for Docker. Radarr's documentation specifically recommends consistent paths because this allows hardlinks and atomic moves instead of unnecessarily copying large files.

---

# 3. Container paths

If these applications run through Synology Container Manager, give qBittorrent, Radarr and Sonarr access to the same structure.

For example:

```text
Synology                      Container

/volume1/data        →        /data
```

Then every application sees exactly the same paths:

```text
/data/torrents
/data/torrents/movies
/data/torrents/tv

/data/media
/data/media/movies
/data/media/tv
```

This is preferable to doing this:

```text
qBittorrent: /downloads
Radarr:      /downloads
Plex:        /movies
```

even if the underlying folders happen to correspond.

Using one consistent `/data` hierarchy greatly reduces import and permissions problems.

---

# 4. qBittorrent

Your existing qBittorrent installation can remain.

Make sure its Web UI is enabled because Radarr and Sonarr communicate with qBittorrent through it.

You'll need:

```text
Host:     Synology IP / container hostname
Port:     your qBittorrent Web UI port
Username: your qBittorrent username
Password: your qBittorrent password
```

Create or allow these categories:

```text
radarr
sonarr
```

A useful mapping is:

```text
radarr → /data/torrents/movies
sonarr → /data/torrents/tv
```

Radarr and Sonarr use categories so they only manage downloads belonging to them.

---

# 5. Install Radarr

Radarr normally runs on:

```text
http://SYNOLOGY-IP:7878
```

## Media root

Open:

**Settings → Media Management → Root Folders**

Add:

```text
/data/media/movies
```

## Connect qBittorrent

Open:

**Settings → Download Clients → + → qBittorrent**

Configure:

```text
Name:       qBittorrent
Host:       your qBittorrent host
Port:       qBittorrent Web UI port
Username:   your username
Password:   your password
Category:   radarr
```

Press:

**Test**

It should report a successful connection.

Radarr will now send downloads to qBittorrent and monitor them until they are ready to import.

---

# 6. Install Sonarr

Sonarr normally runs on:

```text
http://SYNOLOGY-IP:8989
```

Open:

**Settings → Media Management → Root Folders**

Add:

```text
/data/media/tv
```

Then:

**Settings → Download Clients → + → qBittorrent**

Configure:

```text
Name:       qBittorrent
Host:       your qBittorrent host
Port:       qBittorrent Web UI port
Username:   your username
Password:   your password
Category:   sonarr
```

Press:

**Test**

Sonarr officially recommends using a separate category so unrelated qBittorrent downloads don't appear in its activity queue.

---

# 7. Configure your search sources

Radarr and Sonarr still need somewhere to search.

This can be managed directly inside each application, but using Prowlarr makes management easier:

```text
Sources
   ↓
Prowlarr
   ├── Radarr
   └── Sonarr
```

Configure only sources you are authorized to use.

Once this works, test manually inside Radarr:

```text
Add Movie
→ choose movie
→ Search
```

You should see:

```text
Radarr
→ qBittorrent
→ download
→ import
→ /data/media/movies/...
```

Do the same once with Sonarr before adding Seerr.

---

# 8. Check importing

This part is important.

A torrent download should remain something like:

```text
/data/torrents/movies/Example.Movie.2026/
```

while Radarr creates the organized Plex version:

```text
/data/media/movies/Example Movie (2026)/
    Example Movie (2026).mkv
```

With hardlinks configured correctly, these can refer to the same underlying data rather than consuming twice the disk space.

Radarr supports hardlinking completed torrents while leaving the torrent available for seeding.

---

# 9. Connect Plex to the final folders

Your Plex Movie library should point at:

```text
/volume1/data/media/movies
```

Your TV library should point at:

```text
/volume1/data/media/tv
```

Do not point Plex at:

```text
/volume1/data/torrents
```

Plex should only see the organized media folders.

---

# 10. Enable automatic Plex scanning

Open Plex Web.

Go to:

**Settings → Server → Library**

Enable:

```text
✓ Scan my library automatically
✓ Run a partial scan when changes are detected
```

Plex can then detect filesystem changes and scan only the relevant portion of the library.

If automatic filesystem detection does not work reliably on your Synology configuration, also enable:

```text
Scan my library periodically
```

Plex documents periodic scanning as the fallback when filesystem-change notifications aren't available.

---

# 11. Install Seerr

Run Seerr on the Synology.

The default interface is:

```text
http://SYNOLOGY-IP:5055
```

Seerr officially recommends Docker/container installation.

On first launch:

1. Sign in using **Plex**
2. Select your Plex server
3. Connect your Plex libraries
4. Run the initial library synchronization

For a standard local Plex installation, the Plex connection will usually look similar to:

```text
Host: SYNOLOGY-IP
Port: 32400
```

Seerr supports connecting directly to the local Plex IP and Plex's default port is `32400`.

---

# 12. Connect Radarr to Seerr

In Seerr:

**Settings → Services → Radarr**

Add your Radarr server.

Typically:

```text
Hostname: SYNOLOGY-IP
Port:     7878
API key:  Radarr API key
```

You can find the API key in Radarr under:

**Settings → General → Security**

Select:

- your movie quality profile
- `/data/media/movies`
- Default Server

Seerr requires at least one Radarr/Sonarr server to be marked as the default before requests can be sent.

---

# 13. Connect Sonarr to Seerr

In:

**Seerr → Settings → Services → Sonarr**

Configure:

```text
Hostname: SYNOLOGY-IP
Port:     8989
API key:  Sonarr API key
```

Select:

```text
Root Folder: /data/media/tv
Quality Profile: your preferred profile
Default Server: enabled
```

---

# 14. Enable automatic approval

For the seamless Apple TV experience, your own requests should not require manual approval.

In Seerr:

**Users → your Plex user → Permissions**

Enable the appropriate request and automatic approval permissions.

The target flow is:

```text
Plex Watchlist
→ request created
→ immediately approved
→ Radarr/Sonarr
```

instead of:

```text
Plex Watchlist
→ pending request
→ manually open Seerr
→ approve
```

---

# 15. Enable Plex Watchlist Auto Request

This is the important Apple TV feature.

Seerr can monitor a Plex user's Watchlist and automatically create media requests when something new is added.

## Requirement

You must have logged into Seerr at least once using the **same Plex account** used on your Apple TV. Local-only Seerr accounts cannot use Plex Watchlist Auto Request.

## Administrator permissions

In:

**Seerr → Users → Your User → Permissions**

Enable:

```text
Auto-Request
Auto-Request Movies
Auto-Request Series
```

Also grant the appropriate auto-approval permissions if you want completely automatic downloads.

## User setting

Then open your own Seerr user profile:

**Profile → General**

Enable:

```text
Auto-Request Movies
Auto-Request Series
```

Both the administrator permission and the user toggle have to be enabled.

---

# 16. The finished Apple TV experience

Once everything is configured, you only need Plex.

## Download a movie

On Apple TV:

```text
Open Plex
```

Search for a movie:

```text
Dune: Part Two
```

Even if the movie isn't currently on your server, Plex can show discoverable media.

Select:

```text
Add to Watchlist
```

Then automatically:

```text
Plex Watchlist
      ↓
Seerr notices new item
      ↓
Movie request
      ↓
Radarr
      ↓
Search
      ↓
qBittorrent
      ↓
Download
      ↓
Radarr imports + renames
      ↓
/media/movies
      ↓
Plex scans
      ↓
Available in Plex
```

No Synology interface required.

No qBittorrent interface required.

No Radarr interface required.

No phone required.

---

# 17. TV shows

Exactly the same concept applies:

```text
Apple TV
→ Plex
→ Show
→ Add to Watchlist
→ Seerr
→ Sonarr
→ qBittorrent
→ /media/tv
→ Plex
```

Seerr supports separate automatic Watchlist requests for movies and series.

---

# 18. One limitation: 4K requests

Seerr's current Plex Watchlist Auto Request system only automatically requests the standard-quality configuration.

Dedicated 4K requests still need to be made manually through Seerr.

If you simply use one Radarr/Sonarr instance with a quality profile that accepts 4K when appropriate, this may not matter.

If you maintain separate:

```text
1080p library
4K library
```

Seerr supports multiple Radarr/Sonarr instances for that configuration.

---

# 19. Optional: dedicated request app on Apple TV

A separate Apple TV request interface is not necessary with Plex Watchlist automation.

If you still want one, **SeeTV/seerrTV** exists as a native tvOS frontend for Seerr. It supports browsing, search and requesting movies/shows directly from Apple TV.

As of October 2026, its developer distributes the current beta through TestFlight after encountering App Store approval issues.

The official Seerr team is also developing its own TV-capable application, including Apple TV support, but currently states that App Store publishing is still in progress.

For this setup, **Plex Watchlist is cleaner** because you stay inside Plex.

---

# 20. Recommended final configuration

```text
APPLE TV
└── Plex
      │
      └── Watchlist
             │
             ▼

SYNOLOGY
│
├── Seerr :5055
│      │
│      ├── Movies ──────► Radarr :7878
│      │                     │
│      │                     ▼
│      │                 qBittorrent
│      │                     │
│      │                     ▼
│      │              /data/torrents/movies
│      │                     │
│      │                     ▼
│      │              /data/media/movies
│      │
│      └── TV ─────────► Sonarr :8989
│                            │
│                            ▼
│                       qBittorrent
│                            │
│                            ▼
│                     /data/torrents/tv
│                            │
│                            ▼
│                     /data/media/tv
│
└── Plex :32400
       │
       ├── /data/media/movies
       └── /data/media/tv
```

## Normal daily usage

Your entire workflow becomes:

### Want something

```text
Apple TV → Plex → Add to Watchlist
```

### Wait for it to become available

Plex displays it once Radarr/Sonarr have finished importing it.

### Watch it

```text
Apple TV → Plex → Play
```

qBittorrent effectively disappears from the user experience.