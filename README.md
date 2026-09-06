# web-tools-skills

The public skills + bridge daemon for **web-tools** — the shared, open base that
powers two Chrome extensions:

- **WebCLI** — headless, agent-free. Exposes your logged-in browser's primitive
  base (generic browser tools + `eval_js` + recon + site scripts) to external AI
  agents (Claude Code, Codex, …) over a local bridge you talk to with `curl`.
- **localmd Connect** — the same primitive base plus knowledge-base features, for
  the [localmd](https://localmd.app) app.

Both speak the same primitives, so the **adapter skills** here work with either.

## What's in here

```
server.mjs                     the bridge daemon (npx/node entry, web-tools-bridge bin)
skills/
  webcli/SKILL.md              driving guide: how a CLI agent drives WebCLI
  adapters/                    adapter skills — "a way to reach site X", as data
    README.md                  what they are + the robustness ladder for building one
    youtube-transcript/        e.g. a YouTube transcript, pot-free, via the UI panel
```

## Install the skills

```bash
# a CLI agent (WebCLI): install globally (auto-detects Claude Code / Cursor / Codex)
npx skills add whitefoxx/web-tools-skills -g

# localmd: install the adapter skills you want into your KB's .agents/skills/
```

Or just hand this repo URL to your agent and let it read the skill it needs.

## Run the bridge (WebCLI)

```bash
npx -y github:whitefoxx/web-tools-skills          # daemon on 127.0.0.1:9376
# custom port:  BRIDGE_PORT=8790 npx -y github:whitefoxx/web-tools-skills
```

The WebCLI extension dials the daemon automatically (default port **9376**).

## Drive it

```bash
curl -s http://127.0.0.1:9376/status                       # is the extension connected?
curl -s http://127.0.0.1:9376/tools                        # the tool catalog (source of truth)
curl -s http://127.0.0.1:9376/command \
  -d '{"tool":"generic__open_url","args":{"url":"https://example.com"}}'
```

Full driving guide: [`skills/webcli/SKILL.md`](./skills/webcli/SKILL.md). Building
an adapter skill: [`skills/adapters/README.md`](./skills/adapters/README.md).

## HTTP API (binds 127.0.0.1 only)

| Method + path   | Result                                                      |
| --------------- | ----------------------------------------------------------- |
| `GET /ping`     | `{ok:true}`                                                 |
| `GET /status`   | `{ok, connected, port, client, tools}`                      |
| `GET /tools`    | `{ok, tools:[…]}` — tools in OpenAI-tool shape              |
| `POST /command` | body `{tool, args}` → `{ok, result}` or `{ok:false, error}` |

## History

Renamed from `webcli-skills` (2026-09): the base grew beyond WebCLI's generic
tools into the shared **web-tools** primitive base (eval_js + recon + site
scripts), and adapter skills replace the old per-site marketplace. The full
Web Agent extension's own external-control daemon still lives in
[`web-agent-skills`](https://github.com/whitefoxx/web-agent-skills) for now.
