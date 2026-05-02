---
name: readwise-inbox-triage
description: >
  Triage the user's Readwise Reader inbox one article at a time. Use this skill
  whenever the user says things like "triage my Readwise inbox", "walk me through
  my Reader queue", "help me clear my reading list", "Readwise triage", or "inbox
  triage". The skill fetches unread articles, summarizes each one with a relevance
  take, then moves documents based on user decisions (keep, archive, save for later).
  Requires the Readwise MCP to be connected.
compatibility: "Requires Readwise MCP (reader_list_documents, reader_move_documents)"
---

# Readwise Inbox Triage

Walk the user through their Readwise Reader inbox one article at a time,
summarizing each item and letting them decide what to do with it.

---

## Step 1: Clarify batch size

Ask the user how many articles they want to triage in this session (suggest 3–5
as a default). If they've already said a number, use that and skip the question.

## Step 2: Fetch articles

Call `reader_list_documents` with:
- `location`: `new` (the inbox)
- `category`: `article`
- `limit`: the agreed batch size
- `seen`: `false` (unread only)
- `response_fields`: `["title", "author", "summary", "word_count", "reading_time", "saved_at", "source_url"]`

Store the document `id` for each result — you'll need it for moves.

## Step 3: Present each article

For each article, show a card like this:

---
**📄 Article N of N**
**"[Title]"** — [Author]
⏱ [reading_time] read | Saved [saved_at, human-readable date]

[2–3 sentence summary of what the article covers]

**My take:** [1–2 sentence relevance assessment — why this might or might not be
worth the user's time, given what you know about them and their interests]

---

Then ask: **Keep, Save for Later, or Archive?**

### Tips for a good "My take"
- Weigh reading time against depth/novelty — a 2-min read gets more benefit of the doubt
- Flag if the topic overlaps with the user's known interests (tech, AI, leadership,
  faith/philosophy, productivity, software engineering)
- Note if the source is high-signal (e.g., Gergely Orosz, Seth Godin, known newsletters)
- Be honest if it looks like noise or a duplicate of something they likely already know

## Step 4: Act on the decision

| User says | Action |
|-----------|--------|
| **Keep** | Do nothing — leave in inbox |
| **Save for Later** | Call `reader_move_documents` with `location: "later"` |
| **Archive** | Call `reader_move_documents` with `location: "archive"` |

Move immediately after each decision before presenting the next article.

## Step 5: Handle source URL requests

If the user asks for the original URL of an article, call `reader_list_documents`
with `id` set to that document's ID and `response_fields: ["source_url", "url"]`.
Return `source_url` if available, otherwise fall back to the Reader proxy `url`.

## Step 6: Wrap up

After all articles are triaged, give a brief summary:
- How many kept / saved for later / archived
- Any patterns worth noting (e.g., "3 of your unread articles were about AI")
- Offer to run another batch if they want to continue

---

## Notes

- Articles saved from RSS feeds sometimes appear as `category: rss` rather than
  `article` — if the user wants to include those, run a second fetch with
  `category: rss` and `location: feed`.
- Podcasts, tweets, and PDFs are separate categories — don't mix them into an
  article triage session unless the user asks.
- If `summary` is empty for a document, fetch the `content` field or use
  `reader_get_document_details` to get more context before presenting.
