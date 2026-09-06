---
name: youtube-transcript
description: Get the full transcript of a YouTube video (any length, auto or uploaded captions) as timestamped lines. Use when the user wants a video's transcript or captions, or to summarize or quote a YouTube video.
invocation: model
---

# YouTube transcript

Get a video's transcript by driving YouTube's own **"Show transcript"** panel and
reading the rendered DOM. This is pot-free and robust: it uses the exact UI a
person clicks, so it survives YouTube's private-API changes.

Do NOT try the InnerTube `get_transcript` API or fetch a caption track's
`baseUrl` directly — both need a `pot` (proof-of-origin) token the page computes
and cannot be reproduced. Today they return `400 "Precondition failed"` or an
empty `200`. The caption fetch also does not cross the tab's network (it is
ServiceWorker-side), so watching the wire does not help either. The panel is the
reliable route.

## Steps

1. `open_url` the watch URL with **`active: true`** (the panel needs a real,
   foregrounded tab).
2. `wait_for_selector` `#movie_player`, then call `eval_js` with the snippet
   below (`tab_id` = that tab, a generous `timeout_ms` like 90000). It returns
   `{ panel, count, rows }`, each row `{ t: "M:SS", text }`. When a video has no
   transcript it returns `{ error: "no_transcript" }` (no button) or
   `{ error: "panel_empty" }` (panel exists but never fills — e.g. some music
   videos); report that plainly rather than inventing text.
3. `close_tab` when done.

```js
(async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));
  // Two panel generations exist; read cues from either.
  const CUE = 'ytd-transcript-segment-renderer, transcript-segment-view-model';
  const buttons = () => [...document.querySelectorAll('button')]
    .filter(e => /^\s*show transcript\s*$/i.test(e.getAttribute('aria-label') || ''))
    .sort((a, b) => (b.offsetParent ? 1 : 0) - (a.offsetParent ? 1 : 0)); // visible first
  const openPanel = () => [...document.querySelectorAll('ytd-engagement-panel-section-list-renderer')]
    .find(p => /transcript/i.test(p.getAttribute('target-id') || '')
            && !/HIDDEN/.test(p.getAttribute('visibility') || '')
            && p.querySelector(CUE));
  // The button often lives inside the collapsed description.
  const ex = document.querySelector('#expand, tp-yt-paper-button#expand');
  if (ex) { ex.click(); await sleep(700); }
  const bs = buttons();
  if (!bs.length) return { error: 'no_transcript', detail: 'no "Show transcript" control on this video' };
  let panel = null;
  for (const b of bs) {
    b.click();
    for (let i = 0; i < 16 && !panel; i++) { await sleep(400); panel = openPanel(); }
    if (panel) break;
  }
  if (!panel) return { error: 'panel_empty', detail: 'transcript panel opened but stayed empty' };
  // The list virtualizes on long videos; scroll it until the count stops growing.
  const sc = panel.querySelector('#segments-container, .ytwTranscriptSegmentListViewModelHost') || panel;
  let last = -1, stable = 0;
  for (let i = 0; i < 200; i++) {
    const n = panel.querySelectorAll(CUE).length;
    if (n === last) { if (++stable >= 3) break; } else { stable = 0; last = n; }
    try { sc.scrollTop = sc.scrollHeight; } catch (e) {}
    await sleep(120);
  }
  const rows = [...panel.querySelectorAll(CUE)].map(s => {
    // classic: .segment-timestamp / .segment-text ;
    // modern: .ytwTranscriptSegmentViewModelTimestamp (NOT its *A11yLabel sibling)
    //         + the ytAttributedStringHost span[role=text].
    const ts = s.querySelector('.segment-timestamp, .ytwTranscriptSegmentViewModelTimestamp');
    const tx = s.querySelector('.segment-text, .ytAttributedStringHost, span[role="text"]');
    return { t: (ts ? ts.textContent : '').trim(), text: (tx ? tx.textContent : '').replace(/\s+/g, ' ').trim() };
  }).filter(r => r.text);
  return { panel: panel.getAttribute('target-id'), count: rows.length, rows };
})()
```

## Notes

- Both panel generations are handled: the classic
  `ytd-transcript-segment-renderer` and the newer `transcript-segment-view-model`
  ("modern transcript view"). YouTube serves one or the other per video/account,
  which is why an extractor pinned to one set of selectors silently returns
  nothing.
- Long videos virtualize the list, so the snippet scrolls to the end before
  reading.
- A specific language can be chosen from the panel footer's track menu if needed;
  the default track is what a viewer would see.

## Verified

Real machine, each one `open_url` + one `eval_js` (~11-17 s):

| video               | rows | span         |
| ------------------- | ---: | ------------ |
| Let's build GPT     | 1106 | 0:00→1:56:15 |
| Intro to LLMs (1hr) |  581 | 0:00→59:45   |
| a 27-language talk  |  422 | 0:09→19:25   |
| Rick Astley (modern panel) | 24 | 0:01→3:23 |
