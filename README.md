# Usenet Media Stack

A self-hosted Usenet automation + streaming stack, defined as a single
`docker-compose.yml`. Request a movie or show in a web UI, and it gets
downloaded from Usenet, sorted into a library, and made available to stream —
all automatically.

Built and run on [Bazzite](https://bazzite.gg/) (an immutable, atomic Fedora
spin), but it runs on any Linux box with Docker. Atomic-OS-specific notes are
called out where they matter.

## What's in the box

| Service   | Port | Role                                                    |
|-----------|------|---------------------------------------------------------|
| Seerr     | 5055 | Request UI — the day-to-day driver. Ask for content here. |
| Jellyfin  | 8096 | Streaming server (the thing you actually watch in).     |
| Homarr    | 7575 | Dashboard linking everything together.                  |
| SABnzbd   | 8080 | Usenet download client.                                 |
| Prowlarr  | 9696 | Indexer manager — feeds search results to Sonarr/Radarr. |
| Sonarr    | 8989 | TV automation (search, download, rename, import).       |
| Radarr    | 7878 | Movie automation.                                       |
| Bazarr    | 6767 | Subtitle automation for Sonarr/Radarr.                  |
| dashdot   | 3001 | Host resource dashboard (CPU/RAM/disk).                 |

### How it flows

```
You ──request──▶ Seerr ──▶ Sonarr / Radarr ──search──▶ Prowlarr ──▶ indexers
                                  │
                                  ▼
                               SABnzbd  ──downloads──▶  /data/usenet/complete
                                  │
                          (hardlink + rename)
                                  ▼
                       /data/media/{tv,movies}  ──▶  Jellyfin  ──▶  your TV
```

## Prerequisites

You supply your own accounts — these are not included and not free:

1. **A Usenet provider** (the bandwidth/retention). E.g. Frugal Usenet,
   Newshosting, Eweka. You need server host, port (usually 563 SSL),
   username, password, and connection count.
2. **At least one Usenet indexer** (the search engine). E.g. NZBGeek,
   NZBPlanet, DrunkenSlug. You get an API key + URL.
3. **Docker + the Compose plugin.**
   - **Ubuntu / Debian:** follow Docker's official guide to install Docker
     Engine and the Compose plugin
     (<https://docs.docker.com/engine/install/ubuntu/>). In short: add Docker's
     apt repo, then
     `sudo apt install docker-ce docker-ce-cli containerd.io docker-compose-plugin`.
     Add yourself to the `docker` group (`sudo usermod -aG docker $USER`) and
     log out/in so you can run `docker` without `sudo`.
   - **Bazzite / atomic Fedora:** `rpm-ostree install moby-engine docker-compose`
     then reboot. (The `docker-compose-plugin` package is *not* available
     there.)

> The Jellyfin service includes a `deploy:` block for GPU hardware
> transcoding. If you don't have a supported GPU set up, delete that block to
> run Jellyfin CPU-only.

## Setup

```bash
git clone <this-repo-url> usenet
cd usenet

# 1. Configure environment
cp .env.example .env
$EDITOR .env                      # set PUID/PGID, TZ, paths, HOMARR_SECRET
openssl rand -hex 32              # paste the result as HOMARR_SECRET

# 2. Create the media tree on your big disk (must match MEDIA_ROOT in .env).
#    Replace the path below with your own — e.g. /mnt/storage on Ubuntu,
#    or /var/mnt/storage on Bazzite.
MEDIA_ROOT=/path/to/your/media/disk
mkdir -p "$MEDIA_ROOT"/data/usenet/{complete,incomplete}
mkdir -p "$MEDIA_ROOT"/data/media/{tv,movies}

# 3. Bring it up
docker compose up -d
docker compose ps
```

Then open each service and wire it together (see below).

> **SELinux note (Bazzite/Fedora):** the volume mounts in the compose file use
> `:z` / `:Z` so SELinux relabels them. On distros without SELinux these
> suffixes are harmless and can stay.

### First-run wiring

Roughly in this order:

1. **SABnzbd** (`:8080`) — add your Usenet provider server(s). Create
   categories `tv` and `movies`. If other containers get a 403, add their
   hostnames (or `*`) to `Config → Special → host_whitelist`.
2. **Prowlarr** (`:9696`) — add your indexer(s). Under
   `Settings → Apps`, add Sonarr (`http://sonarr:8989`) and Radarr
   (`http://radarr:7878`) so indexers sync to them automatically.
3. **Sonarr** (`:8989`) / **Radarr** (`:7878`) — add SABnzbd as the download
   client (`http://sabnzbd:8080`). Set the root folders to `/data/media/tv`
   and `/data/media/movies`. Pick quality profiles.
4. **Jellyfin** (`:8096`) — add libraries pointing at `/data/media/tv` and
   `/data/media/movies`. Optionally enable hardware transcoding under
   `Dashboard → Playback`.
5. **Seerr** (`:5055`) — connect it to Jellyfin, Sonarr, and Radarr. This
   becomes your everyday "I want to watch X" button.
6. **Homarr** (`:7575`) — optional dashboard; add tiles for the above.

> Use container hostnames (`http://sonarr:8989`), **not** `localhost`, for
> service-to-service connections — they share a Docker network.

## How data is laid out

Two distinct locations, on purpose:

- **`DATA_ROOT`** (the repo dir, on a fast SSD) holds `config/` — each
  service's database and settings. Small, read/write-heavy, worth keeping
  fast. **Not committed to git** (see below).
- **`MEDIA_ROOT`** (a big disk) holds `data/` — downloads *and* the final
  library, together:

```
${MEDIA_ROOT}/data/
├── usenet/{complete,incomplete}/   # SABnzbd writes here
└── media/{tv,movies}/              # final library; Jellyfin reads here
```

**Why downloads and library live under one tree:** Sonarr/Radarr *hardlink*
from `complete/` into `media/` instead of copying. That's instant and uses no
extra space — but hardlinks only work within a single filesystem. Keep both
under `MEDIA_ROOT` (same disk) and you get atomic, space-free imports. Split
them across disks and every import silently doubles your storage.

## What this repo tracks (and what it doesn't)

Committed: `docker-compose.yml`, `.env.example`, and this README.

**Ignored** (`.gitignore`): `config/` and `.env`. The `config/` tree is
gigabytes of databases with your **API keys and provider passwords baked in** —
never push it anywhere. `.env` holds your secrets too. Anyone cloning this sets
up their own services from scratch using the steps above.

## Day-to-day

```bash
docker compose ps                              # status
docker compose logs -f <service>               # tail one service
docker compose up -d                           # apply compose edits
docker compose up -d --force-recreate <svc>    # recreate one container
docker compose pull && docker compose up -d    # update all images
```

## Gotchas worth knowing

- **Always request through Seerr or search inside Sonarr/Radarr** — not through
  Prowlarr's manual search. Grabbing straight from Prowlarr bypasses the
  *arrs, so nothing tracks or auto-imports the download. Prowlarr search is for
  testing indexers only.
- **Transcoding load:** an older GPU may not hardware-*decode* newer codecs
  (HEVC, AV1, Dolby Vision), which forces CPU transcoding and can stutter on 4K
  HDR. If you hit that, cap quality profiles around 1080p and prefer H.264
  releases. Check your GPU's decode support before relying on it.
- **Seerr runs as a fixed UID 1000** and ignores PUID/PGID. If its config files
  end up root-owned, `sudo chown -R 1000:1000 config/jellyseerr` fixes it.
