# Clerkr deployment notes for Vexa (ClerkrDev/vexa, branch `clerkr`)

This branch is upstream tag **v0.12.28** (`f1a62fbbd`, 2026-10-09) plus this file. No code is
changed. The running service is the published image `vexaai/vexa-lite:v0.12.28`; this branch
exists to pin what is deployed and to record what the source says about running it.

- Upstream: https://github.com/Vexa-ai/vexa · licence **Apache-2.0** (SPDX `Apache-2.0`).
- Newer at the time of writing: upstream `main` carries a v0.12.29 hotfix (Google Meet refused
  join is a typed refusal with re-knock, not a 15-minute lobby wait, upstream #1118) and the image
  `vexaai/vexa-lite:v0.12.29` is published, but no git tag `v0.12.29` exists. The `v0.13.x` tags
  are prereleases.
- Written 2026-10-31 from the source at v0.12.28 and `main` of that day. Where the source does not
  settle a point, the point is listed in section 9 rather than guessed.

## 1. The three shapes, and why Lite is the one for Railway

| Shape | Bots run as | Needs | On Railway |
|---|---|---|---|
| Compose (`deploy/compose`) | one container per bot, spawned by the `runtime` service over `/var/run/docker.sock` (`RUNTIME_BACKEND=docker`, `BROWSER_IMAGE=vexaai/vexa-bot:v012`) | the host Docker socket | impossible, no socket |
| Kubernetes (`deploy/helm`) | one Pod per bot (`RUNTIME_BACKEND=k8s`) | a cluster | impossible |
| **Lite** (`deploy/lite`) | child processes inside the one container (`RUNTIME_BACKEND=process`), sharing one Xvfb display and one PulseAudio | Postgres + an S3 store | **this is what runs** |

Lite's own stated limits (`deploy/lite/README.md`, `docs/docs/deployment-lite.mdx`):
bots share a single Xvfb `:99`, "best for one browser session at a time"; the in-container
Valkey is ephemeral unless `/var/lib/redis` is mounted (we point `REDIS_URL` at Railway's Redis
instead); the agent API on `:8100` is not gateway-fronted; no authenticated-bot provisioning.

## 2. The image

- `deploy/lite/Dockerfile.lite`, multi-stage on `mcr.microsoft.com/playwright:v1.56.0-noble`
  (bot builder, terminal builder, valkey builder, final). Published for amd64 and arm64, about
  4.4 GB. Railway pulls it; nothing is built from this repository.
- `EXPOSE 8056 3001 8100 6080`. `HEALTHCHECK` on the gateway `/health` (30 s interval, 10 s
  timeout, 120 s start period, 3 retries). `ENTRYPOINT /entrypoint.sh`, `CMD supervisord -n -c
  /etc/supervisor/conf.d/vexa.conf`.
- Upstream runs the container with `--shm-size=2g` (`deploy/lite/Makefile`, target `up`). The bot
  starts Chromium **without** `--disable-dev-shm-usage` (`docs/docs/deployment-kubernetes.mdx`,
  "/dev/shm — 2 Gi"), and Teams is the platform that consumes it. Railway offers no shm flag;
  **measured on Railway: `/dev/shm` is 62 MB** (`df -h /dev/shm` inside the running service).

## 3. Processes (supervisord, `deploy/lite/supervisord.conf`)

| Program | Command | Port | Reachable |
|---|---|---|---|
| redis | `valkey-server --bind 127.0.0.1 --port 6379 …` | 6379 | loopback (unused when `REDIS_URL` is external) |
| xvfb | `Xvfb :99 -screen 0 1920x1080x24 …` | :99 | – |
| fluxbox | `fluxbox` | – | – |
| pulseaudio | `pulseaudio --system --disallow-exit …` | – | – |
| pulseaudio-setup | `setup-pulseaudio-sinks.sh` (null sink `tts_sink`, remapped `virtual_mic`) | – | – |
| x11vnc / websockify | VNC of the display, noVNC | 5900 / 6080 | not exposed |
| admin-api | `python -m admin_api` | 8001 | loopback only |
| runtime | `python -m runtime_kernel` (`RUNTIME_BACKEND=process`) | 8090 | loopback only |
| agent-api | `uvicorn control_plane.api:app --port 8100` | 8100 | exposed by the image, not given a Railway domain |
| meeting-api | `python -m meeting_api` | 8080 | loopback only |
| gateway | `python -m gateway` | 8056 | **the public domain** |
| terminal | `node server.mjs` (Next.js workbench) | 3001 | not given a Railway domain |

Bots are not supervisord programs: the runtime spawns `/usr/local/bin/vexa-bot-launch`
(`cd /app/core/meetings/services/bot && node dist/index.js`) per meeting, on the shared display
and audio graph. There is no Celery, no beat: scheduled joins (`AUTO_JOIN_*`), webhook retries
(Redis `webhook_retry_queue`) and calendar sync are loops inside meeting-api and admin-api.

`/entrypoint.sh` defaults every variable supervisord interpolates, derives `REDIS_URL` when unset,
waits up to 60 s for Postgres with `pg_isready`, starts `provision-key.sh` in the background and
hands off to supervisord. `provision-key.sh` creates a `self-host@vexa.ai` user and two `bot,tx`
API keys on first boot and writes them to `/run/vexa/key.env` for the UIs; it is a no-op when
`VEXA_API_KEY` is set. All thirteen programs reached RUNNING on the first Railway boot.

## 4. Environment variables (exact names)

Required or placeholder-guarded (a service refuses to boot without them, `config.v1.json` per
service; forbidden values include `changeme`, `secret`, `default`, `lite-internal-secret`):

| Variable | Who needs it | How upstream makes it | Ours |
|---|---|---|---|
| `ADMIN_TOKEN` and `ADMIN_API_TOKEN` | meeting-api (required-explicit), admin-api; **the same value** in both names | `python3 -c "import secrets; print(secrets.token_hex(32))"` (entrypoint) or `openssl rand -hex 32` (compose README) | generated that way, set in Railway |
| `INTERNAL_API_SECRET` | gateway, admin-api (required-explicit), agent-api | same | generated |
| `VEXA_DISPATCH_SIGNING_KEY` | agent plane; default `dev-dispatch-signing-key` is a placeholder | same | generated |
| `NEXTAUTH_SECRET`, `JWT_SECRET` | terminal; defaults are placeholders | same | generated |

Database and Redis (no `DATABASE_URL`: Lite installs asyncpg only, so each service builds
`postgresql+asyncpg://` from parts):

| Variable | Ours |
|---|---|
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` | Railway references `${{Postgres.PGHOST}}`, `PGPORT`, `PGDATABASE`, `PGUSER`, `PGPASSWORD` |
| `REDIS_URL` | `${{Redis.REDIS_URL}}` (an external URL is honoured by the entrypoint; the in-container Valkey still starts, idle) |

Transcription (section 6): `TRANSCRIPTION_SERVICE_URL`, `TRANSCRIPTION_SERVICE_TOKEN`,
`TRANSCRIPTION_MODEL` (empty → `whisper-1`), `TRANSCRIBE_ENABLED` (empty means true),
`RECORDING_ENABLED` (empty means true).

Storage (section 5): `MINIO_ENDPOINT` (host:port, or a full URL), `MINIO_SECURE`, `MINIO_BUCKET`,
`MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`; or the newer spellings `S3_ENDPOINT` (full URL),
`S3_ACCESS_KEY`, `S3_SECRET_KEY`. `AWS_DEFAULT_REGION` for boto3 (see section 5).

Public URLs and names: `VEXA_PUBLIC_API_URL`, `TERMINAL_PUBLIC_URL`, `NEXTAUTH_URL` (the gateway
domain), `DEFAULT_BOT_NAME` (ours: `Clerkr`; the per-request `bot_name` wins), `LOG_LEVEL`.

Optional, left unset: `VEXA_ENV` (anything but `production` is the development posture: public
`/docs`), `GUARD_ENABLED` (default true, gateway rate limiting), `VEXA_LOBBY_BUDGET_S` (900),
`BOT_UI_LOCALE`, `VEXA_SYSTEM_WEBHOOK_*`, `VEXA_SERVICE_AUTHORITY_*`, the `ANTHROPIC_*` /
`HOST_CLAUDE_CREDENTIALS` / `VEXA_AGENT_MODEL` family (the agent plane, unused by Clerkr),
`VEXA_API_KEY` (would skip the self-host mint).

Postgres: upstream pins `postgres:17-alpine`. Railway's plugin is `postgres-ssl:18`. Schema
"converges in-process on service startup" (`ensure_schema()` in admin-api and meeting-api); there
is no migration command and no pre-deploy step. Upgrading is a new image tag.

## 5. Recordings and storage

- meeting-api builds one boto3 S3 client (`meeting_api/__main__.py`, `recordings/adapters.py`):
  bucket `MINIO_BUCKET` (fallback `RECORDING_BUCKET`, default `vexa`); endpoint `S3_ENDPOINT` or
  `http(s)://MINIO_ENDPOINT` by `MINIO_SECURE`; keys `S3_ACCESS_KEY` / `MINIO_ACCESS_KEY` and
  `S3_SECRET_KEY` / `MINIO_SECRET_KEY`. No region or addressing style is passed, so for AWS the
  region comes from boto3's environment: we set `AWS_DEFAULT_REGION=eu-north-1`. The Helm chart
  does the same through a mounted `AWS_CONFIG_FILE`.
- The bot does not write to S3 itself; recordings flow bot → meeting-api → bucket. The bot's
  `BOT_S3_*` variables are the authenticated-bot userdata store, which Lite does not package.
- Recording is **audio only** (`docs/docs/api/meetings.mdx`: "there is no video recording path").
- With `MINIO_ENDPOINT` empty the entrypoint prints "recordings disabled" and the service boots;
  meeting-api's `/health` then reports `object_storage: not_configured`.
- Ours: bucket `clerkr-meeting-bot-dev` in `eu-north-1`, `MINIO_ENDPOINT=s3.eu-north-1.amazonaws.com`,
  `MINIO_SECURE=true`; the IAM user's keys as `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY`.
- Upstream single-host installs use versitygw (Apache-2.0) as the S3 server (ADR 0038, MinIO removed).

## 6. Transcription

- The bot's client (`core/meetings/modules/whisper/src/transcription-client.ts`) POSTs multipart
  WAV chunks (up to 30 s) to `TRANSCRIPTION_SERVICE_URL` + `/v1/audio/transcriptions` (appended
  only when missing) with form parts `file`, `model` (`TRANSCRIPTION_MODEL` or `whisper-1`),
  `language` (the bot's `language` from `POST /bots`, omitted in auto mode),
  `min_silence_duration_ms`, and `Authorization: Bearer <TRANSCRIPTION_SERVICE_TOKEN>`.
  Per-request timeout 30 s, retries with backoff.
- Backends: (a) upstream's own unit `deploy/transcription` (faster-whisper; GPU default
  `large-v3-turbo` int8, about 2.1 GB VRAM; the CPU variant `docker-compose.cpu.yml` defaults to
  `small`, "witnessed to keep pace on 4–6 vCPU"; `medium` and `large*` are refused as CPU-unsafe,
  `large-v3-turbo` measured 0.95x realtime on a 12-core CPU); (b) the hosted unit behind a
  `vexa.ai/account` token, which the source and tests address as `https://transcription.vexa.ai`;
  (c) any OpenAI-compatible endpoint (Groq, OpenAI, Scaleway, vLLM), with `TRANSCRIPTION_MODEL`
  set to the served name.
- `POST /bots` answers **503** when transcription is requested and no backend is configured,
  before any meeting row is written; `transcribe_enabled=false` spawns a capture-only bot.
  meeting-api's `/health` reports `stt: misconfigured` while the URL is set and the token is not,
  and in that state `POST /bots` does **not** refuse: observed 2026-11-02, a request with the URL
  set and no token created meeting id 1 (`status: requested`) and spawned a bot. Leave the URL
  unset until the token exists, or expect bots whose transcription will fail.
- Ours for the spike: `TRANSCRIPTION_SERVICE_URL=https://transcription.vexa.ai`, token from
  Niels's vexa.ai account, `TRANSCRIPTION_MODEL` unset. The hosted unit's region is not stated by
  upstream; the spike uses test audio only.

## 7. The API Clerkr will talk to (gateway, `X-API-Key`)

`docs/docs/api/*.mdx` at this tag. Platforms `google_meet`, `zoom`, `teams` (+ Jitsi, live
validation pending). Highlights:

- `POST /bots` `{platform, native_meeting_id, bot_name?, language?, task?, transcribe_enabled?,
  recording_enabled?}`; a running bot for the same meeting answers 409.
- `GET /bots/status`, `GET /transcripts/{platform}/{native_meeting_id}`, meeting participants and
  recordings endpoints (sections "Meeting participants", "Recordings"), `DELETE
  /bots/{platform}/{native_meeting_id}` (stop), `DELETE /meetings/{platform}/{native_meeting_id}`
  (delete artifacts), `PATCH /meetings/...`.
- Webhooks are **per user**, one URL: `PUT /user/webhook` `{webhook_url, webhook_secret,
  webhook_events: {"meeting.completed": true, ...}}`. Events `meeting.started`,
  `meeting.status_change`, `meeting.completed`, `bot.failed`. Headers `X-Webhook-Signature:
  sha256=<hmac_sha256(secret, "<X-Webhook-Timestamp>." + raw_body)>`, `X-Webhook-Timestamp`,
  and `Authorization: Bearer <secret>` (legacy). Delivery history `GET /user/webhook/deliveries`.
  Transcript updates are not pushed; poll the transcript endpoint or use the live feed.
- `GET /user/webhook` reads the configuration back with the secret masked to its last four
  characters (`webhook_secret_set: true`); the admin read `GET /admin/users/{id}` shows the URL
  and events but never the secret, so it is not the place to check whether one is stored.
- Admin (loopback 8001 in Lite, so through `railway ssh`): `X-Admin-API-Key: <ADMIN_API_TOKEN>`,
  `POST /admin/users {email, name}`, `POST /admin/users/{id}/tokens?scopes=bot,tx`,
  `GET /admin/users/email/{email}`, `DELETE /admin/tokens/{id}`. A user's default
  `max_concurrent_bots` is 3.

## 8. Railway layout, as deployed (2026-10-31)

Project `clerkr-meeting-bot`, workspace Clerkr (preferred region `europe-west4-drams3a`,
Amsterdam), environment `development`.

| Service | Source | Notes |
|---|---|---|
| Postgres | Railway plugin, `ghcr.io/railwayapp-templates/postgres-ssl:18`, 50 GB volume | region EU West |
| Redis | Railway plugin, `redis:8.2` | region EU West |
| `vexa-lite` | image `vexaai/vexa-lite:v0.12.28` | public domain on target port 8056, health check `/health` (timeout 300 s), restart on failure, resource cap **8 GB / 8 vCPU** (the plan allows 24 GB / 24 vCPU per service; a replica needs at least 4 GB) |
| `webhook-capture` | a 60-line Python server, spike only | public domain on 8080; records every delivery; read side guarded by a token |

**Railway's health check probes the port in the `PORT` variable, not the domain's target port.**
Without `PORT` the first deploy succeeded (no health check path yet) and every redeploy after the
path was set died at the 300 s timeout with all programs running. `PORT=8056` on the service fixes
it; it is safe because every Lite program that listens gets an explicit `PORT` from supervisord
(`environment=` lines), so nothing else picks the container-level value up.

Railway gives: one container per service, private IPv6 network between services, variable
references, `railway ssh` into the running container. Railway does not give: a Docker socket,
Kubernetes, GPUs, an shm-size flag (62 MB measured), a second process in the same container
(Lite's supervisord is what makes the single-container shape possible).

Operations: logs with `railway logs -s vexa-lite`; a shell with `railway ssh -s vexa-lite`;
service tree with `supervisorctl status` inside; upgrade by changing the image tag on the service.

## 9. Not settled from the source

1. `/dev/shm`: Railway gives 62 MB, upstream wants 2 GB. Whether Google Meet survives it is what
   the spike shows; upstream names Teams as the platform that needs it.
2. PulseAudio `--system` and Xvfb inside an unprivileged Railway container: both reached RUNNING
   and the sinks exist; confirmed only by a bot actually capturing audio.
3. Postgres 18 (Railway plugin) versus upstream's pinned 17. Schema convergence ran on 18.
4. boto3 against `eu-north-1` with `endpoint_url` set and no explicit `region_name` or addressing
   style: `AWS_DEFAULT_REGION` should suffice; the first upload proves it.
5. Whether `https://transcription.vexa.ai` is exactly the endpoint a `vexa.ai/account` token is
   valid for, and where it runs. Its `/health` answers; the first transcribed chunk proves the pairing.
6. How many bots one Lite container can really run at once; upstream says one browser session.
7. Railway's private network is IPv6: `pg_isready` and asyncpg resolved `postgres.railway.internal`
   on the first boot, so this one is settled in practice.
