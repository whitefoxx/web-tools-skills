---
name: chatgpt-conversations
description: List and read the user's own ChatGPT conversations (titles + full message transcripts) to import into a knowledge base. Use when the user wants to save, summarize, or search their ChatGPT chats.
invocation: model
---

# ChatGPT conversations

Read the user's own ChatGPT chat history. Cookie-authed `fetch_url`, no tab, plus
a short-lived access token the session hands out. (The user must be signed in to
chatgpt.com.)

> **The access token is a credential.** Use it ONLY in the `Authorization`
> header of the two calls below. Never write it into the knowledge base, a note,
> or your reply — it grants access to the account.

## Steps

1. **Access token** (the web session mints it):
   ```
   fetch_url {url:"https://chatgpt.com/api/auth/session", format:"json"}
   → json.accessToken            # sensitive — header only
   ```

2. **List conversations** (newest first; page with `offset`):
   ```
   fetch_url {url:"https://chatgpt.com/backend-api/conversations?offset=0&limit=20&order=updated",
              format:"json", headers:{"Authorization":"Bearer <TOKEN>"}}
   → json.items = [{ id, title, update_time }, …]
   ```
   Find one by title by matching `title`. A `chatgpt.com/c/<id>` URL gives the
   `<id>` directly — skip to step 3.

3. **Read one conversation** in full:
   ```
   fetch_url {url:"https://chatgpt.com/backend-api/conversation/<ID>",
              format:"json", headers:{"Authorization":"Bearer <TOKEN>"}}
   → json.title, json.mapping = { <node_id>: { message: { author:{role}, content:{parts:[…]}, create_time } }, … }
   ```
   `mapping` is a tree. Collect every node whose `message` has non-empty
   `content.parts`, take `author.role` (`user` / `assistant`) + the joined
   `parts`, and order by `create_time` to get the transcript.

## Notes

- Cookie-authed (the session call) + the bearer token for `backend-api`; both
  fail signed out.
- Unlike Claude.ai (cookie-only, no token in play), ChatGPT's token passes
  through this flow — one more reason never to persist it.
- Put the transcript into the KB as a note with the `title` + the
  `chatgpt.com/c/<id>` URL; drop the token.

## Verified

Real machine: session token present → 3 recent conversations (titles +
timestamps) → one read back with 12 messages, first `user` message text intact.
