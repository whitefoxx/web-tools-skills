# web-tools-skills — moved to [`whitefoxx/web-tools`](https://github.com/whitefoxx/web-tools)

> **This repository is archived (2026-09-07) and no longer updated.**
> Everything in it now lives in **[whitefoxx/web-tools](https://github.com/whitefoxx/web-tools)**,
> together with the full source of the two Chrome extensions it drives — which is
> now open source as well.

## Update your commands

| Old (here)                                     | New                                     |
| ---------------------------------------------- | --------------------------------------- |
| `npx skills add whitefoxx/web-tools-skills -g` | `npx skills add whitefoxx/web-tools -g` |
| `npx -y github:whitefoxx/web-tools-skills`     | `npx -y github:whitefoxx/web-tools`     |

The daemon still binds `127.0.0.1`, still defaults to port **9376**, and the HTTP
API (`GET /ping /status /tools`, `POST /command`) is unchanged. Only the package
moved.

## Where things went

| Was here                    | Now in `web-tools`                          |
| --------------------------- | ------------------------------------------- |
| `server.mjs`                | `bridge/server.mjs`                         |
| `skills/webcli/SKILL.md`    | `skills/webcli/SKILL.md`                    |
| `skills/adapters/README.md` | `skills/adapters/README.md`                 |
| —                           | `skills/web-agent/SKILL.md` (newly public)  |

## Why

The skills and the daemon were a separate repository only because the extensions
they drive were closed. That reason is gone: **`web-tools` is now the open base
itself** — the shared browser-tool primitives, the two agent-free extensions
built on them (**WebCLI** and **localmd Connect**), the bridge daemon, and the
agent skills, all in one place.

A skill that documents a tool surface belongs next to the code that defines that
surface. Keeping them in separate repositories is exactly how a skill ends up
telling an agent to call a tool the installed extension does not have.

The full Web Agent extension's own daemon repo (`web-agent-skills`) was retired
at the same time. One daemon now serves all three shells; only the port differs.

---

Everything below is the archived README, kept for reference. **Its commands point
at this repository and no longer work.**

<details>
<summary>Archived README (2026-09-06)</summary>

The public skills + bridge daemon for **web-tools** — the shared, open base that
powers two Chrome extensions:

- **WebCLI** — headless, agent-free. Exposes your logged-in browser's primitive
  base (generic browser tools + `eval_js` + recon + site scripts) to external AI
  agents (Claude Code, Codex, …) over a local bridge you talk to with `curl`.
- **localmd Connect** — the same primitive base plus knowledge-base features, for
  the [localmd](https://localmd.app) app.

Both speak the same primitives, so the **adapter skills** here work with either.

### What's in here

```
server.mjs                     the bridge daemon (npx/node entry, web-tools-bridge bin)
skills/
  webcli/SKILL.md              driving guide: how a CLI agent drives WebCLI
  adapters/                    adapter skills — "a way to reach site X", as data
    README.md                  what they are + the robustness ladder for building one
```

### HTTP API (binds 127.0.0.1 only)

| Method + path   | Result                                                      |
| --------------- | ----------------------------------------------------------- |
| `GET /ping`     | `{ok:true}`                                                 |
| `GET /status`   | `{ok, connected, port, client, tools}`                      |
| `GET /tools`    | `{ok, tools:[…]}` — tools in OpenAI-tool shape              |
| `POST /command` | body `{tool, args}` → `{ok, result}` or `{ok:false, error}` |

### History

Renamed from `webcli-skills` (2026-09): the base grew beyond WebCLI's generic
tools into the shared **web-tools** primitive base (eval_js + recon + site
scripts), and adapter skills replace the old per-site marketplace.

</details>
