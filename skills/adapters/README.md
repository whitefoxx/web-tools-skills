# Adapter skills

An **adapter skill** is the durable form of "a way to reach site X" — a
`SKILL.md` (the knowledge: which route works and why) plus an executable core (a
fenced `eval_js` snippet, portable to any of the web-tools shells). It replaces
the old per-site adapter *marketplace*: capability the agent builds from generic
primitives and keeps as data you own, never shipped extension code.

These are **consumer-agnostic**: the same skill works whether it is driven by a
CLI agent over WebCLI or by the localmd app over localmd Connect — both shells
expose the same `eval_js` primitive.

## Install one

```bash
# a CLI agent (WebCLI): install globally
npx skills add whitefoxx/web-tools-skills -g

# localmd: install into your knowledge base's .agents/skills/
```

Or hand the repo URL to your agent and let it read the skill it needs.

## Build your own — the robustness ladder

Most sites need no pre-written skill: with `eval_js` and the ladder below, an
agent one-shots the extraction live and saves it as a skill. Stop at the first
rung that works:

1. **The site's own JSON API** — `fetch(api, {credentials:'include'})` from
   inside the page. Zero selectors, most durable.
2. **Embedded page state** — `__NEXT_DATA__` / `__NUXT__` / a
   `<script type="application/json">` blob (use `find_structured_data`).
3. **The site's OWN UI as the data source**, when a private API is locked behind
   a token / signature / pot — drive the panel or list a person clicks and read
   the DOM. (A YouTube transcript comes from its "Show transcript" panel, not the
   pot-locked caption API — see `youtube-transcript/`.)
4. **Last resort: scrape the DOM** with STABLE selectors (`data-testid` / `aria` /
   semantic tags / `href`), never random build-hash classes. `get_a11y_tree` and
   `find_in_dom` help pick anchors.

Rules of thumb: **reduce to rows inside the page** — `eval_js` returns the data
you need, not the whole payload. A read that must POST (GraphQL / InnerTube)
needs `allow_write:true`. When it works, save it here as a skill so next time is
one call, not a rebuild.

## The skills

- [`youtube-transcript/`](./youtube-transcript/) — the full transcript of any
  YouTube video, pot-free, by driving the "Show transcript" panel and reading the
  DOM. Verified on five videos (24 → 1106 rows). More robust than a private-API
  adapter, which YouTube's `pot` wall now breaks.
- [`bilibili-subtitle/`](./bilibili-subtitle/) — a Bilibili video's subtitle
  track (CC or AI-generated) as timestamped lines. Pure HTTP: three `fetch_url`
  calls (`view` → `wbi/v2` → the subtitle body), no tab, the `wbi` field works
  unsigned. Verified on the Rick Astley MV (47 lines).
- [`claude-conversations/`](./claude-conversations/) — list + read the user's own
  Claude.ai chats (titles + full transcripts) to import into a KB. Cookie-only
  `fetch_url`, no token. Verified (3 conversations, one read back, 8 messages).
- [`chatgpt-conversations/`](./chatgpt-conversations/) — the same for ChatGPT
  (`api/auth/session` token → `backend-api`; the token is a credential, header
  only). Verified (3 conversations, one read back, 12 messages).
- [`gemini-conversations/`](./gemini-conversations/) — the same for Google Gemini,
  DOM-driven (no read API): an active tab + `eval_js` over `user-query` /
  `model-response`. Verified (5 recents, one read back).
- [`zhihu/`](./zhihu/) — search 知乎 and read answers/articles as clean text with
  author + vote counts. Cookie-authed `api/v4`; articles read as Markdown (the
  article API 403s). Verified (search → answer, 177 votes, 992 chars).
