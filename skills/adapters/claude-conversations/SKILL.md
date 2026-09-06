---
name: claude-conversations
description: List and read the user's own Claude.ai conversations (titles + full message transcripts) to import into a knowledge base. Use when the user wants to save, summarize, or search their Claude chats.
invocation: model
---

# Claude.ai conversations

Read the user's own Claude.ai chat history. Pure HTTP — cookie-authed `fetch_url`
calls, no tab. (The user must be signed in to claude.ai.)

## Steps

1. **Org id** (Claude scopes everything under an organization):
   ```
   fetch_url {url:"https://claude.ai/api/organizations", format:"json"}
   → [0].uuid          # the personal org
   ```

2. **List conversations** (newest first; raise `limit` for more):
   ```
   fetch_url {url:"https://claude.ai/api/organizations/<ORG>/chat_conversations?limit=20", format:"json"}
   → [{ uuid, name, updated_at }, …]
   ```
   To find one by title, list and match `name`. If the user gave a
   `claude.ai/chat/<uuid>` URL, take the `<uuid>` from it and skip to step 3.

3. **Read one conversation** in full:
   ```
   fetch_url {url:"https://claude.ai/api/organizations/<ORG>/chat_conversations/<CONV>?tree=True&rendering_mode=messages&render_all_tools=true", format:"json"}
   → json.name, json.chat_messages = [{ sender, content:[{type,text}], created_at }, …]
   ```
   Each message: `sender` is `"human"` or `"assistant"`; its text is the `text`
   of the `content` blocks joined. Assemble the transcript in order.

## Notes

- Cookie-authed via `fetch_url` (default `with_cookies:true`); fails signed out.
- `updated_at` / `created_at` are ISO timestamps — good for "conversations since
  <date>".
- Long conversations can be large; read markdown/text you will keep, and put the
  transcript into the KB as a note with the conversation's `name` + its
  `claude.ai/chat/<uuid>` URL.

## Verified

Real machine: org list → 3 recent conversations (titles + timestamps) → one read
back with 8 messages, first `human` message text intact.
