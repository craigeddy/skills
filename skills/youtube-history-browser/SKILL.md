---
name: youtube-history-browser
description: >-
  Browse the user's YouTube watch history through their connected Chrome
  browser and report what they've watched, optionally over a configurable
  look-back window in days (defaults to today). Trigger whenever the user
  asks about their own past YouTube viewing — phrasings like "what did I
  watch on YouTube", "show my YouTube history", "find that video I watched
  yesterday on YouTube about X", "summarize my YouTube viewing this week",
  even if they don't say "skill". Do NOT trigger for searching YouTube for
  new videos, trending or recommendation queries, summarizing a single
  YouTube video the user already linked, generic browser history,
  watch history on other platforms (Netflix, Disney+, Hulu, Prime Video),
  playback requests, or destructive actions (deleting, clearing, or pausing
  the watch history) — those are out of scope. Requires the Claude in Chrome
  MCP and an active Chrome session signed in to YouTube.
---

# YouTube History Browser

## What this skill does

Opens the user's YouTube watch history page (`https://www.youtube.com/feed/history`) in their connected Chrome browser, reads the page, and produces a clean grouped-by-day list of videos they've watched within a requested window of days.

YouTube renders watch history as a single page broken into date sections — the most recent section is labelled `Today`, then `Yesterday`, then specific weekday names ("Saturday", "Friday", ...), and eventually full dates further back. Each section contains video cards with a title, channel, runtime, and view count. This skill walks those sections and extracts what the user wants.

## Inputs

The skill takes one optional parameter, **days**, controlling how far back to look:

- `days = 1` (default, also matches "today") — only the `Today` section.
- `days = 2` — `Today` + `Yesterday`.
- `days = N` for N ≥ 3 — `Today`, `Yesterday`, and the next N-2 prior date sections in the page order, regardless of whether they're labelled by weekday or full date.

If the user phrases it differently ("last week", "past 3 days", "everything since Monday"), translate that into a `days` integer before running. Ask the user only if the phrasing is genuinely ambiguous — the user prefers precision, so when in doubt confirm with one short question rather than guessing.

## Procedure

### 1. Make sure Chrome is reachable

Before navigating, confirm the Claude in Chrome MCP has at least one connected browser. If `list_connected_browsers` returns an empty list, stop and tell the user they need to connect their browser via the Claude in Chrome extension first — the skill cannot do anything without it.

If a tab group doesn't exist for the session yet, call `tabs_context_mcp` with `createIfEmpty: true` to bootstrap one. Reuse the empty tab it creates rather than spawning extra tabs.

### 2. Navigate to the watch history page

Navigate the active tab to `https://www.youtube.com/feed/history`. Then call `read_page` on that tab with `filter: "all"` — `get_page_text` returns very little here because the history list is rendered as virtualized cards rather than article text, so the accessibility tree from `read_page` is the reliable source.

If the read shows the user is not signed in (no `Watch history` heading, or YouTube is showing the sign-in prompt instead of history cards), stop and tell the user: they need to sign into YouTube in the connected browser before this can work. Do not attempt to log them in.

### 3. Parse the sections

In the accessibility tree, watch history items appear as a flat sequence under the `main` region. The structure is roughly:

- A `generic "Today"` (or `"Yesterday"`, or a date string) marks the start of a section.
- Then for each video in that section: a `link` whose nested `generic` holds the runtime (e.g. `"21:50"`), a `button "Go to channel <Channel Name>"` (or a `link "Collaboration channels"` for multi-creator videos), a `heading` containing the title, and a `group` containing the channel name text and view count.

Walk the items in order. Whenever a section header `generic` is encountered, switch the "current section" label. For each video record, capture: title, channel(s), runtime, view count, and the `href` from the link (so the user can click through). Strip query parameters that are tracking noise but keep the `v=` video ID and `t=` timestamp if present, since the timestamp tells the user where they left off.

For multi-creator videos where the button reads `"Collaboration channels"`, the channel names appear inside the `group` under a nested `generic` like `"AI Engineer and Matt Pocock"`. Capture that text verbatim.

### 4. Apply the days window

Determine which sections to include:

- The first section in document order is always `Today`.
- The second section, if present, is always `Yesterday`.
- Subsequent sections are older days.

Truncate the section list at `days` entries. If the page has fewer sections than `days` (e.g. user only has 2 days of history), include what exists and mention the cap in the response so the user knows you didn't silently miss anything.

### 5. (Only if needed) Scroll for more history

If `days` exceeds the number of sections currently rendered, scroll the page down to trigger YouTube's lazy load and re-read. A reasonable approach: use `javascript_tool` to call `window.scrollTo(0, document.body.scrollHeight)`, wait briefly, then `read_page` again. Repeat until you have enough sections or two consecutive scrolls return no new sections (in which case you've hit the end of the user's history).

Don't scroll preemptively when `days = 1` — the Today section is rendered immediately, and unnecessary scrolling slows the response and risks pulling in irrelevant content.

### 6. Format the response

Present the result grouped by section, with the section label as a small header and each video as a single line containing title, channel, and runtime. Include the YouTube link on the title so the user can click through. Keep view counts out of the main list — they're rarely what the user cares about — but offer to include them if they ask. Example shape:

```
**Today**
1. [How to Make Claude Code Your AI Engineering Team](https://www.youtube.com/watch?v=...) — Y Combinator (21:50)
2. ...

**Yesterday**
1. ...
```

End with a brief one-sentence observation about the day's viewing if there's an obvious theme (e.g. "Heavy on AI coding workflows today"), but only if it's actually useful — don't manufacture a pattern out of three unrelated videos.

## Edge cases

- **Empty Today section**: YouTube still renders a `Today` heading even if the user hasn't watched anything yet that day, in which case the next item in the tree is the `Yesterday` heading. Handle this by treating any section with zero videos as empty rather than erroring.
- **Shorts vs. long-form**: The history page has tabs for All / Videos / Shorts / Podcasts / Music. Default to All. If the user asks for a specific category, click the matching tab via its accessibility ref before reading.
- **History paused**: If the user has YouTube watch history paused, the page shows a "Your watch history is off" message instead of the list. Report this clearly — there is nothing to extract.
- **Non-English UI**: Section labels may be localized ("Hoy", "Aujourd'hui"). If the labels don't match the expected English strings, fall back to "the first/second/Nth section in order" and mention the localization in the response so the user knows you weren't strict about matching.

## What not to do

- Don't attempt to delete entries, clear history, or click "Pause watch history" — those are destructive actions outside the scope of this skill, and they require explicit user confirmation through the chat anyway.
- Don't sign the user in. If they're signed out, say so and stop.
- Don't recommend "similar videos" or use the Search box. The user asked what they watched, not what to watch next.
