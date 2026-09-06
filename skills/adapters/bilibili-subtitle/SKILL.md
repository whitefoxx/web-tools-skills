---
name: bilibili-subtitle
description: Get a Bilibili video's subtitles/transcript (CC or AI-generated) as timestamped lines. Use when the user wants a 哔哩哔哩/B站 video's subtitles, transcript, or to summarize/quote a Bilibili video.
invocation: model
---

# Bilibili subtitle

Get a video's subtitle track as text. Pure HTTP — three `fetch_url` calls with
the user's Bilibili login cookies, no tab needed. (`fetch_url` sends cookies by
default; the user must be signed in to bilibili.com for the subtitle list.)

## Steps

1. **Parse the `bvid`** from the URL: the `BV…` id in
   `https://www.bilibili.com/video/BV1GJ411x7h7`. (Short `b23.tv/…` links:
   `fetch_url` it first and read the redirected URL.)

2. **`bvid` → `cid`** (a video may have several parts; the first `cid` is part 1):
   ```
   fetch_url {url:"https://api.bilibili.com/x/web-interface/view?bvid=<BVID>", format:"json"}
   → json.data.cid, json.data.title
   ```

3. **`cid` → the subtitle track list** — the player-info endpoint. The `wbi`
   variant works WITHOUT a signature for this field:
   ```
   fetch_url {url:"https://api.bilibili.com/x/player/wbi/v2?bvid=<BVID>&cid=<CID>", format:"json"}
   → json.data.subtitle.subtitles = [{ lan, lan_doc, subtitle_url }, …]
   ```
   `lan` is like `zh-CN` / `zh-Hans` / `en-US`; `ai-zh` and `zh-CN` are usually
   AI-generated. An **empty list means the video has no subtitles** — report that,
   don't invent text. Pick the language the user asked for, else the first.

4. **Fetch the chosen `subtitle_url`** (it starts with `//` — prefix `https:`):
   ```
   fetch_url {url:"https://<subtitle_url>", format:"json"}
   → json.body = [{ from, to, content }, …]
   ```
   Assemble `{ t: round(from), text: content }` rows in order.

## Notes

- Everything is cookie-authed via `fetch_url` (default `with_cookies:true`);
  subtitles fail for signed-out sessions.
- The `subtitle_url` host is `aisubtitle.hdslb.com` (AI tracks) or
  `s1.hdslb.com`; both return the same `{body:[{from,to,content}]}` shape.
- No `wbi` signature is needed for the subtitle field. If a future change starts
  rejecting the unsigned call, the `w_rid`/`wts` signature can be computed in
  `eval_js` on a bilibili tab (the page's own `getMixinKey` logic), but that has
  not been necessary.

## Verified

`BV1GJ411x7h7` (Rick Astley MV) → 47 lines, first `0s 永不放弃你——瑞克·艾斯里`.
