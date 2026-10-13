# AI-Assisted Synology Media Server Setup

## Goal

Give an AI agent controlled access to your Synology so it can install and configure:

```text
Seerr
Radarr
Sonarr
Prowlarr
qBittorrent integration
Docker networking
Media/download folders
Permissions
Plex integration
```

The architecture:

```text
Mac
│
├── Codex / AI agent
│       │
│       └── SSH
│             │
│             ▼
│        Synology NAS
│
└── Tailscale
        │
        └──────── Synology Tailscale
```

Then the AI can execute things such as:

```text
ssh synology

docker compose pull
docker compose up -d

mkdir ...
chmod ...
docker inspect ...
docker logs ...
curl ...
```

The AI should not receive unrestricted permanent administrator access.

---

# 1. Recommended access model

Use:

**Codex on Mac → SSH → Synology**

Do not expose Synology SSH directly to the internet.

If you're only doing this at home on the same network, local LAN access is enough.

For remote access, use **Tailscale**.

Tailscale provides encrypted private access to Synology without opening an SSH port on your router. Synology is officially supported through Package Center.

---

# 2. Install Tailscale

On Synology:

```text
DSM
→ Package Center
→ Search "Tailscale"
→ Install
```

Sign in.

Install Tailscale on the Mac as well and sign into the same account.

Your Synology will then receive something similar to:

```text
synology.tail1234.ts.net
```

or a private address:

```text
100.x.x.x
```

You can SSH through this address without opening router ports.

This step is optional if the Mac and NAS are always on the same LAN.

---

# 3. Install Container Manager

On Synology:

```text
Package Center
→ Container Manager
→ Install
```

This is one of the few things worth installing manually before giving control to the AI.

The AI can then manage containers through Docker commands rather than clicking around DSM.

---

# 4. Create a temporary AI installer account

Do not give the AI your normal Synology account.

Create something like:

```text
ai-installer
```

In:

```text
DSM
→ Control Panel
→ User & Group
```

Initially give it administrator privileges because Docker management on Synology will likely require elevated permissions.

Use a strong random password.

This account is temporary.

When installation is finished:

```text
disable ai-installer
```

or remove it.

---

# 5. Enable SSH

Synology officially supports SSH through:

```text
Control Panel
→ Terminal & SNMP
→ Terminal
→ Enable SSH service
```

Synology warns that SSH provides powerful system access, so it should only be enabled when necessary.

You can use a custom SSH port, for example:

```text
2222
```

Then from your Mac:

```bash
ssh -p 2222 ai-installer@SYNOLOGY-IP
```

With Tailscale:

```bash
ssh -p 2222 ai-installer@synology.tail1234.ts.net
```

Verify this works before involving the AI.

---

# 6. Use an SSH key instead of giving the AI your password

On the Mac:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/synology_ai
```

You will get:

```text
~/.ssh/synology_ai
~/.ssh/synology_ai.pub
```

The first file is private.

Never paste it into ChatGPT or another AI conversation.

Add the `.pub` key to:

```text
/home/ai-installer/.ssh/authorized_keys
```

Then test:

```bash
ssh \
  -i ~/.ssh/synology_ai \
  -p 2222 \
  ai-installer@synology.tail1234.ts.net
```

The important design is:

```text
AI
↓
local SSH command
↓
Mac reads private key
↓
Synology
```

The actual private key stays on your Mac.

---

# 7. Add an SSH shortcut

On your Mac create/edit:

```text
~/.ssh/config
```

Add:

```text
Host synology
    HostName synology.tail1234.ts.net
    User ai-installer
    Port 2222
    IdentityFile ~/.ssh/synology_ai
```

Now:

```bash
ssh synology
```

should connect.

This makes the AI's job much easier.

It doesn't need to know:

```text
IP
port
password
private key
```

It only needs permission to run:

```bash
ssh synology
```

---

# 8. Install Codex on the Mac

Use Codex locally because it can operate a terminal and execute commands.

Current Codex builds can be installed with:

```bash
npm install -g @openai/codex@latest
```

OpenAI's current documentation also supports updating Codex this way.

Then:

```bash
codex
```

Sign in.

---

# 9. Keep approval prompts enabled

Do not immediately give the agent unrestricted execution.

Use:

```text
approval_policy = on-request
```

OpenAI documents `on-request` as the mode where Codex performs routine work in its sandbox but requests permission when it needs to operate outside those boundaries.

For this project, that means you want to see things like:

```text
Codex wants to run:

ssh synology "sudo docker compose up -d"

Approve?
```

Then approve it.

Do not initially use:

```text
danger-full-access
+
approval_policy = never
```

OpenAI describes that combination as effectively full access.

For a NAS containing personal data, that is unnecessary.

---

# 10. Give the AI a dedicated workspace

On your Mac:

```bash
mkdir -p ~/Projects/synology-media-stack
cd ~/Projects/synology-media-stack
```

Start Codex from here:

```bash
codex
```

The AI can create things such as:

```text
synology-media-stack/
├── compose.yaml
├── .env
├── .env.example
├── README.md
├── backup/
└── setup-notes.md
```

Keeping the configuration on your Mac means you retain a readable copy of everything the AI deployed.

---

# 11. First task: audit the NAS

Do **not** start by telling the AI to install everything.

First tell it to inspect the machine.

Use this prompt:

```text
Connect to my Synology NAS using:

ssh synology

Perform a READ-ONLY audit.

Do not modify anything yet.

Determine:

- Synology DSM version
- CPU architecture
- available RAM
- storage volumes
- free storage
- Docker/Container Manager availability
- Docker version
- Docker Compose version
- existing containers
- existing Docker networks
- existing Docker volumes
- current Plex installation
- current qBittorrent installation
- qBittorrent container mounts
- qBittorrent networking
- existing media folders
- existing download folders
- filesystem ownership and permissions relevant to media
- UID/GID being used by Plex and qBittorrent
- whether /volume1/data already exists
- whether hardlink-compatible media/download paths are possible

Do not:

- stop containers
- restart services
- change permissions
- modify files
- install packages
- remove anything
- change DSM
- change firewall rules
- expose ports

Create AUDIT.md locally with your findings and a recommended installation plan.
```

This is important because your existing qBittorrent and Plex configuration should be preserved.

---

# 12. Review the audit

You want the AI to discover things such as:

```text
Plex:
Native DSM package

qBittorrent:
Docker container

Existing downloads:
/volume1/downloads

Existing Plex movies:
/volume1/media/movies

Existing TV:
/volume1/media/tv
```

The AI can then adapt the architecture instead of blindly creating:

```text
/volume1/data
```

and breaking your existing setup.

---

# 13. Take a backup before modifying anything

Have the AI back up relevant Docker configs before touching them.

Example:

```text
/volume1/docker/qbittorrent
```

should be copied/snapshotted before changing its configuration.

The AI should never:

```text
rm -rf
```

existing media/config folders.

It should never re-create the existing qBittorrent container until it understands its current configuration.

---

# 14. Installation task

After reading `AUDIT.md`, give Codex this main task.

## AI installation prompt

```text
You are configuring my personal Synology media server.

You have permission to modify this Synology through:

ssh synology

Goal:

Build a clean media automation stack around my EXISTING Plex and qBittorrent installation.

Install and configure:

- Seerr
- Radarr
- Sonarr
- Prowlarr

Integrate them with:

- my existing qBittorrent instance
- my existing Plex server

End-user experience:

Apple TV
→ Plex
→ Add movie/show to Plex Watchlist
→ Seerr detects it
→ Radarr/Sonarr handles it
→ qBittorrent downloads it
→ Radarr/Sonarr imports it
→ Plex automatically sees it
→ content becomes playable in Plex

IMPORTANT SAFETY RULES

Do not:

- delete existing files
- delete existing containers
- remove existing media
- overwrite Plex databases
- overwrite qBittorrent configuration
- expose services directly to the public internet
- modify my router
- disable Synology security
- disable the Synology firewall
- modify unrelated DSM settings
- install anything directly onto DSM if a Docker container is appropriate
- use privileged Docker containers unless absolutely necessary
- run destructive commands without asking first

Before modifying an existing configuration file:

1. create a timestamped backup
2. show what you intend to change
3. preserve existing settings

Existing Plex and qBittorrent must keep working throughout the migration whenever possible.

Use Docker Compose.

Store application configuration underneath:

/volume1/docker/

Prefer a structure similar to:

/volume1/docker/media-stack/
    compose.yaml

/volume1/docker/radarr/
/volume1/docker/sonarr/
/volume1/docker/prowlarr/
/volume1/docker/seerr/

Adapt this if AUDIT.md shows that another existing structure is more appropriate.

PATH ARCHITECTURE

Radarr, Sonarr and qBittorrent should see downloads/media through a consistent container path.

Prefer:

/data

For example:

Host:

/volume1/data/torrents
/volume1/data/media/movies
/volume1/data/media/tv

Container:

/data/torrents
/data/media/movies
/data/media/tv

However:

Do NOT migrate my existing media automatically just to achieve this structure.

Inspect my existing paths first.

If moving existing data would be necessary, stop and ask for approval.

Hardlinks should work where the Synology filesystem and current folder structure permit them.

Use appropriate PUID/PGID values based on the existing media ownership rather than guessing.

TIMEZONE

Use:

Europe/Amsterdam

PORTS

Prefer:

Seerr     5055
Radarr    7878
Sonarr    8989
Prowlarr  9696

Do not change the existing qBittorrent or Plex ports unnecessarily.

SECURITY

Use:

restart: unless-stopped

Where supported:

security_opt:
  - no-new-privileges:true

Avoid privileged mode.

Do not expose management interfaces beyond my LAN/Tailscale unless explicitly approved.

SEERR

Use the official Seerr image:

ghcr.io/seerr-team/seerr:latest

Follow current Seerr documentation.

RADARR / SONARR / PROWLARR

Use well-maintained Docker images recommended by their respective projects.

Use one consistent /data volume strategy.

DEPLOYMENT PROCESS

1. Inspect current configuration.
2. Produce the proposed compose.yaml locally.
3. Validate it.
4. Explain any effect on existing services.
5. Deploy new containers.
6. Verify every container is healthy.
7. Verify ports respond.
8. Verify Docker networking.
9. Verify filesystem permissions.
10. Configure Radarr/Sonarr download-client connectivity to qBittorrent where possible.
11. Configure Prowlarr integration where possible.
12. Configure Seerr service connectivity where possible.
13. Do not attempt to bypass Plex authentication.
14. Stop when Plex authentication or user credentials require my interaction.
15. After I authenticate, continue configuration.
16. Test movie workflow.
17. Test TV workflow.
18. Verify Plex library detection.
19. Produce FINAL-CONFIGURATION.md documenting everything.

After installation, provide:

- container list
- ports
- volumes
- Docker networks
- media paths
- download paths
- backup locations
- service URLs
- remaining manual steps
- rollback instructions
- update instructions

Do not mark the project complete until:

docker compose ps

shows the intended services operational and basic connectivity between services has been verified.
```

---

# 15. What the AI should create

A resulting Compose architecture may look roughly like:

```yaml
services:

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=...
      - PGID=...
      - TZ=Europe/Amsterdam
    volumes:
      - /volume1/docker/radarr:/config
      - /volume1/data:/data
    ports:
      - 7878:7878
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=...
      - PGID=...
      - TZ=Europe/Amsterdam
    volumes:
      - /volume1/docker/sonarr:/config
      - /volume1/data:/data
    ports:
      - 8989:8989
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=...
      - PGID=...
      - TZ=Europe/Amsterdam
    volumes:
      - /volume1/docker/prowlarr:/config
    ports:
      - 9696:9696
    restart: unless-stopped

  seerr:
    image: ghcr.io/seerr-team/seerr:latest
    container_name: seerr
    environment:
      - TZ=Europe/Amsterdam
    volumes:
      - /volume1/docker/seerr:/app/config
    ports:
      - 5055:5055
    restart: unless-stopped
```

That is an **example only**.

The AI should generate the real configuration after inspecting your NAS.

Seerr recommends Docker and currently publishes its official container as `ghcr.io/seerr-team/seerr`.

Radarr and Sonarr specifically recommend consistent container paths such as a common `/data` mount because mismatched `/downloads`, `/movies`, etc. can interfere with atomic moves and hardlinks.

---

# 16. Let the AI perform verification

After deployment, tell it:

```text
Perform a post-install verification.

Check:

docker compose ps
docker logs for all new services
disk permissions
container mounts
container networks
HTTP health of ports:
5055
7878
8989
9696

Test connectivity:

Seerr → Radarr
Seerr → Sonarr
Radarr → qBittorrent
Sonarr → qBittorrent
Prowlarr → Radarr
Prowlarr → Sonarr

Do not download any media yet.

Report every pass/failure and fix configuration-level issues where safe.
```

---

# 17. Authentication should remain manual

There are several credentials the AI should **not** need to see.

### Plex

When Seerr asks you to authenticate with Plex:

You do that yourself in the browser.

Do not give the AI:

```text
Plex password
Plex account password
```

The AI can continue once authentication has generated the required internal tokens.

### Private indexers

Add account credentials yourself where required.

### qBittorrent

If the AI needs qBittorrent API access, it may need the qBittorrent Web UI credentials.

Prefer temporarily changing these credentials or configuring them yourself.

Do not place permanent passwords directly inside the AI prompt.

OpenAI's security guidance recommends keeping long-lived credentials outside agent-controlled environments wherever possible.

---

# 18. Configure Plex Watchlist

Once Seerr is authenticated with Plex, tell the AI:

```text
Inspect the current Seerr configuration.

Configure everything technically possible for Plex Watchlist Auto Request.

Do not modify my Plex account.

Tell me exactly which user-interface settings I must enable manually.
```

You then enable:

```text
Seerr
→ User permissions

Auto-Request
Auto-Request Movies
Auto-Request Series
```

and your own profile's corresponding auto-request settings.

Then:

```text
Apple TV
→ Plex
→ Add to Watchlist
```

becomes the request system.

---

# 19. First real test

Do not start with several downloads.

Use one small test item you are legally entitled to download.

Have the AI monitor:

```text
Seerr request
↓
Radarr/Sonarr
↓
qBittorrent
↓
completed torrent
↓
import
↓
Plex library
```

Tell it:

```text
Monitor the first test request.

Do not make unrelated changes.

Check:

- request creation
- Radarr/Sonarr queue
- qBittorrent category
- download path
- completed download handling
- import path
- filename
- hardlink status if applicable
- Plex detection

If something fails, diagnose the exact failure before changing configuration.
```

---

# 20. Give the AI a troubleshooting command set

After setup, your Mac can become an AI-powered administration console.

Examples:

```text
Connect to the Synology and determine why Radarr cannot import the latest completed download.
```

```text
Check all media-stack containers and identify anything unhealthy. Do not modify anything until you have diagnosed the issue.
```

```text
Update Seerr, Radarr, Sonarr and Prowlarr. Back up configuration first and do not update Plex or qBittorrent.
```

```text
Check how much storage is being used by torrents versus the Plex library and determine whether hardlinks are working correctly.
```

```text
Read the last 200 Sonarr log lines and diagnose this failed import.
```

The agent handles:

```text
ssh
docker
logs
filesystem
configuration
curl/API testing
```

rather than you manually debugging the NAS.

---

# 21. After installation: reduce access

Once everything works:

### Disable the temporary account

```text
DSM
→ User & Group
→ ai-installer
→ Disable
```

Or remove it.

### Optionally disable SSH

```text
DSM
→ Terminal & SNMP
→ Disable SSH
```

Then AI access is gone entirely.

When you later want the AI to perform maintenance:

```text
enable SSH
enable ai-installer
perform maintenance
disable again
```

This gives you a hard kill switch.

---

# 22. Recommended permanent setup

The safest long-term design:

```text
                    INTERNET
                       │
                       X
                 no open SSH port
                       │

Mac ──────── Tailscale ──────── Synology
 │                                  │
 │                                  ├── Plex
 │                                  ├── qBittorrent
 │                                  ├── Radarr
Codex                                ├── Sonarr
 │                                  ├── Prowlarr
 └──────── SSH ────────────────────► └── Seerr
```

Keep:

```text
Codex approval prompts: ON
SSH key: Mac only
NAS access: Tailscale/LAN only
AI account: separate
Backups: before modification
Plex authentication: human
Secrets: human-managed
```

This gives the AI enough control to effectively build and maintain the entire media stack without turning the Synology into an unrestricted AI-controlled machine.