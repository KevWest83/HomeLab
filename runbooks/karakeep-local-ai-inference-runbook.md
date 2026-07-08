# Karakeep — Local AI Inference Setup (Tagging, Summarization, Screenshots)

## Overview

Karakeep (bookmark manager) is wired to a local Ollama instance for AI-powered
tagging and summarization, with a headless Chrome sidecar added for real page
screenshots. Everything runs local-only — no cloud AI, no external API calls
for inference.

**Stack role reference** (see repo networking docs for actual host IPs):
- Karakeep + Meilisearch + headless Chrome: Docker Compose, on the primary
  AI/NAS server
- Ollama: bare-metal (systemd) on the same host, not containerized

---

## Why these choices (read before changing anything)

- **Text tagging/summarization model: `phi4:14b`.** Chosen over Qwen3.5/3.6,
  Mistral Nemo/Small, and GPT-OSS-20B after comparing structured-output/
  value-accuracy benchmarks — Phi-4 14B specifically scored ahead of GPT-5
  and GPT-5-Mini on a JSON value-accuracy benchmark, which is the closest
  public proxy to "does it tag correctly," not just "does it follow
  instructions in general."
- **Image/OCR model: intentionally unset.** The user doesn't bookmark
  images, so `INFERENCE_IMAGE_MODEL` is left unconfigured. Karakeep falls
  back to its default non-AI OCR (tesseract.js) for the rare edge case.
  The image tagging *prompt* is staged and ready (see below) for whenever
  this gets turned on later — don't forget it exists.
- **Connection method: OpenAI-compatible endpoint (`OPENAI_BASE_URL`),
  not native `OLLAMA_BASE_URL`.** This is Karakeep's currently recommended
  integration path — it handles message formatting automatically and is
  more reliable across different models than the native endpoint.
- **Curated Tags: locked down (not freeform).** The AI can only select from
  an existing, cleaned-up tag list rather than inventing new tags per
  bookmark. This trades "tags can drift/sprawl over time" for "tagging is
  constrained to a taxonomy that was deliberately curated." If a bookmark's
  actual topic doesn't fit anything in the curated list, it may come through
  under-tagged — that's expected behavior, not a bug. Expand the curated
  list, don't fight it with prompt engineering.
- **Headless Chrome screenshot sidecar was added deliberately, not by
  default.** The original deployment intentionally ran with no
  `BROWSER_WEB_URL` set (HTTP-only crawling, no JS execution, no
  screenshots) — this was a Phase 1 decision. It was revisited because
  thumbnail quality was poor for JS-heavy/social pages. See "Known
  Limitations" below for what this does and doesn't fix, and the privacy
  trade-off section for what running a browser against arbitrary bookmarked
  URLs actually costs.
- **Cookie-based crawler authentication was considered and explicitly
  rejected.** Login-walled pages (YouTube Subscriptions, Reddit home feed,
  TikTok feed, etc.) will still render blank thumbnails. Loading personal
  session cookies into the automated headless browser was ruled out —
  platforms like Instagram/TikTok/Reddit actively fingerprint automated
  sessions even with valid cookies, risking account flags, plus ongoing
  cookie-refresh maintenance. Thumbnail completeness was judged not worth
  that risk. Revisit only if this becomes a real priority later, and treat
  it as a deliberate risk decision, not a default setting.

---

## Current `docker-compose.yml`

```yaml
services:
  karakeep:
    image: ghcr.io/karakeep-app/karakeep:${KARAKEEP_VERSION:-release}
    container_name: karakeep
    restart: unless-stopped
    ports:
      - "3000:3000"
    env_file: [.env]
    environment:
      MEILI_ADDR: http://meilisearch:7700
      DATA_DIR: /data
      DB_WAL_MODE: "true"
      BROWSER_WEB_URL: http://chrome:9222
      CRAWLER_STORE_SCREENSHOT: "true"
      CRAWLER_FULL_PAGE_SCREENSHOT: "true"
      CRAWLER_DOWNLOAD_BANNER_IMAGE: "true"
      CRAWLER_NAVIGATE_TIMEOUT_SEC: "60"
      CRAWLER_SCREENSHOT_TIMEOUT_SEC: "30"
      OPENAI_API_KEY: ollama
      OPENAI_BASE_URL: http://<OLLAMA_HOST_IP>:11434/v1
      INFERENCE_TEXT_MODEL: phi4:14b
      INFERENCE_CONTEXT_LENGTH: "8192"
      # INFERENCE_IMAGE_MODEL intentionally unset — see "Why these choices"
      # INFERENCE_LANG intentionally unset — cosmetic gap only, see Known Limitations
    volumes:
      - ${KARAKEEP_DATA_DIR}:/data
    depends_on:
      - meilisearch
      - chrome
    networks:
      - default
      - proxy

  chrome:
    image: gcr.io/zenika-hub/alpine-chrome:123
    container_name: karakeep-chrome
    restart: unless-stopped
    command:
      - --no-sandbox
      - --disable-gpu
      - --disable-dev-shm-usage
      - --remote-debugging-address=0.0.0.0
      - --remote-debugging-port=9222
      - --hide-scrollbars
    networks:
      - default

  meilisearch:
    image: getmeili/meilisearch:v1.13.3
    container_name: karakeep-meilisearch
    restart: unless-stopped
    env_file: [.env]
    environment:
      MEILI_NO_ANALYTICS: "true"
      MEILI_MASTER_KEY: ${MEILI_MASTER_KEY}
    volumes:
      - ${MEILI_DATA_DIR}:/meili_data
    networks:
      - default

networks:
  proxy:
    external: true
```

> **Before pushing to the public repo:** replace `<OLLAMA_HOST_IP>` with the
> real IP locally, but keep the placeholder in the committed version —
> consistent with the "no internal IPs/hostnames in public files" rule
> already established for this repo.

### `chrome` service flags, explained

- `--no-sandbox` — disables Chromium's internal renderer sandboxing.
  Standard/expected for headless Chrome in Docker (proper sandboxing needs
  extra kernel privileges that widen the attack surface more than this
  does). Real trade-off, not zero-risk — mitigated by the container having
  no volume mounts, no host network mode, no device access. Blast radius if
  compromised is contained to this one disposable container.
- `--disable-gpu` — software rendering only. This container has no GPU
  device passthrough, so this just prevents Chromium from trying (and
  failing) to use hardware acceleration it doesn't have. **Does not touch
  the host GPU or compete with Ollama for VRAM** — scope is limited to this
  container's own rendering.
- `--hide-scrollbars` — prevents scrollbar artifacts from being baked into
  screenshots. Karakeep's own release notes flag this as needed once
  screenshots are turned on.
- Remote debugging port (9222) is bound to `0.0.0.0` **inside the container
  only** — no `ports:` mapping publishes it to the host or LAN. Currently
  only reachable by other containers on the `default` Docker network
  (Karakeep). CDP (what that port exposes) has no built-in authentication —
  anything that can reach it has full control of the browser. **Flagged for
  revisit:** if anything else ever joins the `default` network, consider
  splitting `chrome` onto a network shared only with `karakeep`.

---

## AI Settings (in-app configuration)

### Tag Style
Set to **Title case with spaces** (e.g. `Machine Learning`). This drives tag
formatting automatically — don't duplicate casing instructions in custom
prompts, it's redundant and wastes the character budget.

### Curated Tags
Populated and locked (AI can only select from this list, not invent new
tags). Keep this list current as bookmark topics evolve — under-tagged
bookmarks are a sign the list needs expanding, not that the prompt is wrong.

### Custom Prompts (Tagging Rules)

These are **appended** to Karakeep's built-in base prompts, not full
replacements. Character limit is roughly 500 per entry — keep additions
tight and specific.

**Text prompt** (scope: Text):
```
Select only tags from the list that specifically match this content's actual subject matter. Do not select a tag just to hit a target count — fewer accurate tags is better than padding with loosely related ones. When both a specific and a broad tag apply, prefer the specific one.
```

**Image prompt** (scope: Image) — *staged, inactive until `INFERENCE_IMAGE_MODEL` is set*:
```
Select only tags from the list that specifically match what's shown in the image. Do not select a tag just to hit a target count — fewer accurate tags is better than padding with loosely related ones. When both a specific and a broad tag apply, prefer the specific one.
```
Note: Karakeep's built-in base image prompt targets 10-15 tags, while the
text side targets 3-5. This custom instruction pulls against that default
aim toward "if the list doesn't support it, don't force it" — expected to
win since custom rules override default tendencies, but worth knowing if
image tagging output looks inconsistent whenever this gets activated.

**Summarization prompt** (scope: Summary):
```
Focus on the practical takeaway or key technical detail rather than restating the title or intro. Skip any marketing or promotional language from the source. If the content is a tutorial or how-to, state what it teaches, not how it's structured.
```
Note: written as a general-purpose prompt since bookmarks span technical and
non-technical content. Expect summaries on non-technical/social bookmarks to
occasionally be generic or off — accepted trade-off rather than maintaining
per-category prompts.

---

## Known Bug + Fix: Ollama model tag mismatch (404 on all inference jobs)

**Symptom:** Bulk "Regenerate AI Tags/Summaries for All Bookmarks" fails
almost entirely (e.g. 244 failed / 0 succeeded in Inference Jobs). Karakeep's
own logs show nothing useful — just routine polling. The real error is in
Ollama's log:

```
journalctl -u ollama -n 100 --no-pager
```
```
[GIN] ... | 404 | ... | POST "/v1/chat/completions"
```

Sub-2ms response times on every request — Ollama is rejecting instantly, not
timing out or struggling under load.

**Root cause:** A 404 on `/v1/chat/completions` from Ollama specifically
means *the requested model tag doesn't exist locally* — Ollama's OpenAI-
compatible endpoint returns 404 (not a clearer error) when the model name
in the request isn't found in `ollama list`. This is confirmed Ollama
behavior, not a routing or proxy problem.

In this case: the compose file specified `INFERENCE_TEXT_MODEL: phi4:14b`,
but `ollama list` only had `phi4:latest` pulled. Ollama tags are literal
strings — it does not know `latest` and `14b` refer to the same underlying
model unless both tags actually exist in the local store.

**Diagnostic steps, in order:**
```bash
# 1. Confirm Ollama itself is reachable and check version
ollama --version

# 2. Check what Karakeep is actually configured to request
docker exec karakeep env | grep -i OLLAMA

# 3. Check what's actually pulled locally
ollama list

# 4. Compare the exact tag string in the compose file against ollama list output
```

**Fix — retag rather than re-download:**
```bash
ollama cp phi4:latest phi4:14b
docker restart karakeep
```
This is a local metadata copy (instant, no re-download of the ~9GB model).
Preferred over just changing the compose file to `phi4:latest`, because
floating `:latest` tags can silently point to a different model after a
future pull — pinning to an explicit tag (`phi4:14b`) is self-documenting
and consistent with how `KARAKEEP_VERSION` is already pinned via env var in
this same compose file, rather than tracking `:latest` there too.

**Verify the fix:**
```bash
journalctl -u ollama -n 20 --no-pager
```
Should show `200` instead of `404` on `/v1/chat/completions` lines after
triggering a new inference job.

---

## Known Limitations

- **Login-walled pages will not get real thumbnails.** YouTube
  Subscriptions, personalized Reddit feeds, TikTok feeds, and similar pages
  require an authenticated session the crawler doesn't have. The headless
  Chrome addition fixes public JS-rendered pages; it does not and cannot fix
  pages that need you to be logged in. This was a deliberate scope decision
  (see cookie-auth rejection above), not an oversight.
- **`og:image`-sourced thumbnails will vary in quality per-site.** Some
  sites declare minimal/generic preview images (e.g. just a square logo) —
  that's the site's own metadata, not a crawler defect.
- **`INFERENCE_LANG` is unset.** The Prompt Preview UI shows a cosmetic gap
  ("tags must be in .") as a result. Harmless — Phi-4 defaults to the
  content's own language — but could be set to `en` to clean up the preview
  display if desired.

---

## Backup Checklist

For a complete backup of this stack, capture:
- `docker-compose.yml` (this file)
- `.env` (contains `MEILI_MASTER_KEY` and other secrets — **do not commit
  to the public repo**)
- `${KARAKEEP_DATA_DIR}` bind-mount host path (bookmark data, assets)
- `${MEILI_DATA_DIR}` bind-mount host path (search index — technically
  rebuildable via Reindex All Bookmarks, but faster to restore from backup)
- No named Docker volumes in this stack — everything is bind-mounted, so
  host-path backups are sufficient; no `docker volume` export needed.

---

## Revisit Items (flagged, not urgent)

- [ ] CDP port (9222) on `chrome` service has no network isolation from
      other future containers on `default` network — consider a dedicated
      internal network shared only with `karakeep` if the stack grows.
- [ ] `INFERENCE_LANG` unset — cosmetic only, set to `en` if the blank
      Prompt Preview display becomes annoying.
- [ ] Image tagging (`INFERENCE_IMAGE_MODEL`) — prompt is staged and ready;
      model selection was deliberately deferred since the user doesn't
      bookmark images. Revisit if that changes.
- [ ] Cookie-based crawler authentication — explicitly rejected due to
      platform fingerprinting/account-flagging risk on Instagram/TikTok/
      Reddit. Do not implement without re-evaluating that risk first.
