<div align="center">

<img src="assets/orcadub.png" alt="OrcaDub" width="340">

### Translate any video. Keep the voice.

**AI video dubbing for your agent — the `dub-video` skill, driving the [`@orcadub/cli`](https://www.npmjs.com/package/@orcadub/cli) command-line tool.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[Website](https://orcadub.orcarouter.ai) · [CLI / server repo](https://github.com/Continuum-AI-Corp/orcadub-mcp-server) · [Get an API key](https://www.orcarouter.ai/console)

</div>

---

Give any agent the ability to dub a video into another language
conversationally: upload a file or pass a URL, submit a job to the `orca/dub`
model through the [OrcaRouter](https://www.orcarouter.ai) gateway, poll
progress, and download the finished MP4.

One tool, one skill:

- **CLI** [`@orcadub/cli`](https://www.npmjs.com/package/@orcadub/cli) — a
  one-shot command-line tool (`npx -y @orcadub/cli <subcommand>` or a
  prebuilt `orcadub` binary), no resident server required.
- **Skill `dub-video`** — a plain `SKILL.md` workflow guide for skill-capable
  agents (Claude Code/Desktop, Codex, …) that calls the CLI: when to dub, ask
  for required parameters before the billed submit, poll etiquette, confirm
  before downloading.

## Install

**Claude Code — one-click plugin (skill that calls the OrcaDub CLI):**

```
/plugin marketplace add Continuum-AI-Corp/orcadub-plugin
/plugin install orcadub@orcadub-skills
```

**Codex — one-click plugin (skill that calls the OrcaDub CLI):**

```bash
codex plugin marketplace add Continuum-AI-Corp/orcadub-plugin
codex plugin add orcadub@orcadub-skills
```

**Any agent — install the skill directly:**

```bash
# Copy the skill (install into an orcadub/ folder):
cp -r skills/dub-video ~/.claude/skills/orcadub   # Claude Code / Desktop
cp -r skills/dub-video ~/.codex/skills/orcadub    # Codex

# Make sure Node is available so the skill can run `npx -y @orcadub/cli`
# (or put a prebuilt `orcadub` binary on PATH), and export your API key:
export ORCADUB_API_KEY=sk-orca-...
```

You'll need an **OrcaRouter API key** (`sk-orca-...`) — create one at
[the OrcaRouter console](https://www.orcarouter.ai/console) (token management
page). Export it as `ORCADUB_API_KEY` so the CLI can authenticate; dubbing
jobs are billed per minute of source video.

## Use

Just ask:

> Dub this into Japanese: https://www.youtube.com/watch?v=…

The `dub-video` skill drives the whole flow — it confirms the required details
(source/target language, the source file or URL, a title), submits the job,
polls until it's done, then asks whether to save the MP4 locally.

## Dubbing options

`create` requires `--source-lang`, `--target-lang`, a source (`--file-id` from
`upload` **or** a remote `--url`), and `--video-name` (with `--file-id`). The
optional toggles map to the labels on the [OrcaDub site](https://orcadub.orcarouter.ai)
and are passed as repeatable `--opt <param>=true` flags:

| Site label | Parameter | Site label | Parameter |
|---|---|---|---|
| Keep background audio | `--opt preserve_bgm=true` | Loudness matching | `--opt loudness_enabled=true` |
| Overlay watermark | `--opt watermark=true` | Translate songs | `--opt song_translation=true` |
| Remove watermark / subtitles (paid) | `--opt remove_watermark=true` | | |

See [`skills/dub-video/SKILL.md`](skills/dub-video/SKILL.md) for details.

## What's inside

| Component | What it does |
|---|---|
| CLI `@orcadub/cli` | 5 subcommands: `health`, `upload`, `create`, `get`, `download` |
| Skill `dub-video` | Teaches the agent when to dub and how to run the upload → create → poll → download flow (asks before billing, confirms before downloading) |

## CLI / binary details

Prebuilt binaries, Docker usage and the full subcommand reference live in the
[orcadub-mcp-server](https://github.com/Continuum-AI-Corp/orcadub-mcp-server)
repository. The same binary also runs as an MCP server when invoked with no
subcommand, for hosts that prefer a resident MCP connection instead of the
one-shot CLI.

## License

[MIT](LICENSE)
