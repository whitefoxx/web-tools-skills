---
name: x-twitter
description: Read an X (Twitter) thread/tweet or the user's own bookmarks as structured rows, to bring into a knowledge base. Use when the user wants to save/summarize an X thread or import their X bookmarks.
invocation: model
---

# X (Twitter) — threads + bookmarks

X has no open API, but its web GraphQL is reachable from an **x.com tab** with the
user's own session. This is the most fragile skill here (X rotates its query ids
and feature flags), so it is two steps: fetch the current ids from a maintained
config, then make the request from the page with `eval_js`.

> Signed in to x.com required. This reads private data (bookmarks) — treat it as
> the user's, and never post/like/follow from here (that is a write).

## Step 1 — current query ids (they change; a community config tracks them)

```
fetch_url {url:"https://raw.githubusercontent.com/fa0311/twitter-openapi/refs/heads/main/src/config/placeholder.json",
           format:"json", with_cookies:false}
→ ph[<Op>] = { "@path": "/i/api/graphql/<queryId>/<Op>", queryId, variables, features, fieldToggles }
```
Ops you want: **`TweetDetail`** (a tweet + its thread/replies — set
`variables.focalTweetId`) and **`Bookmarks`** (the user's bookmarks — set
`variables.count`, page with `variables.cursor`). Others exist (`UserTweets`,
`HomeTimeline`, `SearchTimeline`) with the same shape.

## Step 2 — request from an x.com tab

```
open_url {url:"https://x.com/i/bookmarks", active:true}   # any x.com page works
wait_for_selector {selector:'article, [data-testid="cellInnerDiv"]'}
```
Then `eval_js` on that tab — pass the op's `@path`, `variables`, `features`,
`fieldToggles` from step 1 into `PATH` / `VARS` / `FEAT` / `TOGG`:

```js
const ct0 = (document.cookie.match(/(?:^|; )ct0=([^;]+)/) || [])[1];
if (!ct0) return { err: 'no ct0 cookie — is the user signed in?' };
// The public web bearer (the same token every x.com page uses):
const BEARER = 'AAAAAAAAAAAAAAAAAAAAANRILgAAAAAAnNwIzUejRCOuH5E6I8xnZz4puTs%3D1Zv7ttfk8LF81IUq16cHjhLTvJu4FA33AGWWjCpTnA';
const url = PATH + '?variables=' + encodeURIComponent(JSON.stringify(VARS))
                 + '&features=' + encodeURIComponent(JSON.stringify(FEAT))
                 + '&fieldToggles=' + encodeURIComponent(JSON.stringify(TOGG));
const x = new XMLHttpRequest(); x.open('GET', url, false);
x.setRequestHeader('authorization', 'Bearer ' + BEARER);
x.setRequestHeader('x-csrf-token', ct0);
x.setRequestHeader('x-twitter-auth-type', 'OAuth2Session');
x.setRequestHeader('x-twitter-active-user', 'yes');
x.setRequestHeader('content-type', 'application/json');
x.send(null);
if (x.status !== 200) return { status: x.status, head: (x.responseText || '').slice(0, 200) };
const j = JSON.parse(x.responseText);
// Reduce to rows IN THE PAGE — never return the 150 KB payload. Walk for tweet
// legacy objects (full_text + id_str), de-duped, in encounter order:
const seen = new Set(), rows = [];
(function walk(o) {
  if (!o || typeof o !== 'object') return;
  if (o.full_text && o.id_str && !seen.has(o.id_str)) {
    seen.add(o.id_str);
    rows.push({ id: o.id_str, text: o.full_text, at: o.created_at });
  }
  for (const k in o) walk(o[k]);
})(j);
// cursors for the next page (Bookmarks): entries of type TimelineTimelineCursor
const cursors = [];
(function c(o){ if(!o||typeof o!=='object')return; if(o.cursorType==='Bottom'&&o.value)cursors.push(o.value); for(const k in o)c(o[k]); })(j);
return { count: rows.length, rows, nextCursor: cursors[0] || null, errors: (j.errors||[]).map(e=>e.message).slice(0,2) };
```

Page bookmarks by putting `nextCursor` into `VARS.cursor` and repeating.

## Notes / when it breaks

- **`ct0` + the public bearer** are what authenticate; both come from the page.
- **A `queryId`/feature error** (400 with a message about a bad query id or an
  unknown feature) means the config lagged X's latest deploy. Refresh the
  placeholder file, or extract the ids from the page's own JS: scan the
  `client-web` bundles (`document.scripts` srcs + `performance.getEntriesByType('resource')`)
  for `operationName:"<Op>"` and read the nearby `queryId:"…"` +
  `featureSwitches:[…]` — the marketplace's twitter adapter did exactly this.
- The `full_text` walk pulls quoted/retweeted tweets too; for a strict thread use
  `data.threaded_conversation_with_injections_v2.instructions`, for bookmarks
  `data.bookmark_timeline_v2.timeline.instructions`.

## Verified

Real machine: `Bookmarks` (count 5) → 7 tweets; `TweetDetail`
(focalTweetId a Karpathy thread) → 30 tweets. Both status 200, reduced to rows
in the page.
