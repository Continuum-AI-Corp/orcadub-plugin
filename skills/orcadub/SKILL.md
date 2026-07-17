---
name: orcadub
description: Dub a video into another language with the OrcaDub service (OrcaRouter model orca/dub) via the orcadub MCP tools (dub_upload / dub_create / dub_get / dub_download). Use whenever the user wants to dub, translate, or re-voice a video into another language, submit a dubbing job, check a dubbing job's status, or fetch a dubbed result — e.g. "dub this video into English", "dub this YouTube video into Japanese", "is the dubbing job done?", "download the dubbed result". Requires the orcadub MCP server with an OrcaRouter API key.
---

# OrcaDub dubbing workflow

OrcaDub translates + re-voices a video (ASR -> translate -> TTS voice clone
-> align -> mux). All requests go through the OrcaRouter gateway, which routes
them to the `orca/dub` model. Jobs are asynchronous: create, then poll.

## Prerequisites

- This skill drives the **orcadub MCP server** — any agent that can run MCP
  servers works (Claude Code, Claude Desktop, Codex, Cursor, …). The server
  is launched as `npx -y @orcadub/mcp` with the `ORCADUB_API_KEY` environment
  variable set to an OrcaRouter key (`sk-orca-...`, from
  https://www.orcarouter.ai/console). If you installed the OrcaDub Claude Code
  plugin, you were prompted for the key on install; other agents set it in
  their MCP config.
- While the key is missing/invalid, every `dub_*` call returns a
  "not authorized" error with the sign-up link — give the user the link,
  have them set the key, then retry.
- If the `dub_*` tools are missing entirely, the MCP server isn't connected.

## Required parameters — ASK, never guess

`dub_create` submits a **billed** job (per minute of source video). Before
calling it, every REQUIRED parameter must come from the user; if any is
missing, ASK (in Claude Code use the question tool) instead of assuming:

| Parameter | Notes |
|---|---|
| `source_lang` | The video's spoken language (code below). |
| `target_lang` | Language to dub into (code below; never `auto`). |
| `file_id` **or** `url` | Exactly one. `file_id` comes from `dub_upload`; `url` is a remote video (YouTube etc.), fetched server-side. |
| `video_name` | REQUIRED when using `file_id` (ask for a title); optional for `url`. |

Language codes: `en zh ja ko fr de es pt ru ar it hi tr th vi id bn pl nl uk
fil el cs sv da no fi sk`.

## Options (the toggles on the OrcaDub site)

Optional booleans, off by default unless the deploy sets otherwise. Set one
only when the user asks for that behaviour; the labels match the toggles on
https://orcadub.orcarouter.ai.

| Site label | Parameter | Meaning |
|---|---|---|
| Keep background audio | `preserve_bgm` | Keep music/SFX via source separation |
| Overlay watermark | `watermark` | Burn the watermark onto the output |
| Remove watermark / subtitles (paid) | `remove_watermark` | Paid add-on that erases source logo/subtitles — confirm explicitly, it costs extra |
| Loudness matching | `loudness_enabled` | EBU R128 loudness-match gate on the final mux |
| Translate songs | `song_translation` | Dub sung segments instead of passing the original audio through |

Additional advanced parameters exist in the MCP tool schema (e.g. glossary,
translation_style, content profile, resolution) — set them only on the user's
explicit request.

## Standard flow

1. **Sanity**: `dub_health` once per session — probes the gateway ->
   orca/dub route without creating a job.
2. **Source**: local file -> `dub_upload {path}` -> note the returned `id`
   (file_id; up to 8 GiB, chunked automatically). Remote URL -> skip upload.
3. **Create**: `dub_create {source_lang, target_lang, file_id|url,
   video_name, ...}` after confirming the required params above.
4. **Poll**: `dub_get {video_id}` — status queued -> in_progress ->
   completed | failed, with integer `progress`. Poll every ~30s for short
   clips, every 1-2 min for long videos; back off if progress hasn't moved.
   Only ids returned by `dub_create` are queryable.
5. **Deliver — confirm, then download locally**: when status is completed,
   ASK whether to download the finished mp4, offering:
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

- `dub_health` errors name the broken hop: network -> gateway unreachable;
  "Model name not specified" -> routing misconfigured; 401 -> invalid/expired
  key (re-issue on the OrcaRouter console).
- 402 insufficient_credit -> top up on OrcaRouter (per-minute billing);
  429 free_quota_exceeded -> free-tier limit hit.
- `task_not_exist` on dub_get -> the id wasn't created here (or a typo);
  use the id exactly as returned by dub_create.
- Job listing / cancel / delete aren't exposed — manage those in the
  OrcaDub web console.

## Example (URL -> Japanese dub)

```
dub_create {"source_lang":"en","target_lang":"ja",
            "url":"https://www.youtube.com/watch?v=...",
            "preserve_bgm":true}
-> {"id":"<job>","status":"queued"}
-> dub_get {"video_id":"<job>"} until completed
-> confirm download -> dub_download {"video_id":"<job>","dest":"./out.mp4"}
```
