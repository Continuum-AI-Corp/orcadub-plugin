---
name: dub
description: Dub a video into another language with the OrcaDub service (OrcaRouter model orca/dub) via the orcadub MCP tools (dub_upload / dub_create / dub_get / dub_download). Use whenever the user wants to 配音/dub/translate a video's voice track into another language, submit a dubbing job, check dubbing job status, or fetch a dubbed result — e.g. "把这个视频配成英文", "dub this YouTube video into Japanese", "配音任务跑完了吗", "下载配音结果". Requires an OrcaRouter API key (the plugin prompts for it on install).
---

# OrcaDub dubbing workflow

OrcaDub translates + re-voices a video (ASR → translate → TTS voice clone →
align → mux). All requests go through the OrcaRouter gateway, which routes
them to the `orca/dub` model — the MCP server attaches the routing field
automatically. Jobs are asynchronous: create, then poll.

## Prerequisites — OrcaRouter authorization

- These tools require an OrcaRouter API key (`sk-orca-...`). The plugin
  prompts for it on install; it is passed to the server as
  `ORCADUB_API_KEY`. Get a key at https://www.orcarouter.ai/console
  (token management page).
- While the key is missing/invalid, every `dub_*` call returns a
  "not authorized" error carrying the sign-up link — when that happens,
  give the user the link and have them set the key (re-run
  `/plugin` config or set `ORCADUB_API_KEY`), then retry.
- If the `dub_*` tools are missing entirely, the plugin/MCP server isn't
  connected — have the user check `/plugin` and restart the session.

## Required parameters — ASK, never guess

`dub_create` submits a **billed** job (per-minute pricing). Before calling
it, every REQUIRED parameter must come from the user; if any is missing,
ASK (in Claude Code use the question tool) instead of assuming:

1. **source_lang** — the video's spoken language.
2. **target_lang** — the language to dub into (never "auto").
3. **Source video** — exactly one of a local file (→ `dub_upload` first,
   then use the returned `file_id`) or a remote `url`.
4. **video_name** — REQUIRED when using `file_id` (ask for a title);
   optional for `url`.

Optional knobs (profile, glossary, translation_style, preserve_bgm,
lipsync, resolution/ratio, remove_watermark, …) are all exposed on
`dub_create` — set them only when the user asked for the behaviour;
deploy defaults are tuned. `remove_watermark` costs extra — confirm
explicitly before enabling.

Language codes: en zh ja ko fr de es pt ru ar it hi tr th vi id bn pl nl
uk fil el cs sv da no fi sk.

## Standard flow

1. **Sanity**: `dub_health` once per session — probes the gateway →
   orca/dub route end to end without creating a job.
2. **Source**: local file → `dub_upload {path}` → note the returned `id`
   (file_id; up to 8 GiB, chunked automatically). Remote URL → skip upload.
3. **Create**: `dub_create {source_lang, target_lang, file_id|url,
   video_name, ...}` after confirming the required params above.
4. **Poll**: `dub_get {video_id}` — status queued → in_progress →
   completed | failed, with integer `progress`. Poll every ~30s for short
   clips, every 1-2 min for long videos; back off if progress hasn't moved.
   Only ids returned by `dub_create` are queryable.
5. **Deliver — confirm, then download locally**: when status reaches
   completed, ASK the user whether to download the finished mp4, offering:
   - **current directory (DEFAULT)** — dest = working directory, filename
     `<video_name>-<target_lang>.mp4` (sanitize spaces; if it exists,
     append -1, -2, …);
   - **a custom path** — use exactly what the user gives;
   - **link only** — skip the download.
   On confirmation call `dub_download {video_id, dest}` (not billed) and
   report the saved path. In every case also hand over `content_url` from
   dub_get as the re-download address — Bearer-authenticated, never expires
   (`curl -H "Authorization: Bearer sk-orca-..." <content_url> -o out.mp4`).
   The response's `job_id` is the underlying dub job uuid (for support).

## Debugging

- `dub_health` errors name the broken hop: network → gateway unreachable;
  "Model name not specified" → routing misconfigured; 401 → invalid/expired
  key (re-issue on the OrcaRouter console).
- 402 insufficient_credit → top up on OrcaRouter (per-minute billing);
  429 free_quota_exceeded → free-tier limit hit.
- `task_not_exist` on dub_get → the id wasn't created here (or a typo);
  use the id exactly as returned by dub_create.
- Job listing / cancel / delete aren't exposed — manage those in the
  OrcaDub web console.

## Example (URL → Chinese dub)

```
dub_create {"source_lang":"en","target_lang":"zh",
            "url":"https://www.youtube.com/watch?v=...",
            "profile":"lecture","glossary":{"OrcaRT":"鲸鸣实时"}}
→ {"id":"<job>","status":"queued"}
→ dub_get {"video_id":"<job>"} until completed
→ confirm download → dub_download {"video_id":"<job>","dest":"./out.mp4"}
```
