[中文](README.md) | English

# NSFW Image Detection Service

[![Docker Image](https://github.com/QCEnjoyLL/nsfw-detector/actions/workflows/docker-publish.yml/badge.svg)](https://github.com/QCEnjoyLL/nsfw-detector/actions/workflows/docker-publish.yml)
[![GHCR](https://img.shields.io/badge/ghcr.io-nsfw--detector-2496ED?logo=docker&logoColor=white)](https://github.com/QCEnjoyLL/nsfw-detector/pkgs/container/nsfw-detector)

An HTTP service for NSFW (Not Safe For Work) image detection built on a ViT model. CPU-only inference, suitable for a modest VPS or cloud instance, providing image moderation for forums, image hosts, chat platforms and content sites.

The model is [Marqo/nsfw-image-detection-384](https://huggingface.co/Marqo/nsfw-image-detection-384) (ViT-tiny, 5.6M parameters). Quantized to int8 it weighs only 6.4MB and runs one image in about 18ms on 16 threads.

Two ways to call it: pass an image URL, or upload a file. It returns a `0-1` risk score plus a boolean verdict, so callers can decide what to do next.

---

## Contents

- [Features](#features)
- [Getting started](#getting-started)
- [Automated builds and image releases](#automated-builds-and-image-releases)
- [Configuration](#configuration)
- [Token management](#token-management)
- [API reference](#api-reference)
- [Result cache](#result-cache)
- [Data and backups](#data-and-backups)
- [Performance](#performance)
- [Project layout](#project-layout)
- [FAQ](#faq)
- [License and credits](#license-and-credits)

---

## Features

**Detection**

- CPU-only inference, no GPU required; about 135MB resident memory per worker, model only 6.4MB
- Supports jpg / png / bmp / webp / gif (first frame for animated images)
- Optional QR-code detection (`qr_detect=enable`)
- Returns raw probabilities; the decision threshold is configurable

**Engineering**

- **URL checks accept POST only** — the URL goes in the request body, avoiding escaping issues and keeping URLs out of access logs
- **Multi-token management** with expiry, daily quota and total quota, managed from a separate admin port
- **Result cache** in SQLite, shared across workers, survives restarts, sub-millisecond hits
- **Fully local build** — the image is built from this repository's Dockerfile, no third-party prebuilt images
- Built-in overload protection: inference semaphore plus a connection limit, failing fast instead of melting the box
- Optional web UI on the index page (`WEBUI=on`) for manual testing

---

## Getting started

### Requirements

- Docker 20.10+ (or Docker Compose v2)
- At least 1GB RAM and 1 CPU core; 2+ cores recommended
- About 300MB disk (image, model and runtime data)

### Option 1: Docker Compose (recommended)

```bash
git clone https://github.com/QCEnjoyLL/nsfw-detector.git nsfw
cd nsfw

# 1. Set your admin secret (do not keep the default)
sed -i 's/your_admin_token_here/your-secret-here/' docker-compose.yaml

# 2. Build and start (the image is built on first run)
docker compose up -d --build
```

Once running:

- API: `http://your-server:6086`
- Admin page: `http://127.0.0.1:6087` (**on the machine running the container**; log in with `ADMIN_TOKEN`). Compose binds it to host `127.0.0.1`; see [Opening the admin page](#opening-the-admin-page) for remote access

### Option 2: Build the image manually

```bash
cd nsfw

# Build the local image (tagged nsfw:local by default)
./build.sh

# Run it
docker run -d \
  --name nsfw \
  -p 6086:6086 \
  -p 127.0.0.1:6087:6087 \
  -e ADMIN_TOKEN=your-secret-here \
  -v "$(pwd)/data:/opt/nsfw/data" \
  --restart always \
  nsfw:local
```

`build.sh` options:

```bash
./build.sh --no-cache                    # rebuild without cache
IMAGE=myrepo/nsfw:1.0 ./build.sh         # custom image name
```

### Option 3: Pull the prebuilt image (GHCR)

The repository ships a GitHub Actions workflow —
[`.github/workflows/docker-publish.yml`](.github/workflows/docker-publish.yml) —
that builds the image, smoke-tests it and pushes it to the GitHub Container Registry
on every push to `main` and every `v*` tag.

```bash
docker pull ghcr.io/qcenjoyll/nsfw-detector:latest

docker run -d \
  --name nsfw \
  -p 6086:6086 \
  -p 127.0.0.1:6087:6087 \
  -e ADMIN_TOKEN=your-secret-here \
  -v "$(pwd)/data:/opt/nsfw/data" \
  --restart always \
  ghcr.io/qcenjoyll/nsfw-detector:latest
```

> ⚠️ **The package is Private by default.** After the first push, open **Packages →
> nsfw-detector → Package settings → Danger Zone → Change visibility** and switch it to
> **Public**, otherwise nobody can pull it anonymously. Until then everyone has to log in:
>
> ```bash
> echo "$GITHUB_TOKEN" | docker login ghcr.io -u qcenjoyll --password-stdin
> ```
>
> The token needs `read:packages`. Without a login, pulls fail with `unauthorized` / `denied`.

**A note on casing**: the GitHub account is `QCEnjoyLL`, but Docker / GHCR image names
**must be lowercase**, so the image is always `ghcr.io/qcenjoyll/nsfw-detector`.
Writing `ghcr.io/QCEnjoyLL/...` is rejected by the registry
(`invalid reference format`). GitHub pages, `git clone` and `docker login -u` are
case-insensitive, so keep `QCEnjoyLL` there.

Common tags (see [the next section](#automated-builds-and-image-releases)):

```bash
ghcr.io/qcenjoyll/nsfw-detector:latest      # default branch / newest release
ghcr.io/qcenjoyll/nsfw-detector:1.2.3       # semver tag (requires pushing a v1.2.3 tag)
ghcr.io/qcenjoyll/nsfw-detector:sha-9bf94ea # immutable snapshot of a commit, for rollbacks
```

### Option 4: Without Docker

Requires Python 3.10+ (3.13 recommended).

```bash
cd nsfw
python3 -m venv myenv
. myenv/bin/activate
pip install -r app/requirements.txt

ADMIN_TOKEN=your-secret-here ./run.sh
```

`run.sh` detects the CPU core count and derives the worker and thread counts. Use `./run.sh dev` for hot reload during development.

### First run

1. Open `http://127.0.0.1:6087` (on the machine running the container) and log in with `ADMIN_TOKEN`
2. Click **New Token**, give it a name (e.g. "production"), set expiry and quotas as needed
3. **Copy the plaintext token** — it is shown only once
4. Call the API with it

```bash
curl -X POST 'http://your-server:6086/api/url_check' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer nsfw_xxxxxxxx' \
  -d '{"url":"https://example.com/test.jpg"}'
```

> ⚠️ **Until the first token is created, the API does not require authentication**
> (this preserves the old behaviour). If the service is exposed to the internet,
> create a token first.

---

## Automated builds and image releases

Workflow: [`.github/workflows/docker-publish.yml`](.github/workflows/docker-publish.yml).
The goal is simple: **every change produces an image that is proven to run, published
somewhere it can be pulled from directly.**

### Triggers

| Trigger | What happens |
|---|---|
| Push to `main` / `master` | build → smoke test → push to GHCR |
| Tag `v*` (e.g. `v1.2.3`) | same, plus semantic-version tags |
| Pull request | build + smoke test only, **never pushed** |
| Manual dispatch (Actions → Docker Image → Run workflow) | choose platforms, optionally skip pushing |

Doc-only changes (`*.md`, `LICENSE`) do not trigger a build, saving Actions minutes.

### The two jobs

1. **Build and smoke test**: builds the `linux/amd64` image and loads it locally, then really
   starts the container → waits for the in-image `HEALTHCHECK` to go green → requests the
   index page and expects 200 → generates an image with the image's own Pillow and POSTs it
   to `/api/upload_check` for a real inference, asserting `code=200` and a `nsfw` score in
   `0..1` → finally calls `/api/cache_stats` to confirm the SQLite cache works.
   So any published image is guaranteed to **boot, load the model and return a score**.
2. **Push to GHCR**: QEMU + Buildx multi-platform build, tags generated by
   `docker/metadata-action`, and the resulting tags plus digest written to the job summary
   (visible straight on the Actions page).

Each job has its own GHA cache scope, and `concurrency` cancels a still-running build when
you push the same branch again.

### Tag scheme

| Trigger | Tags |
|---|---|
| Push to `main` | `main`, `latest`, `sha-<short>` |
| Tag `v1.2.3` | `1.2.3`, `1.2`, `latest`, `sha-<short>` |
| Other branches | branch name, `sha-<short>` |

`sha-*` is an immutable snapshot — use it to pin production and to roll back. `latest`
follows the default branch and the newest release.

### Cutting a release

```bash
git tag v1.2.3          # semver, must start with v
git push origin v1.2.3  # triggers the workflow, produces 1.2.3 / 1.2 / latest
```

### Repository variables (optional)

Under **Settings → Secrets and variables → Actions → Variables**:

| Variable | Default | Description |
|---|---|---|
| `PLATFORMS` | `linux/amd64` | target platforms, comma separated |

> **arm64 does not work yet**: `zxing-cpp==2.3.0` publishes manylinux wheels for x86_64 /
> i686 only, so on arm64 pip falls back to a source build and the image has no cmake / g++,
> which fails (confirmed by an actual run). Supporting arm64 needs a builder stage in the
> Dockerfile that compiles zxing-cpp into a wheel for the runtime stage.

No secrets need to be configured: pushes use the automatically injected `GITHUB_TOKEN`,
scoped down in the workflow to `contents: read` + `packages: write`.

### Reproducing the checks locally

```bash
docker build -t nsfw:ci .
docker run -d --name nsfw-ci -p 6086:6086 -e ADMIN_TOKEN=test nsfw:ci
curl -fsS http://127.0.0.1:6086/
docker exec nsfw-ci python -c "from PIL import Image; Image.new('RGB',(96,96)).save('/tmp/a.jpg')"
docker cp nsfw-ci:/tmp/a.jpg a.jpg
curl -X POST http://127.0.0.1:6086/api/upload_check -F "file=@a.jpg"
docker rm -f nsfw-ci
```

---

## Configuration

Everything is configured through environment variables. For Docker, put them under
`environment` in `docker-compose.yaml`.

### Business and inference

| Variable | Default | Description |
|---|---|---|
| `NSFW_THRESHOLD` | `0.8` | Decision threshold. `is_nsfw=true` when the `nsfw` score is **greater than** this value. Raise it to be more permissive (more false negatives), lower it to be stricter. It only affects the boolean; `nsfw` is always the raw 0-1 probability. This model scores lower overall, so the default moved from 0.9 to 0.8 |
| `WORKERS` | `1` | Worker processes, clamped to `min(4, cores/2)` |
| `ORT_NUM_THREADS` | auto | Inference threads per process, defaults to `cores / WORKERS`. **The biggest factor for latency**; verify on your hardware |
| `INFER_CONCURRENCY` | `=ORT_NUM_THREADS` | Max concurrent inference requests; extras queue |
| `LIMIT_CONCURRENCY` | `WORKERS×(N×2+4)` | Max connections; extras get 503 |
| `ORT_ALLOW_SPINNING` | `0` | Disable ORT thread spinning when idle. **Do not set to 1** — multi-threaded inference gets noticeably slower (see [Performance](#performance)) |
| `APP_HOST` / `APP_PORT` | `0.0.0.0` / `6086` | API listen address and port |
| `WEBUI` | `off` | `on` enables the web UI on the index page |
| `ENABLE_DOCS` | `off` | `on` exposes `/docs` and `/openapi.json` |
| `CORS_ORIGINS` | `*` | Comma-separated allowlist; `off` disables CORS |
| `LOG_LEVEL` | `warning` | `info` enables the access log |

### Image fetching

| Variable | Default | Description |
|---|---|---|
| `MAX_IMAGE_BYTES` | `10485760` | Max image size (10MB) |
| `MAX_IMAGE_PIXELS` | `64000000` | Max decoded pixels, guards against decompression bombs |
| `HTTP_TIMEOUT` | `30` | Total fetch timeout (seconds) |
| `HTTP_CONNECT_TIMEOUT` | `10` | Connect timeout (seconds). Overseas CDNs can be slow to connect; do not set it too low |
| `HTTP_TRUST_ENV` | `1` | Honour `HTTP_PROXY` etc. Set to `0` if the host has a proxy that should not be used for image fetching |

### Token management

| Variable | Default | Description |
|---|---|---|
| `ADMIN_TOKEN` | empty | **Admin page login secret**. The admin page is unusable without it |
| `ADMIN_ENABLED` | `1` | Set to `0` to disable the admin port |
| `ADMIN_HOST` | see note | Admin listen address. Bare-metal default `127.0.0.1` (localhost only); **the container image defaults to `0.0.0.0`**, because Docker port mapping DNATs traffic to the container IP and a loopback-only bind is unreachable from the host (see [Opening the admin page](#opening-the-admin-page)) |
| `ADMIN_PORT` | `6087` | Admin port |
| `TOKENS_DB_PATH` | `data/tokens.db` | Token database path |
| `TOKEN` | empty | **Deprecated**, kept for compatibility: this single secret is still accepted if set |

### Cache

| Variable | Default | Description |
|---|---|---|
| `CACHE_ENABLED` | `1` | Cache switch; `0` disables it and creates no database |
| `CACHE_DB_PATH` | `data/cache.db` | Cache database path |
| `CACHE_MAX_SIZE` | `1000` | Max entries, `0` unlimited; evicts least-recently-used |
| `CACHE_TTL` | `604800` | TTL in seconds, 7 days by default |

---

## Token management

### Why a token table instead of a single secret

A single secret in an environment variable has two problems: changing it requires a container
restart, and there is no way to give different callers different permissions. Tokens now live
in SQLite behind a dedicated admin page:

| | Single secret | Now |
|---|---|---|
| Count | 1 | Unlimited |
| Changes | Edit env var + restart | A few clicks, **effective immediately** |
| Expiry | none | Configurable |
| Quotas | none | Daily and total |
| Usage stats | none | Total / today / last used |
| Leak response | Rotate globally | Disable that one token |

### Security design

- Tokens look like `nsfw_` plus 64 hex characters; the database stores **only a sha256 hash**,
  so a leak does not expose usable tokens
- The plaintext is shown only once, at creation time
- The admin port is bound to host `127.0.0.1` only by default (container side listens on `0.0.0.0` because of Docker port mapping)
- Auth failures return distinct status codes for easier debugging

### Opening the admin page

```
http://127.0.0.1:6087        # compose / docker run bind it to host 127.0.0.1
```

Open it **on the machine running the container** and log in with `ADMIN_TOKEN`.

> **Why `0.0.0.0` inside the container?**
> The admin listen address comes from `ADMIN_HOST`. The image defaults to `0.0.0.0`:
> `docker run -p 6087:6087` is a DNAT rule that forwards to the *container IP*:6087,
> never to the container's `127.0.0.1`. With a loopback-only bind, every host-side
> mapping ends in `Connection refused` — the startup log then prints
> `admin_host=127.0.0.1` as a hint.
>
> The security boundary lives on the **host** side: mapping
> `-p 127.0.0.1:6087:6087` (as this repo's compose does) still restricts the admin page
> to localhost.

**To reach it from another machine, prefer an SSH tunnel** rather than changing the
mapping to `"6087:6087"`:

```bash
ssh -L 6087:127.0.0.1:6087 user@your-server
# then open http://127.0.0.1:6087 locally
```

If you must expose it, put it behind a firewall allowlist or a reverse proxy with HTTPS,
and make sure `ADMIN_TOKEN` is strong.

### What the page does

| Action | Description |
|---|---|
| Create | Name, expiry (empty = never), daily quota (0 = unlimited), total quota (0 = unlimited) |
| List | Name, token prefix, expiry, quota config, total usage, today's usage, enabled state |
| Disable / enable | A disabled token is rejected immediately |
| Reset usage | Clears both counters (handy for monthly billing) |
| Delete | Removes it permanently |

### Open mode

- When **no tokens exist at all**, the API does **not** require authentication
- As soon as **at least one** token exists (enabled or not), authentication is enforced

This keeps upgrades from returning 401 everywhere, while tightening automatically once you
start using tokens.

### Calling the API

```bash
curl -X POST 'http://localhost:6086/api/url_check' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer nsfw_xxxxxxxxxxxxxxxx' \
  -d '{"url":"https://example.com/a.jpg"}'
```

### Auth error codes

| HTTP status | `msg` | Meaning |
|---|---|---|
| `401` | `Token invalid` | Missing, wrong, disabled, or expired token |
| `403` | `Token limit exceeded` | Daily or total quota exceeded |

### Admin API

The page is backed by JSON endpoints, usable for scripting. Authenticate with either the
session cookie or the `X-Admin-Token` header.

```bash
ADMIN=http://127.0.0.1:6087
KEY='X-Admin-Token: your-secret-here'

# List
curl "$ADMIN/api/admin/tokens" -H "$KEY"

# Create: 30-day expiry, 1000/day, 100000 total
curl -X POST "$ADMIN/api/admin/tokens" -H "$KEY" \
  -H 'Content-Type: application/json' \
  -d '{"name":"customer-a","expires_at":'$(( $(date +%s) + 30*86400 ))',"daily_limit":1000,"total_limit":100000}'

# Disable
curl -X PUT "$ADMIN/api/admin/tokens/1" -H "$KEY" \
  -H 'Content-Type: application/json' -d '{"enabled":false}'

# Reset usage
curl -X POST "$ADMIN/api/admin/tokens/1/reset" -H "$KEY"

# Delete
curl -X DELETE "$ADMIN/api/admin/tokens/1" -H "$KEY"
```

---

## API reference

### POST /api/url_check — check by URL

> **POST only.** GET returns `405`.

```bash
curl -X POST 'http://localhost:6086/api/url_check' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer your-token' \
  -d '{
    "url": "https://example.com/a.jpg",
    "qr_detect": "disable"
  }'
```

| Field | Required | Description |
|---|---|---|
| `url` | yes | Image URL, http/https only |
| `qr_detect` | no | `enable` / `disable`, default `disable`. Also detects QR codes when enabled |

`qr_detect` may also be passed in the query string (`POST /api/url_check?qr_detect=enable`);
the body value wins.

### POST /api/upload_check — check by uploading a file

```bash
curl -X POST 'http://localhost:6086/api/upload_check' \
  -H 'Authorization: Bearer your-token' \
  -F 'file=@/path/to/image.jpg'
```

### GET /api/cache_stats — cache statistics

```bash
curl 'http://localhost:6086/api/cache_stats'
```

Returns hit rate, entry count and database sizes. This endpoint **does not require auth**.

### Response format

```json
{
  "code": 200,
  "msg": "success",
  "data": {
    "sfw": 0.0014,
    "nsfw": 0.9986,
    "is_nsfw": true,
    "qr_status": "unknown"
  }
}
```

| Field | Description |
|---|---|
| `code` | `200` on success, anything else is a failure |
| `msg` | Message; explains the failure reason |
| `data.sfw` | Safety score `[0,1]`, higher is safer |
| `data.nsfw` | Risk score `[0,1]`, higher is riskier |
| `data.is_nsfw` | `true` when `nsfw > NSFW_THRESHOLD` (default 0.8) |
| `data.qr_status` | `unknown` not checked / `qr` contains a QR code / `not_qr` |

> If false negatives are costly for you, ignore `is_nsfw` and tier the `nsfw` score yourself —
> for example block above `0.8` and send `0.4-0.8` to human review.

### Business errors

| `code` | `msg` | Description |
|---|---|---|
| `-1000` | `Invalid URL, only http/https are supported.` | Malformed URL |
| `-1000` | `Unsupported file type: ..., only jpg/png/bmp/webp/gif allowed.` | Unsupported image type |
| `-1000` | `Download failed, status code: 404` | Failed to fetch the image |
| `-1000` | `Image parsing failed.` | Not a valid image |
| `-1000` | `File too large (>10MB), refusing to process.` | Exceeds the size limit |
| `-1000` | `Image too large to decode.` | Exceeds the pixel limit |
| `-1000` | `Invalid qr_detect value, ...` | `qr_detect` only accepts enable / disable |

---

## Result cache

URL check results are cached in SQLite (`data/cache.db` by default). A second request for the
same URL hits the cache, dropping end-to-end latency from tens of milliseconds to under 1ms.

- **Shared across workers** (not one copy per process)
- Survives container restarts
- The cache key includes the `qr_detect` flag, so the two modes never mix
- **Only model output is cached** (scores plus QR status), not the verdict — so changing
  `NSFW_THRESHOLD` applies immediately to existing entries without clearing the cache
- Only successful results are cached

Hit rate depends on your traffic. If URLs are rarely repeated this buys you almost nothing;
turn it off with `CACHE_ENABLED=0`.

```bash
curl 'http://localhost:6086/api/cache_stats'
```

```json
{
  "code": 200,
  "data": {
    "backend": "sqlite",
    "hits": 120, "misses": 30, "hit_rate": 0.8,
    "rows": 30, "expired_rows": 0,
    "db_bytes": 40960, "wal_bytes": 45352, "total_bytes": 86312
  }
}
```

---

## Data and backups

`docker-compose.yaml` mounts host `./data` into `/opt/nsfw/data`:

```
data/
├── cache.db       # result cache (rebuildable, losing it only loses cache)
└── tokens.db      # tokens and usage records (important, back this up)
```

### Back up the cache (online, no downtime)

```bash
docker compose exec nsfw python -m app.cache /opt/nsfw/data/cache-backup.db
```

Uses SQLite's official backup API, so it is consistent even while running.

### Back up the token database (stop the container first)

```bash
docker compose stop
cp data/tokens.db "data/tokens-$(date +%F).db"
docker compose start
```

> ⚠️ Both databases use WAL mode, so recent writes sit in the `-wal` file.
> **Do not `cp cache.db` or `cp tokens.db` while running** — you will lose the latest data.
> Measured: after 10 records the main file was only 4KB, almost everything was in `-wal`.

### Restore

Copy the backup back over `data/cache.db` or `data/tokens.db`, remove the matching
`-wal` and `-shm` files, then start the container.

---

## Performance

### Benchmarks

Measured on a 16-core x86_64 machine, one 384×384 input, inference only (no download or decode).

| `ORT_NUM_THREADS` | 1 | 2 | 4 | 8 | 16 |
|---|---|---|---|---|---|
| Single inference | 58ms | 36ms | 23ms | 19ms | 18ms |

Returns diminish after 4 threads; 8 and 16 threads differ by about 1ms. End-to-end latency adds
image download and decoding on top, usually a few tens of milliseconds.

> ⚠️ **Do not remove `ORT_ALLOW_SPINNING=0`.** It is a default of this service (exported in
> `run.sh` and set as a session option in `engine.py`). With spinning enabled, multi-threaded
> inference degrades noticeably: at 16 threads this model goes from 18ms to 32ms, and the
> previous larger model went from 178ms to 381ms — a dozen threads spinning idle fight each
> other for CPU, and the more threads the worse it gets.

### Thread count is the lever that matters

`WORKERS × ORT_NUM_THREADS` above the physical core count is pure waste. Two directions:

- **Low latency** → `WORKERS=1`, `ORT_NUM_THREADS=cores`
- **High throughput** → `WORKERS=cores/2`, `ORT_NUM_THREADS=2`

### Reference configurations

Latency below is **inference only**; add download and decode for end-to-end numbers:

| Machine | `WORKERS` | `ORT_NUM_THREADS` | Memory | Single inference |
|---|---|---|---|---|
| 1 core 1GB | 1 | 1 | ~140MB | ~58ms |
| 2 cores 1GB | 1 | 2 | ~140MB | ~36ms |
| 2 cores 2GB | 1 | 2 | ~140MB | ~36ms |
| 4 cores 2GB | 2 | 2 | ~270MB | ~36ms, 2x throughput |
| 4 cores 4GB | 1 | 4 | ~140MB | ~23ms |

### Resource usage

- About **135MB** resident per worker (including the 6.4MB model)
- About **2.2s** startup (includes model warm-up, so the first request is not slow)
- About **160MB** image size

### Overload protection

Inference is CPU-bound; letting requests pile up only makes everyone slower. Two gates:

- `INFER_CONCURRENCY`: concurrent inference requests; extras queue
- `LIMIT_CONCURRENCY`: connection limit; extras get 503 immediately

Upstream callers should back off and retry on 503 rather than pushing harder.

---

## Project layout

```
nsfw/
├── app/
│   ├── main.py               # FastAPI app, lifespan, CORS
│   ├── routers.py            # route registration
│   ├── engine.py             # inference engine: session/preprocess/fetch/QR
│   ├── cache.py              # result cache (SQLite)
│   ├── tokens.py             # token storage and auth (SQLite)
│   ├── admin.py              # token admin port (page + API)
│   ├── api/
│   │   ├── url_check.py      # URL check
│   │   ├── upload_check.py   # upload check
│   │   └── index.py          # index page
│   ├── middleware/auth.py    # auth middleware
│   ├── onnx/model_nsfw_int8.onnx  # model (int8 quantized, 6.4MB)
│   ├── static/html/          # built-in web UI
│   └── requirements.txt
├── .github/workflows/
│   └── docker-publish.yml    # CI: build + smoke test + push to GHCR
├── Dockerfile
├── docker-compose.yaml
├── build.sh                  # local image build
├── install.sh                # in-image dependency install
├── run.sh                    # launcher (auto worker/thread sizing)
└── data/                     # runtime data (cache + tokens, not in git)
```

---

## FAQ

**First call is slow?**

Startup warms up inference once, so the first real request does not pay the weight-packing
cost. Total startup to serving is about 2.2s. Use a health check to wait for readiness in
production.

**Do I need to clear the cache after changing `NSFW_THRESHOLD`?**

No. The cache stores raw model scores; the verdict is computed on the way out using the
current threshold.

**Do I need to restart after changing a token?**

No. Auth queries the database on every request, so changes apply immediately.

**Are usage counters accurate with multiple workers?**

Yes. Counters use atomic `UPDATE ... SET total_used = total_used + 1` in SQLite, shared
across processes.

**Why doesn't URL checking support GET?**

Image URLs often contain `?`, `&`, `#` which need extra escaping in a query string, and GET
URLs are recorded verbatim by browsers, CDNs and access logs — leaking what you are checking.
POST only.

**The admin page won't load (`ERR_CONNECTION_REFUSED`)?**

Check the startup line `[run.sh] admin port=6087 admin_host=...` in the container log.

- `admin_host=127.0.0.1` → the container only bound its loopback, so no host `-p` mapping can
  ever reach it. The image defaults to `ADMIN_HOST=0.0.0.0`; if you are on an older image or
  overrode `environment`, drop that override.
- `admin_host=0.0.0.0` but still refused → the port mapping is missing: check that `docker ps`
  shows 6087, and that compose maps `"127.0.0.1:6087:6087"` (localhost only) or `"6087:6087"`
  (reachable from the LAN).
- From another machine → it is bound to the host's `127.0.0.1` by default; use an SSH tunnel:
  `ssh -L 6087:127.0.0.1:6087 user@your-server`.
- It opens but login fails → `ADMIN_TOKEN` is unset or mistyped.

**No access logs in the output?**

`LOG_LEVEL=warning` disables the access log to reduce disk writes. Set `LOG_LEVEL=info` if
you need it.

**Does it detect AI-generated images?**

Noticeably less accurately than real photos. The training data is mostly real images and
traditional artwork.

**`docker pull` fails with `unauthorized` / `denied`?**

Two likely causes: (1) the package is still Private — that is the default after the first
push; log in first, or flip the visibility to Public (see
[Option 3](#option-3-pull-the-prebuilt-image-ghcr)); (2) the image name casing is wrong —
the registry accepts lowercase only, so it must be `ghcr.io/qcenjoyll/nsfw-detector`.

**Can it run on arm64 (Raspberry Pi / ARM cloud / some NAS)?**

Not yet. `zxing-cpp==2.3.0` ships no aarch64 wheel, so pip falls back to a source build and
the image has no cmake / g++, which fails. The workaround is to add a build toolchain stage
to the Dockerfile and build locally on the arm64 host.

**Why did CI not build an image for my commit?**

Commits touching only `*.md` or `LICENSE` are skipped by `paths-ignore`. Pull requests also
build and smoke-test but never push.

**How do I pin the image used in production?**

Use a `sha-<short>` or semver tag instead of `latest`:

```bash
docker pull ghcr.io/qcenjoyll/nsfw-detector:sha-9bf94ea
```

Point `image:` in your local `docker compose` at a fixed tag; rolling back is just changing
the tag and restarting.

---

## License and credits

Licensed under AGPL-3.0; see [LICENSE](LICENSE).

The model comes from [Marqo/nsfw-image-detection-384](https://huggingface.co/Marqo/nsfw-image-detection-384).
Thanks to the authors. Please comply with the model's own license terms.

> Detection is never 100% accurate. No content moderation solution is perfectly precise;
> combine automated screening with human review rather than relying on a verdict alone.
