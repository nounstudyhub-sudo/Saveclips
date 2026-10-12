# SaveClip — Social Video Downloader

Paste a public link from **TikTok, X/Twitter, Instagram, Facebook, Snapchat or YouTube**, pick MP4 High, MP4 Standard or MP3, and download the file.

- **Frontend:** Next.js 16 (App Router), React 19, Tailwind CSS 4, Framer Motion, Lucide icons
- **Validation:** Zod (URL schema, format enum) on both client and server
- **Engine:** [yt-dlp](https://github.com/yt-dlp/yt-dlp) for extraction, [ffmpeg](https://ffmpeg.org) for merging and MP3 conversion
- **Deploy target:** Render (Docker runtime), also runs locally with Node 22

## How it works

1. `POST /api/download` validates the link, strips tracking parameters, and returns title, thumbnail, duration and the options that exist for that video.
2. Clicking an option calls `POST /api/jobs`, which starts a yt-dlp job writing into a private temporary folder.
3. The browser polls `GET /api/jobs/:id` and shows real progress (`queued → downloading → processing → ready`).
4. When ready, the browser calls `GET /api/jobs/:id/file`, which streams the file as an attachment. The file is deleted once fully delivered, or after `JOB_TTL_MINUTES` if never collected.

## Project structure

```
Dockerfile                          Production image: Node 22 + ffmpeg + yt-dlp (+ yt-dlp-ejs)
render.yaml                         Render Blueprint (one-click deploy)
.env.example                        All configuration variables
src/
  app/
    page.tsx                        Home: hero, platform chips, downloader, FAQ + FAQPage JSON-LD
    how-it-works/ terms/ dmca/ privacy/
    api/
      download/route.ts             Step 1: validate link, return metadata + options
      jobs/route.ts                 Step 2: start a download job (202)
      jobs/[id]/route.ts            Step 3: poll status
      jobs/[id]/file/route.ts       Step 4: stream file, then delete it
      health/route.ts               Health check (Render) + tool versions
  components/
    URLInput.tsx                    Input, Auto-paste button, Zod validation
    VideoResultCard.tsx             Thumbnail, title, duration, platform badge, format rows with live progress
    Downloader.tsx                  Orchestrates the loader steps and the result card
    Header.tsx Footer.tsx ThemeToggle.tsx PlatformChips.tsx FAQSection.tsx LegalPage.tsx
  instrumentation.ts                Clears temp files on boot and starts the job janitor
  lib/
    config.ts                       Environment-driven settings
    platforms.ts                    Platform allow-list, URL sanitizer, Zod schemas, shared types
    ytdlp.ts                        yt-dlp wrapper: metadata, format arguments, error mapping
    jobs.ts                         In-memory job store, concurrency cap, timeouts, cleanup
    api.ts                          Response helpers, URL parsing, rate-limit wrapper
    ratelimit.ts                    Per-IP fixed-window limiter
```

---

## 1. Run locally

**Requirements**

| Tool | Why | Install |
|---|---|---|
| Node.js 20.9+ (22 recommended) | Runs Next.js | https://nodejs.org |
| Python 3.10+ with pip | Installs yt-dlp | https://python.org |
| ffmpeg | Merges video+audio, creates MP3 | macOS `brew install ffmpeg` · Windows `winget install Gyan.FFmpeg` · Debian/Ubuntu `sudo apt install ffmpeg` |
| yt-dlp | Extraction | **Optional.** If it isn't on PATH, the app downloads the official standalone build into `.tools/bin` on first start (SHA-256 verified). To manage it yourself: `pip install -U yt-dlp yt-dlp-ejs` |
| ffmpeg | Merging and MP3 | **Optional.** If it isn't on PATH, the `ffmpeg-static` npm package installed with the app is used. |

Verify the tools:

```bash
yt-dlp --version
ffmpeg -version | head -1
node -v
```

**Start the app**

```bash
npm install
cp .env.example .env          # optional; defaults work
npm run dev                   # http://localhost:3000
```

Check the tools are visible to the app: `http://localhost:3000/api/health` should show `"ok": true`.

> The Postgres/Drizzle files from the starter template (`src/db`, `drizzle.config.json`) are not used by SaveClip. You can leave them or delete them. `DATABASE_URL` is not required.

---

## 2. Deploy to Render

### What you need

1. A **GitHub (or GitLab) repository** containing this project. Push it:
   ```bash
   git init && git add . && git commit -m "SaveClip"
   git branch -M main && git remote add origin git@github.com:<you>/saveclip.git
   git push -u origin main
   ```
2. A **Render account** with a payment method on file. Render's free tier is not suitable: free instances sleep when idle, so the first download after a pause fails or times out, and their memory is too small for ffmpeg merges.
3. Your **legal details** for the policy pages (see section 5).

### Option A — Blueprint (recommended)

1. Render dashboard → **New +** → **Blueprint** → select your repository → **Apply**.
2. Render reads `render.yaml`, builds the Dockerfile and creates the `saveclip` web service.
3. If you see a prompt for `YTDLP_PROXY`, leave it blank unless you need it (see section 3).
4. Wait for the build (3–6 minutes the first time). The service turns **Live** when `/api/health` passes.

### Option B — Manual web service

1. **New +** → **Web Service** → connect the repository.
2. **Runtime:** `Docker`. **Dockerfile path:** `./Dockerfile`. **Docker context:** `.`
3. **Instance type:** `Starter` minimum. Use `Standard` if HD merges hit out-of-memory restarts.
4. **Health check path:** `/api/health`.
5. **Environment:** add the variables from the table in section 4. Set `YTDLP_JS_RUNTIME=node`.
6. **Instances:** `1` (see the scaling note below).
7. **Create Web Service**.

### After deploying: smoke test

```bash
curl https://<your-service>.onrender.com/api/health
# → {"ok":true,"tools":{"ytdlp":"2026.xx.xx","ffmpeg":"...","node":"v22..."}}
```

Then open the site, paste a public YouTube or TikTok link, and download each format.

### Keeping yt-dlp current

Platforms change their sites often, and yt-dlp releases fixes almost weekly. The Docker image installs the latest yt-dlp at build time. To update:

- Render dashboard → **Manual Deploy** → **Clear build cache & deploy**.
- Do this whenever downloads start failing across the board. Consider doing it monthly.

### Scaling note

Jobs live in the memory of one Node process. **Keep the instance count at 1.** If you scale out, a job created on one instance can't be polled or downloaded from another. To scale horizontally, move the job store (and the temporary files) to Redis plus shared object storage such as S3 or Cloudflare R2. That is a larger change, so it is not in this version.

---

## 3. YouTube: what actually makes it work

YouTube is the platform most likely to need attention in production.

**Why it can fail on a server.** YouTube often blocks requests from cloud data-centre IP ranges (including Render's) and shows *"Sign in to confirm you're not a bot"*. The app reports that as `YOUTUBE_BOT_CHECK` (HTTP 503) with a friendly message. Two levers help:

| Lever | When to use | How |
|---|---|---|
| **Cookies file** | Bot checks persist | Export cookies for youtube.com from a browser where you are logged into a **throwaway** Google account. Use an extension such as "Get cookies.txt LOCALLY". On Render: **Environment → Secret Files → add `youtube-cookies.txt`** with the file contents. The app reads `/etc/secrets/youtube-cookies.txt` automatically (`YTDLP_COOKIES_FILE` is already set in `render.yaml`). |
| **Proxy** | The cookies don't help, or you want a residential IP | Set `YTDLP_PROXY=http://user:pass@host:port` |

Cautions:

- **Cookies belong to a real account.** Use a dedicated account, never your main one. Account bans and session expiry are common. Treat the file as a secret, and never commit it (`.gitignore` already excludes `*cookies*.txt`).
- **Platform terms.** YouTube's Terms of Service prohibit downloading content except where it offers a download feature or the rights holder grants permission. Using cookies or proxies to get around bot checks may also breach its terms. Get legal advice before enabling YouTube for the public. The Terms page already tells users they are responsible for complying with platform terms.
- **Keep yt-dlp current.** YouTube changes its player frequently, and old yt-dlp builds fail. See "Keeping yt-dlp current".

For the other platforms, public posts usually work without cookies. Instagram and some Facebook and Snapchat content often require a login, and those requests return `PRIVATE`.

---

## 4. Environment variables

| Variable | Default | Description |
|---|---|---|
| `YTDLP_PATH` | `yt-dlp` | yt-dlp executable (in the Docker image it is on `PATH`). |
| `YTDLP_JS_RUNTIME` | `node` | JS runtime for YouTube challenges (`node`, `deno`, `bun`, or empty to disable). |
| `YTDLP_COOKIES_FILE` | _(empty)_ | Netscape cookies file. Ignored if the file doesn't exist. |
| `YTDLP_PROXY` | _(empty)_ | Proxy URL for yt-dlp. |
| `DOWNLOAD_DIR` | `<os tmp>/saveclip` | Temporary folder. Wiped on startup, so use a dedicated folder. |
| `RATE_LIMIT_PER_MINUTE` | `10` | Metadata lookups per IP per minute. Job creation allows 3× this. |
| `MAX_CONCURRENT_JOBS` | `2` | Simultaneous yt-dlp jobs. Extra requests get `503 BUSY`. |
| `MAX_DURATION_SECONDS` | `3600` | Longer videos are refused (`TOO_LONG`). |
| `MAX_FILESIZE_MB` | `1024` | Largest output yt-dlp may create. |
| `JOB_TTL_MINUTES` | `15` | Idle temporary files are deleted after this long. |
| `JOB_TIMEOUT_SECONDS` | `600` | Hard cap on one yt-dlp run. |
| `NODE_ENV` | `production` on Render | Standard Next.js setting. |
| `PORT` | `10000` in Docker | Render sets this automatically. |

Set them in Render under **Environment**, or in `render.yaml`. Never put cookies or proxy credentials in the repository.

---

## 5. Before you launch: legal checklist

The Terms, Privacy and DMCA pages are production-ready drafts. You must complete them:

- [ ] Replace every `[PLACEHOLDER]` (company name, address, emails, jurisdiction, venue, log retention period).
- [ ] Register a DMCA designated agent with the **U.S. Copyright Office** if you serve U.S. users.
- [ ] Have a lawyer review the Terms, especially the YouTube and platform-terms wording.
- [ ] Confirm the Privacy Policy matches what you run. Temporary files are kept up to `JOB_TTL_MINUTES`. Hosting logs are kept by Render.
- [ ] Set up a monitored inbox for takedown notices, and act on them promptly.
- [ ] Decide whether to keep YouTube enabled, given the risk described in section 3.

---

## 6. API reference

All responses are JSON except the file stream. Errors use `{ "ok": false, "code": "...", "message": "..." }`.

| Method & path | Body / params | Success | Notes |
|---|---|---|---|
| `POST /api/download` | `{ "url": "https://..." }` | `200 { ok, result: { platform, title, thumbnail, duration, author, options[] } }` | Metadata only. |
| `POST /api/jobs` | `{ "url": "...", "format": "mp4-hd" \| "mp4-sd" \| "mp3" }` | `202 { ok, jobId, status, progress, ... }` | Starts a download. |
| `GET /api/jobs/:id` | — | `200 { ok, status, progress, filename, size, error }` | Poll every ~1s. |
| `GET /api/jobs/:id/file` | — | `200` file stream with `Content-Disposition: attachment` | Only when `status = ready`. Deleted after delivery. |
| `GET /api/health` | — | `200 { ok, tools: { ytdlp, ffmpeg, node } }` | Used by Render. |

### Error codes

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `INVALID_URL`, `BAD_JSON`, `INVALID_FORMAT` | Malformed input. |
| 422 | `UNSUPPORTED_DOMAIN`, `UNSUPPORTED_URL`, `LIVE` | Not a supported platform, not a video page, or a live stream. |
| 403 | `PRIVATE` | Private, members-only, age-restricted or login-required content. |
| 404 | `NOT_FOUND`, `JOB_NOT_FOUND`, `FILE_NOT_FOUND` | Video removed, or the job/file has expired. |
| 413 | `TOO_LONG` | Longer than `MAX_DURATION_SECONDS`. |
| 429 | `RATE_LIMITED` | Too many requests from this IP (`Retry-After: 60`). |
| 451 | `GEO_BLOCKED` | Not available in the server's region. |
| 503 | `YOUTUBE_BOT_CHECK`, `UPSTREAM_LIMIT`, `BUSY`, `TOOL_MISSING` | Temporary; retry later. `TOOL_MISSING` means yt-dlp isn't installed. |
| 504 | `TIMEOUT` | The platform or download took too long. |
| 500 | `SERVER_MISCONFIGURED` | ffmpeg is missing on the server. |
| 502 | `EXTRACTION_FAILED`, `NO_OUTPUT` | Generic failure. Server logs contain the yt-dlp detail. |

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `/api/health` shows `"ok": false` | yt-dlp or ffmpeg missing | Local: install them (section 1). Render: check the Docker build log. |
| Every download fails after a while | yt-dlp is outdated | Clear build cache & deploy on Render. |
| YouTube: "blocked because it looks automated" | Render's IP is flagged | Add cookies (section 3) or a proxy. |
| Download button stuck at "Merging & converting…" | Large HD file on a small instance | Upgrade the instance (more CPU/RAM). |
| Random `503 BUSY` | More than `MAX_CONCURRENT_JOBS` jobs running | Raise the limit only if memory allows. |
| `Out of memory` restarts | ffmpeg merging on a small plan | Use Standard, or lower `MAX_FILESIZE_MB`. |
| Download works locally but not on Render | Platform blocks datacenter IPs | Use a proxy, and expect some failures on Instagram, Facebook and Snapchat. |

View logs: Render dashboard → your service → **Logs**. yt-dlp error details appear there prefixed with `[job …]` or `[yt-dlp metadata]`.

---

## 8. Security notes

- URLs are validated with Zod and checked against an allow-list of hosts. Only `http(s)` is accepted.
- yt-dlp is started with `execFile`/`spawn` and an argument array. No shell is involved, and the URL is placed after `--` so it cannot be read as an option.
- Client responses never include filesystem paths or the source URL.
- Downloads run in per-job folders with `0700` permissions. Files are deleted after delivery or after the TTL.
- Rate limiting is per IP using the last `X-Forwarded-For` entry, which Render sets from the connecting client. Limits are per instance (see the scaling note).
- Security headers beyond the defaults are not configured. Add a CSP in `next.config.ts` before a public launch if you need one.

## Licence and attribution

yt-dlp is released under the Unlicense, ffmpeg under LGPL/GPL. This project does not host any media, and it is not affiliated with TikTok, X, Meta, Snap Inc. or YouTube.
"# Saveclips" 
