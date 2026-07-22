---
name: dub-video
description: Dub a video into another language with the OrcaDub service (OrcaRouter model orca/dub) via the `@orcadub/cli` command-line tool (`npx -y @orcadub/cli <subcommand>`). Use whenever the user wants to dub, translate, or re-voice a video into another language, submit a dubbing job, check a dubbing job's status, or fetch a dubbed result — e.g. "dub this video into English", "dub this YouTube video into Japanese", "is the dubbing job done?", "download the dubbed result". Requires an OrcaRouter API key in `ORCADUB_API_KEY` and Node (for `npx`) or the orcadub binary on PATH.
---

# OrcaDub dubbing workflow

OrcaDub translates + re-voices a video (ASR -> translate -> TTS voice clone
-> align -> mux). All requests go through the OrcaRouter gateway, which routes
them to the `orca/dub` model. Jobs are asynchronous: create, then poll.

## Prerequisites

This skill calls the **`@orcadub/cli`** command-line tool — one short-lived
process per operation, no resident server. Invoke it as
`npx -y @orcadub/cli <subcommand>` (needs Node) or, if a prebuilt binary is
on PATH, `orcadub <subcommand>`. Auth comes from the `ORCADUB_API_KEY`
environment variable (an OrcaRouter `sk-orca-...` key from
https://www.orcarouter.ai/console). If it is unset/invalid, every command
exits non-zero with a "not authorized" message and the sign-up link on
stderr — have the user `export ORCADUB_API_KEY=sk-orca-...`, then retry.
If `npx`/`node` itself is missing, install Node or use the prebuilt binary.

Every command prints JSON on stdout on success; a non-zero exit + a message
on stderr is the error path.

## Required parameters — ASK, never guess

The `create` subcommand submits a **billed** job (per minute of source
video). Before calling it, every REQUIRED parameter must come from the user;
if any is missing, ASK (in Claude Code use the question tool) instead of
assuming:

| Parameter | Notes |
|---|---|
| `--source-lang` | The video's spoken language (code below). |
| `--target-lang` | Language to dub into (code below; never `auto`). |
| `--file-id` **or** `--url` | Exactly one. `--file-id` comes from `upload`; `--url` is a remote video (YouTube etc.), fetched server-side. |
| `--video-name` | REQUIRED when using `--file-id` (ask for a title); optional for `--url`. |

Language codes: `en zh ja ko fr de es pt ru ar it hi tr th vi id bn pl nl uk
fil el cs sv da no fi sk`.

## Options (the toggles on the OrcaDub site)

Optional booleans, off by default unless the deploy sets otherwise. Set one
only when the user asks for that behaviour; the labels match the toggles on
https://orcadub.orcarouter.ai. Pass them to `create` as repeatable
`--opt <parameter>=<value>` flags, e.g. `--opt preserve_bgm=true`.

| Site label | Parameter | Meaning |
|---|---|---|
| Keep background audio | `preserve_bgm` | Keep music/SFX via source separation |
| Overlay watermark | `watermark` | Burn the watermark onto the output |
| Remove watermark / subtitles (paid) | `remove_watermark` | Paid add-on that erases source logo/subtitles — confirm explicitly, it costs extra |
| Loudness matching | `loudness_enabled` | EBU R128 loudness-match gate on the final mux |
| Translate songs | `song_translation` | Dub sung segments instead of passing the original audio through |

Additional advanced `--opt` keys exist (set them only on the user's explicit
request): `adapt_idioms`, `comet_enabled`, `bed_level_match`, `bed_duck`,
`align_per_word`, `lipsync`, `lipsync_visemes`, `lipsync_identity_guard`,
`compact_output`, `voice_clone_consent` (bools); `profile`,
`translation_style`, `tts_backend`, `project_id`, `resolution`, `ratio`,
`bed_reverb_preset` (strings); `glossary.<term>=<rendering>` and
`speaker_assignments.<label>=<charid>` (maps).

## Standard flow

1. **Sanity**: `npx -y @orcadub/cli health` once per session — probes the
   gateway -> orca/dub route without creating a job.
2. **Source**: local file -> `npx -y @orcadub/cli upload --path <path>`
   (optionally `--purpose <p>`) -> note the `id` field in the returned JSON
   (file_id; up to 8 GiB, chunked automatically). Remote URL -> skip upload.
3. **Create**: after confirming the required params above,
   `npx -y @orcadub/cli create --source-lang <c> --target-lang <c>
   (--url <u> | --file-id <id> --video-name <name>) [--opt key=val ...]`.
4. **Poll**: `npx -y @orcadub/cli get --video-id <id>` — status queued ->
   in_progress -> completed | failed, with integer `progress`. Poll every
   ~30s for short clips, every 1-2 min for long videos; back off if progress
   hasn't moved. Only ids returned by `create` are queryable.
5. **Deliver — confirm, then download locally**: when status is completed,
   ASK whether to download the finished mp4, offering:
   - **current directory (DEFAULT)** — dest = working directory, filename
     `<video_name>-<target_lang>.mp4` (sanitize spaces; if it exists,
     append -1, -2, …);
   - **a custom path** — use exactly what the user gives;
   - **link only** — skip the download.
   On confirmation call `npx -y @orcadub/cli download --video-id <id>
   --dest <path>` (not billed) and report the saved path. In every case also
   hand over `content_url` from `get` as the re-download address —
   Bearer-authenticated, never expires
   (`curl -H "Authorization: Bearer sk-orca-..." <content_url> -o out.mp4`).
   The response's `job_id` is the underlying dub job uuid (for support).

## Debugging

- `health` errors name the broken hop: network -> gateway unreachable;
  "Model name not specified" -> routing misconfigured; 401 -> invalid/expired
  key (re-issue on the OrcaRouter console).
- If the command fails to launch (command not found / npx cannot fetch the
  package), Node/npx is unavailable or the network blocked the download —
  install Node or drop in the prebuilt binary.
- 402 insufficient_credit -> top up on OrcaRouter (per-minute billing);
  429 free_quota_exceeded -> free-tier limit hit.
- `task_not_exist` on `get` -> the id wasn't created here (or a typo); use
  the id exactly as returned by `create`.
- Job listing / cancel / delete aren't exposed — manage those in the
  OrcaDub web console.

## Example (URL -> Japanese dub)

```
npx -y @orcadub/cli create --source-lang en --target-lang ja \
  --url "https://www.youtube.com/watch?v=..." --opt preserve_bgm=true
-> {"id":"<job>","status":"queued"}
npx -y @orcadub/cli get --video-id <job>   # until completed
npx -y @orcadub/cli download --video-id <job> --dest ./out.mp4
```
