<div align="center">

<img src="assets/orcadub.png" alt="OrcaDub" width="340">

### Translate any video. Keep the voice.

**AI video dubbing for your agent — the `orcadub` skill + the [`@orcadub/mcp`](https://www.npmjs.com/package/@orcadub/mcp) server.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[Website](https://orcadub.orcarouter.ai) · [MCP server](https://github.com/Continuum-AI-Corp/orcadub-mcp-server) · [Get an API key](https://www.orcarouter.ai/console)

</div>

---

Give any agent the ability to dub a video into another language
conversationally: upload a file or pass a URL, submit a job to the `orca/dub`
model through the [OrcaRouter](https://www.orcarouter.ai) gateway, poll
progress, and download the finished MP4.

Two components, each portable on its own:

- **MCP server** [`@orcadub/mcp`](https://www.npmjs.com/package/@orcadub/mcp) —
  works in **any** MCP client (Claude Code, Claude Desktop, Codex, Cursor,
  Windsurf, …).
- **Skill `dub-video`** — a plain `SKILL.md` workflow guide for skill-capable
  agents (Claude Code/Desktop, Codex, …): when to dub, ask for required
  parameters before the billed submit, poll etiquette, confirm before
  downloading.

## Install

**Claude Code — one-click plugin (MCP + skill):**

```
/plugin marketplace add Continuum-AI-Corp/orcadub-plugin
/plugin install orcadub@orcadub-skills
```

**Codex — one-click plugin (MCP + skill):**

```bash
codex plugin marketplace add Continuum-AI-Corp/orcadub-plugin
codex plugin add orcadub@orcadub-skills
```

**Any other MCP client — server only, then optionally copy the skill:**

```bash
# Cursor / Windsurf / Claude Desktop etc.: add a stdio server with
#   command: npx   args: -y @orcadub/mcp   env: ORCADUB_API_KEY=sk-orca-...
# (per-host config files: see the MCP server repo linked below)

# Optional, for skill-capable agents (install into an orcadub/ folder):
cp -r skills/dub-video ~/.claude/skills/orcadub   # Claude Code / Desktop
cp -r skills/dub-video ~/.codex/skills/orcadub    # Codex
```

You'll need an **OrcaRouter API key** (`sk-orca-...`) — create one at
[the OrcaRouter console](https://www.orcarouter.ai/console) (token management
page). The plugin prompts for it on install and passes it to the MCP server as
`ORCADUB_API_KEY`; dubbing jobs are billed per minute of source video.

## Use

Just ask:

> Dub this into Japanese: https://www.youtube.com/watch?v=…

The `dub-video` skill drives the whole flow — it confirms the required details
(source/target language, the source file or URL, a title), submits the job,
polls until it's done, then asks whether to save the MP4 locally.

## Dubbing options

`dub_create` requires `source_lang`, `target_lang`, a source (`file_id` from
`dub_upload` **or** a remote `url`), and `video_name` (with `file_id`). The
optional toggles map to the labels on the [OrcaDub site](https://orcadub.orcarouter.ai):

| Site label | Parameter | Site label | Parameter |
|---|---|---|---|
| Keep background audio | `preserve_bgm` | Loudness matching | `loudness_enabled` |
| Overlay watermark | `watermark` | Translate songs | `song_translation` |
| Remove watermark / subtitles (paid) | `remove_watermark` | | |

See [`skills/dub-video/SKILL.md`](skills/dub-video/SKILL.md) for details.

## What's inside

| Component | What it does |
|---|---|
| MCP server `@orcadub/mcp` | 5 tools: `dub_health`, `dub_upload`, `dub_create`, `dub_get`, `dub_download` |
| Skill `dub-video` | Teaches the agent when to dub and how to run the upload → create → poll → download flow (asks before billing, confirms before downloading) |

## MCP server details

Per-host configuration examples (Cursor, Windsurf, Claude Desktop, Codex),
Docker usage, prebuilt binaries and the tool reference live in the
[orcadub-mcp-server](https://github.com/Continuum-AI-Corp/orcadub-mcp-server)
repository.

## License

[MIT](LICENSE)
