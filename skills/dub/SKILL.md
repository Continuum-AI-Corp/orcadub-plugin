---
name: dub
description: Dub a video into another language with the OrcaDub service (OrcaRouter model orca/dub) via the orcadub MCP tools (dub_upload / dub_create / dub_get / dub_download). Use whenever the user wants to dub, translate, or re-voice a video into another language, submit a dubbing job, check a dubbing job's status, or fetch a dubbed result — e.g. "dub this video into English", "dub this YouTube video into Japanese", "is the dubbing job done?", "download the dubbed result". Requires an OrcaRouter API key (the plugin prompts for it on install).
---

# OrcaDub dubbing workflow

OrcaDub translates + re-voices a video (ASR -> translate -> TTS voice clone
-> align -> mux). All requests go through the OrcaRouter gateway, which routes
them to the `orca/dub` model — the MCP server attaches the routing field
automatically. Jobs are asynchronous: create, then poll.

## Prerequisites — OrcaRouter authorization

- These tools require an OrcaRouter API key (`sk-orca-...`). The plugin
  prompts for it on install and passes it to the server as `ORCADUB_API_KEY`.
  Get a key at https://www.orcarouter.ai/console (token management page).
- While the key is missing/invalid, every `dub_*` call returns a
  "not authorized" error with the sign-up link — give the user the link,
  have them set the key, then retry.
- If the `dub_*` tools are missing entirely, the MCP server isn't connected —
  have the user check `/plugin` and restart the session.

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

## Optional parameters (mirror the OrcaDub site's options)

Set these only when the user asks for the behaviour — deploy defaults are
tuned. Booleans accept true/false. The "Site label" column matches the
toggle names on https://orcadub.orcarouter.ai.

**Content & translation**

| Parameter | Site label | Meaning |
|---|---|---|
| `profile` | Content preset | `movie` \| `podcast` \| `lecture` \| `music_video` \| `short_drama` \| `ad_creative` |
| `translation_style` | Translation style | `formal` \| `casual` \| `literary` \| `news` \| `drama` \| `humorous` \| `business` \| `cute` |
| `glossary` | Glossary | Pinned source->target term renderings, up to 64 entries |
| `adapt_idioms` | Localise idioms | Render idioms as natural target-language equivalents |
| `comet_enabled` | Translation quality gate | COMET machine-translation quality gate |
| `song_translation` | Translate songs | Dub sung segments instead of passing the original audio through |

**Voice & TTS**

| Parameter | Site label | Meaning |
|---|---|---|
| `tts_backend` | TTS backend | `qwen3` \| `higgs` |
| `project_id` | Project | Cross-job character-voice memory |
| `speaker_assignments` | Speaker assignments | ASR diarization label -> character id (requires `project_id`) |
| `voice_clone_consent` | Voice clone consent | Attest rights/consent to clone the source voices |

**Audio bed & mix**

| Parameter | Site label | Meaning |
|---|---|---|
| `preserve_bgm` | Keep background audio | Keep music/SFX via source separation |
| `bed_level_match` | Bed level match | Match bed loudness to broadcast level |
| `bed_duck` | Bed ducking | Duck the bed under dialog |
| `bed_reverb_preset` | Bed reverb | `none` \| `small_room` \| `hall` \| `outdoor` |
| `loudness_enabled` | Loudness matching | EBU R128 loudness-match gate on the final mux |

**Alignment & video output**

| Parameter | Site label | Meaning |
|---|---|---|
| `align_per_word` | Per-word alignment | Per-word forced-alignment atempo |
| `lipsync` | Lipsync | Enable lipsync |
| `lipsync_visemes` | Viseme-aware lipsync | Viseme-aware lipsync plan |
| `lipsync_identity_guard` | Face-identity guard | Post-lipsync face-identity guard |
| `watermark` | Overlay watermark | Burn the watermark |
| `remove_watermark` | Remove watermark / subtitles (paid) | Paid MPS add-on; confirm explicitly — it costs extra |
| `resolution` | Output resolution | `source` \| `720p` \| `1080p` \| `2k` |
| `ratio` | Output ratio | `source` \| `16:9` \| `9:16` \| `1:1` |
| `compact_output` | Compact output | Re-encode a smaller final mp4 |

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
            "profile":"lecture","preserve_bgm":true,
            "glossary":{"OrcaRouter":"OrcaRouter"}}
-> {"id":"<job>","status":"queued"}
-> dub_get {"video_id":"<job>"} until completed
-> confirm download -> dub_download {"video_id":"<job>","dest":"./out.mp4"}
```
