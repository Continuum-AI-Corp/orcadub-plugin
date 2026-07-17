<div align="center">

<img src="assets/orcadub.png" alt="OrcaDub" width="340">

### Translate any video. Keep the voice.

**Claude Code plugin for [OrcaDub](https://orcadub.orcarouter.ai) — AI video dubbing.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

[Website](https://orcadub.orcarouter.ai) · [MCP server](https://github.com/Continuum-AI-Corp/orcadub-mcp-server) · [Get an API key](https://www.orcarouter.ai/console)

</div>

---

This plugin bundles the [`@orcadub/mcp`](https://www.npmjs.com/package/@orcadub/mcp)
server **and** a `dub` skill, so Claude Code (or any host that supports Claude
Code plugins) can dub a video into another language conversationally: upload a
file or pass a URL, submit a job to the `orca/dub` model through the
[OrcaRouter](https://www.orcarouter.ai) gateway, poll progress, and download
the finished MP4.

## Install

```
/plugin marketplace add Continuum-AI-Corp/orcadub-plugin
/plugin install orcadub@orcadub
```

On install you'll be prompted for your **OrcaRouter API key** (`sk-orca-...`) —
create one at [the OrcaRouter console](https://www.orcarouter.ai/console) (token
management page). The key is stored by Claude Code and passed to the MCP server
as `ORCADUB_API_KEY`; dubbing jobs are billed per minute of source video.

## Use

Just ask:

> Dub this into Japanese: https://www.youtube.com/watch?v=…

The bundled `dub` skill drives the whole flow — it confirms the required
details (source/target language, the source file or URL, a title), submits the
job, polls until it's done, then asks whether to save the MP4 locally.

## What's inside

| Component | What it does |
|---|---|
| MCP server `@orcadub/mcp` | 5 tools: `dub_health`, `dub_upload`, `dub_create`, `dub_get`, `dub_download` |
| Skill `dub` | Teaches the agent when to dub and how to run the upload → create → poll → download flow (asks before billing, confirms before downloading) |

## Without the plugin

Prefer to wire the MCP server directly (Cursor, Windsurf, Codex, plain Claude
Code)? See [orcadub-mcp-server](https://github.com/Continuum-AI-Corp/orcadub-mcp-server)
for per-host config and `npx -y @orcadub/mcp` usage.

## License

[MIT](LICENSE)
