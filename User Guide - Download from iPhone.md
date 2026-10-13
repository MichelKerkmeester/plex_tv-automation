Yes. For a **simple iPhone-only workflow**, you do not need Radarr/Sonarr at all.

Use:

```text
iPhone
├── Safari → qBittorrent Web UI
└── DS file → move completed download
                    ↓
              Plex media folder
```

### 1. Control qBittorrent from your iPhone

Enable qBittorrent's Web UI on the Synology. Then on the iPhone open:

```text
http://YOUR-SYNOLOGY-IP:8080
```

or whatever port your qBittorrent Web UI uses.

In Safari you can use **Share → Add to Home Screen**, which makes qBittorrent behave almost like an iPhone app.

From there:

```text
Find magnet/torrent on iPhone
↓
Copy magnet link / download .torrent
↓
Open qBittorrent
↓
Add Torrent
↓
Download starts on Synology
```

qBittorrent's Web UI supports choosing the save location when adding a torrent. [GitHub](https://github.com/qbittorrent/qBittorrent/wiki/Explanation-of-Options-in-qBittorrent/bd97fa8b714e8cdeb6f3e09ae8c6bb7c1d382210?utm_source=chatgpt.com)

For access outside your house, use something like **Tailscale** rather than opening qBittorrent's port directly to the internet.

---

## 2. Have one download folder

For example:

```text
/volume1/downloads/
```

qBittorrent downloads everything there.

Then your Plex libraries might be:

```text
/volume1/media/movies/
/volume1/media/tv/
```

---

## 3. Install DS file on your iPhone

Synology's **DS file** app lets you browse and manage files directly on the NAS, including moving/renaming files without first downloading them to the iPhone. [Synology Knowledge Center](https://kb.synology.com/en-us/SRM/help/DSfile/iPhone?utm_source=chatgpt.com)

So after the torrent finishes:

```text
DS file

Downloads
└── Movie.Name.2026.1080p.mkv
```

Select it and move it to:

```text
Media
└── Movies
    └── Movie Name (2026)
        └── Movie Name (2026).mkv
```

For TV:

```text
Media
└── TV
    └── Show Name
        └── Season 01
            └── Show Name - S01E01.mkv
```

Then Plex scans the media folder and picks it up.

---

# Even easier: use qBittorrent categories

I would configure three categories:

```text
Downloads
Movies
TV
```

with paths like:

```text
Downloads → /downloads
Movies    → /media/movies
TV        → /media/tv
```

qBittorrent supports giving each category its own save path. With **Automatic Torrent Management**, changing a torrent's category can also relocate its data to the category's configured folder. [GitHub](https://github.com/qbittorrent/qBittorrent/discussions/24035?utm_source=chatgpt.com)

That gives you this workflow:

```text
iPhone → qBittorrent
         ↓
      Download
         ↓
torrent finishes
         ↓
Change Category → Movies
         ↓
qBittorrent moves it
         ↓
Plex sees it
```

So you wouldn't even need DS file for most downloads.

### I would still use this structure

```text
/volume1/data/
├── downloads/
│
└── media/
    ├── movies/
    └── tv/
```

and expose them inside the qBittorrent container as something like:

```text
/data/downloads
/data/media/movies
/data/media/tv
```

Then configure:

```text
qBittorrent

Default:
    /data/downloads

Category Movies:
    /data/media/movies

Category TV:
    /data/media/tv
```

### Best minimal UX

On your iPhone:

```text
1. Find torrent
2. Copy magnet
3. Open qBittorrent Home Screen shortcut
4. Paste
5. Wait
6. Change category → Movies or TV
7. Open Plex
```

The only downside versus Radarr/Sonarr is that **qBittorrent won't intelligently rename/organize the files for Plex**. For clean Plex metadata, you may occasionally still need DS file to rename a movie/show folder correctly.