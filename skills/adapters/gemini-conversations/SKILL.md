---
name: gemini-conversations
description: List and read the user's own Google Gemini conversations (titles + full turn transcripts) to import into a knowledge base. Use when the user wants to save, summarize, or search their Gemini chats.
invocation: model
---

# Gemini conversations

Read the user's own Gemini chat history. Gemini has no read API here, so this is
DOM-driven: an **active tab** on gemini.google.com + `eval_js`. (The user must be
signed in to their Google account.)

## Steps

1. **Open the app** and let it render:
   ```
   open_url {url:"https://gemini.google.com/app", active:true}
   wait_for_selector {selector:"main"}    # then give it a couple seconds
   ```

2. **List recents** — the sidebar links to each conversation:
   ```js
   // eval_js on that tab:
   const seen = new Set(), rows = [];
   for (const a of document.querySelectorAll('a[href*="/app/"]')) {
     const t = (a.textContent || '').trim();
     const m = (a.getAttribute('href') || '').match(/\/app\/([a-f0-9]+)/);
     if (m && t && !seen.has(m[1])) { seen.add(m[1]); rows.push({ id: m[1], title: t }); }
   }
   return rows;
   ```
   Match `title` to find one. A `gemini.google.com/app/<id>` URL gives the `<id>`.

3. **Read one conversation** — navigate the same tab to it, then read the turns:
   ```js
   // eval_js #1: navigate (SPA) and let it load
   window.location.href = 'https://gemini.google.com/app/<ID>'; return true;
   // wait ~5s, then eval_js #2:
   const strip = s => (s || '').replace(/^\s*(You said|Gemini said)\s*/i, '').replace(/\s+/g, ' ').trim();
   // Long chats virtualize — scroll the container up to load earlier turns first.
   const sc = document.querySelector('main [data-test-id="chat-history-container"], main');
   for (let i = 0; i < 30; i++) { sc.scrollTop = 0; await new Promise(r => setTimeout(r, 150)); }
   const turns = [];
   for (const el of document.querySelectorAll('user-query, model-response')) {
     const role = el.tagName.toLowerCase() === 'user-query' ? 'user' : 'model';
     const text = strip(el.innerText);
     if (text) turns.push({ role, text });
   }
   return turns;
   ```

## Notes

- Needs an **active/foregrounded tab** (Gemini renders lazily); the other AI-chat
  skills (claude-conversations / chatgpt-conversations) are API-based and need no
  tab — prefer those when the user is on Claude/ChatGPT.
- `user-query` / `model-response` are Gemini's custom elements; the role comes
  from the tag. Strip the `You said` / `Gemini said` a11y prefixes.
- Very long conversations virtualize — scroll to the top to force earlier turns
  in before reading, or read in chunks.

## Verified

Real machine: 5 recent conversations (ids + titles) → one read back with its
user/model turns, a11y prefixes stripped.
