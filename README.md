<!-- last_verified: 2026-05-26 -->
# Video to Insights Pipeline

> Repo: **[backblaze-labs/video-to-insights-nvidia-parakeet-llama](https://github.com/backblaze-labs/video-to-insights-nvidia-parakeet-llama)**

Paste a YouTube URL. The backend downloads the source video with `yt-dlp`,
uploads it to **[Backblaze B2](https://www.backblaze.com/sign-up/ai-cloud-storage?utm_source=github&utm_medium=referral&utm_campaign=ai_artifacts&utm_content=video-to-insights-pipeline)**,
transcribes it with NVIDIA Parakeet (free hosted NIM), extracts a small
list of seekable "insight" sections with Llama-3.3-70B-Instruct (also free),
and serves the B2-hosted video back to the browser with clickable insight
cards that jump the player to each section.

![Video to Insights dashboard](./video-to-insights.png)

B2 is the system of record: the source MP4, the transcript JSON, the
insights JSON, and a schema-versioned manifest all live in B2. The
frontend keeps nothing but a list of recent `job_id`s in `localStorage`.

## Pipeline

```
YouTube URL
   │
   ▼  yt-dlp (subprocess, asyncio.to_thread)
  source.mp4  ─►  Backblaze B2  (source-of-truth for the video)
   │
   ▼  ffmpeg extract + chunk (subprocess, asyncio.to_thread)
  mono 16k WAV
   │
   ▼  NVIDIA Parakeet (free NIM, segment timestamps)
  transcript.json  ─►  B2
   │
   ▼  NVIDIA Llama-3.3-70B-Instruct (free NIM, JSON mode)
  insights.json    ─►  B2
   │
   ▼
  manifest.json    ─►  B2  (schema_version=1, points at the three above)
```

Without `NVIDIA_API_KEY` the pipeline still finishes — source video lands
in B2, the manifest records `analysis_status: "skipped_no_api_key"`, and
the UI renders the player with a muted notice in place of the insights
panel. This makes the B2 half of the sample fully usable on B2 creds alone.

## What it looks like

Idle:

```
┌────────────────────────────────────────┐
│ Video to Insights Pipeline             │
│ [ Video URL (YouTube) ............ ]   │
│ [ Run ]                                │
│ Recent jobs (localStorage):            │
│  · f3b4… https://youtube.com/…  done   │
└────────────────────────────────────────┘
```

Done:

```
┌──────────────────────────────┐ ┌────────────────┐
│  <video controls>            │ │ Insights       │
│  (presigned B2 source)       │ │ ▸ 00:00 Intro  │
│                              │ │ ▸ 02:34 Setup  │
│  ↗ View source video         │ │ ▸ 08:12 Demo   │
└──────────────────────────────┘ └────────────────┘
```

Clicking an insight card sets `videoRef.current.currentTime` and resumes
playback — no clip slicing, just `currentTime` + HTTP Range.

## Quick Start

Prereqs:
- Node.js ≥ 20, pnpm ≥ 9, Python ≥ 3.11
- `ffmpeg` and `ffprobe` on PATH (`brew install ffmpeg` on macOS, `apt-get install ffmpeg` on Debian/Ubuntu)
- `yt-dlp` on PATH (`pip install --upgrade yt-dlp` or `brew install yt-dlp`)
- A free **[Backblaze B2 account](https://www.backblaze.com/sign-up/ai-cloud-storage?utm_source=github&utm_medium=referral&utm_campaign=ai_artifacts&utm_content=video-to-insights-pipeline)**
- (Optional) A free **[NVIDIA NIM API key](https://build.nvidia.com/)** if you want transcription and insights

```bash
# 1) Clone and install JS deps
git clone https://github.com/backblaze-labs/video-to-insights-nvidia-parakeet-llama.git
cd video-to-insights-nvidia-parakeet-llama
pnpm install

# 2) Set up the Python backend
cd services/api
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cd ../..

# `pyproject.toml` is the canonical dependency manifest. `requirements.txt`
# is maintained in parallel so the `pip install -r` flow above still works
# for contributors who don't want to install the project as a package.

# 3) Configure
cp .env.example .env
# edit .env with your B2 credentials (and optionally NVIDIA_API_KEY)

# 4) Run
pnpm dev
```

Frontend at `localhost:3000`, API at `localhost:8000`. `pnpm dev` runs
`pnpm doctor` first to surface the common setup gotchas (wrong Node /
Python, missing `ffmpeg`/`yt-dlp`, placeholder `.env`).

## Environment variables

Per the parent [sampleapps/CLAUDE.md](../CLAUDE.md), B2 keys use the
canonical names — no `AWS_*`, no `B2_S3_*`.

| Key | Required | Notes |
|---|---|---|
| `B2_ENDPOINT` | yes | `https://s3.<region>.backblazeb2.com` |
| `B2_REGION` | yes | e.g. `us-west-004`. Derived from your bucket. |
| `B2_KEY_ID` | yes | B2 application key id |
| `B2_APPLICATION_KEY` | yes | B2 application key |
| `B2_BUCKET_NAME` | yes | Bucket the sample writes into |
| `NVIDIA_API_KEY` | no | When unset, pipeline finishes with `done_no_analysis` |
| `NVIDIA_ASR_MODEL` | no | Default `nvidia/parakeet-tdt-0.6b-v2` |
| `NVIDIA_INSIGHTS_MODEL` | no | Default `meta/llama-3.3-70b-instruct` |
| `WORK_DIR` | no | Per-job scratch + state files. Default `./.work` |
| `MAX_VIDEO_SECONDS` | no | Reject videos longer than this. Default `5400` |
| `MAX_CONCURRENT_JOBS` | no | Lifespan semaphore. Default `1` |
| `ALLOWED_VIDEO_HOSTS` | no | Comma-separated host allowlist |

## API

```
POST   /jobs                     {youtube_url, segment_seconds?} -> {job_id, status}
GET    /jobs/{id}                full JobStatus
GET    /jobs/{id}/source         302 -> presigned B2 source.mp4 (1h)
GET    /jobs/{id}/manifest       302 -> presigned manifest.json
GET    /jobs/{id}/transcript     302 -> presigned transcript.json
GET    /jobs/{id}/insights       302 -> presigned insights.json
DELETE /jobs/{id}                set cancel flag; returns updated JobStatus
GET    /health                   {status, b2_connected, nvidia_configured}
```

`segment_seconds` is accepted for forward-compat but currently unused —
the MVP does no clip slicing.

## Free-tier model notes

`nvidia/parakeet-tdt-0.6b-v2` accepts ~24 minutes of audio per call.
Longer videos are chunked at ~22 minutes per piece by `ffmpeg -f segment`;
the ASR client shifts each chunk's segment timestamps by its start offset
so the stitched transcript reads as if processed in one pass.

`meta/llama-3.3-70b-instruct` is called in JSON mode and asked for 3-6
non-overlapping insight sections with `start_seconds` / `end_seconds`
that align to the supplied transcript timeline.

NIM's free tier limits requests to roughly 40 req/min and starts you with
~1000 credits. The pipeline is sequential per job, so a single user
won't hit the rate cap.

## Reliability / scope

- **Single worker.** State lives at `${WORK_DIR}/jobs/{job_id}.json` per
  process. Don't run multiple uvicorn workers — they'll race.
- **Job state is one file per job, written via tmp + `os.replace`** (atomic
  on the same filesystem). No in-memory dict, no `asyncio.Lock`, no Redis.
- **Cooperative cancellation.** `DELETE /jobs/{id}` flips a flag; the
  pipeline checks it between stages. A subprocess already in flight will
  finish its current stage.
- **Cleanup.** The per-video scratch dir is removed in a `finally` block
  after every job. B2 holds the durable artifacts.

## Legal

The sample is meant for content you own, content under a permissive
license, or fair-use research. Respect YouTube's
[Terms of Service](https://www.youtube.com/static?template=terms) and
the rights of creators. This is **not** a downloader product.

## Documentation map

| Doc | Purpose |
|---|---|
| [AGENTS.md](AGENTS.md) | Control surface for coding agents — start here |
| [ARCHITECTURE.md](ARCHITECTURE.md) | System layout, layering, data flows |
| [docs/features/youtube-ingest.md](docs/features/youtube-ingest.md) | yt-dlp + host allowlist + error taxonomy |
| [docs/features/ai-analysis.md](docs/features/ai-analysis.md) | Parakeet + Llama call shapes + graceful degradation |
| [docs/features/video-playback.md](docs/features/video-playback.md) | Presigned URLs + insight-card seek |
| [docs/app-workflows.md](docs/app-workflows.md) | Submit → poll → done; cancel |
| [docs/SECURITY.md](docs/SECURITY.md) | URL allowlist, subprocess hygiene, presigned URL expiry |
| [docs/RELIABILITY.md](docs/RELIABILITY.md) | Single-worker constraint, atomic state file, partial success |

## License

MIT — see [LICENSE](LICENSE).
