---
name: zhihu
description: Search Zhihu (知乎) and read answers/articles as clean text with author and vote counts, to bring into a knowledge base. Use when the user wants to search 知乎, or save/summarize a Zhihu answer, article, or question.
invocation: model
---

# Zhihu (知乎)

Search Zhihu and read answers/articles through its own JSON API — cookie-authed
`fetch_url`, no tab. (Signed-in gets full content; some is gated for
signed-out.)

## Search

```
fetch_url {url:"https://www.zhihu.com/api/v4/search_v3?t=general&q=<URL-ENCODED>&limit=20", format:"json"}
→ json.data = [{ object: { type, id, url, title/excerpt, author, … } }, …]
```
Keep `object.type` of `answer` / `article` / `zvideo`; skip `ai_zhida` and
`relevant_query` (not content). Each object's `url` / `id` feeds the reads below.

## Read an answer

```
fetch_url {url:"https://www.zhihu.com/api/v4/answers/<ANSWER_ID>?include=data%5B*%5D.content%2Cvoteup_count%2Cauthor", format:"json"}
→ json.content (HTML — strip tags), json.author.name, json.voteup_count, json.question.name
```
An answer URL is `zhihu.com/question/<Q>/answer/<ANSWER_ID>` — take the last id.

## Read a question's answers

```
fetch_url {url:"https://www.zhihu.com/api/v4/questions/<QUESTION_ID>/answers?include=data%5B*%5D.content%2Cvoteup_count%2Cauthor&limit=10&sort_by=default", format:"json"}
→ json.data = [ …answers… ]   (page with json.paging.next)
```

## Read an article (column post)

The article API (`api/v4/articles/<id>`) returns **403** — do NOT use it. Read
the column page as Markdown instead:
```
fetch_url {url:"https://zhuanlan.zhihu.com/p/<ARTICLE_ID>", format:"markdown", selector:".Post-Title, .Post-RichTextContainer"}
→ markdown
```

## Notes

- The `content` fields are HTML — strip tags (or keep as HTML if the KB wants
  it). `include` needs the bracketed form `data[*].content,voteup_count,author`,
  URL-encoded as `data%5B*%5D.content%2C…`.
- Put an answer into the KB as a note with the question name, author, vote count,
  and the `zhihu.com/question/<Q>/answer/<A>` URL.

## Verified

Real machine: search "Obsidian笔记" → 5 results (answer/article types) → one
answer read back (author 退役搬砖工, 177 votes, 992 chars of text).
