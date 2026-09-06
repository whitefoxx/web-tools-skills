---
name: reddit
description: Read a Reddit thread (post + comment tree), search Reddit, or list a subreddit, as clean structured data to bring into a knowledge base. Use when the user wants to save/summarize a Reddit discussion or search Reddit.
invocation: model
---

# Reddit

Reddit exposes JSON for almost every page by appending `.json` — no API key, no
tab. `fetch_url` sends the user's cookies (so quarantined/private-subscribed
content works too). Always add `raw_json=1` to get unescaped text.

## Read a thread (post + comments)

```
fetch_url {url:"https://www.reddit.com/r/<SUB>/comments/<ID>.json?limit=50&raw_json=1&sort=top", format:"json"}
→ [ <post listing>, <comments listing> ]
```
- **Post**: `json[0].data.children[0].data` → `title`, `selftext`, `author`,
  `score`, `num_comments`, `url`, `created_utc`.
- **Comments**: `json[1].data.children[]`. Each has `kind`:
  - `"t1"` → a comment: `data.author`, `data.score`, `data.body`, and
    `data.replies` (another listing, or `""` when none) — **recurse** for the
    tree.
  - `"more"` → collapsed children (`data.children` = ids). Fetch them with
    `.../api/morechildren.json?link_id=t3_<ID>&children=<id1,id2,…>&raw_json=1`
    only if you need the deep tail; the first page usually suffices.

Any thread URL works — take `/r/<SUB>/comments/<ID>/…` from it and append `.json`.

## Search

```
fetch_url {url:"https://www.reddit.com/search.json?q=<URL-ENCODED>&limit=20&raw_json=1&sort=relevance", format:"json"}
→ json.data.children[].data = { title, subreddit, author, score, permalink, url, num_comments }
```
Within one subreddit: `https://www.reddit.com/r/<SUB>/search.json?q=…&restrict_sr=1`.

## List a subreddit

```
fetch_url {url:"https://www.reddit.com/r/<SUB>/hot.json?limit=25&raw_json=1", format:"json"}
# or top.json?t=week / new.json / rising.json
→ json.data.children[].data   (page with json.data.after → &after=<t3_id>)
```

## Notes

- `raw_json=1` everywhere, or `&amp;`/`&lt;` litter the text.
- `permalink` is relative — prefix `https://www.reddit.com`.
- Put a thread into the KB as a note with the post title, subreddit, top
  comments, and the permalink.

## Verified

Real machine: an r/SideProject thread → post (title, author, 97 score, 809-char
body) + 4 top-level comments; search "obsidian alternative" → 3 titled results.
