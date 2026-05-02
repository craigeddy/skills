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

Opens the user's YouTube watch history page (`https://www.youtube.com/feed/history`) in their connected Chrome browser, reads the JSON payload baked into the page, and produces a clean grouped-by-day list of videos they've watched within a requested window of days.

YouTube's history page ships with the *entire* list of date sections in a single `window.ytInitialData` object on initial load — typically months of history, far more than the rendered DOM ever virtualizes. This skill reads from that JSON directly. It is both more complete and faster than scrolling the page.

## Inputs

The skill takes one optional parameter, **days**, interpreted as "include sections whose date falls within the last N calendar days, today inclusive":

- `days = 1` (default, also matches "today") — only the `Today` section.
- `days = 2` — `Today` + `Yesterday`.
- `days = 7` — covers the past week.
- `days = N` for arbitrary N — all sections whose date is `>= today − (N − 1)`.

If the user phrases it differently ("last week", "past 3 days", "everything since Monday"), translate that into a `days` integer before running. Ask the user only if the phrasing is genuinely ambiguous — the user prefers precision, so when in doubt confirm with one short question rather than guessing.

## Procedure

### 1. Make sure Chrome is reachable

Before navigating, confirm the Claude in Chrome MCP has at least one connected browser. If `list_connected_browsers` returns an empty list, stop and tell the user they need to connect their browser via the Claude in Chrome extension first — the skill cannot do anything without it.

If a tab group doesn't exist for the session yet, call `tabs_context_mcp` with `createIfEmpty: true` to bootstrap one. Reuse the empty tab it creates rather than spawning extra tabs.

### 2. Navigate and spoof visibility

Navigate the active tab to `https://www.youtube.com/feed/history`.

**Critical:** the Cowork-connected tab is almost always a *background* tab from Chrome's perspective, so `document.visibilityState === "hidden"`. YouTube's SPA refuses to mount `ytd-browse` or populate the section list in hidden tabs — the page will look stuck or completely empty even after waiting many seconds. Immediately after navigating, run this in the tab via `javascript_tool`:

```js
new Promise(r => setTimeout(r, 6000)).then(() => {
  try {
    Object.defineProperty(document, 'hidden', { configurable: true, get: () => false });
    Object.defineProperty(document, 'visibilityState', { configurable: true, get: () => 'visible' });
    document.dispatchEvent(new Event('visibilitychange'));
  } catch (e) {}
  return {
    ready: !!window.ytInitialData,
    sectionCount: window.ytInitialData
      ? window.ytInitialData.contents?.twoColumnBrowseResultsRenderer?.tabs?.[0]?.tabRenderer?.content?.sectionListRenderer?.contents?.length
      : 0
  };
})
```

If `ready` is false, sleep a few more seconds and re-check. If after ~15s `ytInitialData` is still missing, fall through to the DOM-scrape fallback in section 6.

If the page is showing the signed-out screen instead, stop and tell the user to sign into YouTube in the connected browser. Do not attempt to log them in.

### 3. Read the section list from `ytInitialData`

The full date-grouped section list lives at:

```
window.ytInitialData
  .contents.twoColumnBrowseResultsRenderer
  .tabs[0].tabRenderer.content
  .sectionListRenderer.contents
```

Each entry is either an `itemSectionRenderer` (a date section) or a `continuationItemRenderer` (the lazy-load sentinel — ignore it; the initial payload typically already contains months of history, far more than any reasonable `days` window).

For each `itemSectionRenderer`:

- **Header label** — `.header.itemSectionHeaderRenderer.title.simpleText`, or `.title.runs[0].text`. Possible shapes: `"Today"`, `"Yesterday"`, a weekday name (e.g. `"Thursday"`), or a full date (e.g. `"Apr 25"`, `"Jan 3"`).
- **Items** — `.contents` array; each item is one of the three renderer shapes documented in section 4.

### 4. Parse each item

Three renderer shapes appear inside a section. Handle all three.

#### `lockupViewModel` (modern shape, most items)

```js
function parseLockup(lvm) {
  const meta = lvm.metadata?.lockupMetadataViewModel;
  const title = meta?.title?.content || '';
  const id = lvm.contentId || '';
  const rows = meta?.metadata?.contentMetadataViewModel?.metadataRows || [];
  // First row: channel name(s); second row: view count etc.
  const channel = (rows[0]?.metadataParts || [])
    .map(p => p.text?.content)
    .filter(Boolean)
    .filter(t => !/views?$/i.test(t))
    .join(' & ');
  let runtime = '';
  for (const o of lvm.contentImage?.thumbnailViewModel?.overlays || []) {
    for (const b of o.thumbnailBottomOverlayViewModel?.badges || []) {
      const t = b.thumbnailBadgeViewModel?.text;
      if (t && /^\d/.test(t)) runtime = t;
    }
  }
  return { title, id, channel, runtime, kind: 'video' };
}
```

Video URL: `https://www.youtube.com/watch?v=${id}`.

#### `videoRenderer` (older shape, still appears occasionally)

```js
function parseVideoRenderer(vr) {
  return {
    title: vr.title?.runs?.[0]?.text || vr.title?.simpleText || '',
    id: vr.videoId || '',
    channel: vr.ownerText?.runs?.map(r => r.text).join(', ')
      || vr.longBylineText?.runs?.map(r => r.text).filter(t => t.trim() && t !== ' and ').join(' and ')
      || '',
    runtime: vr.lengthText?.simpleText || '',
    kind: 'video'
  };
}
```

#### `reelShelfRenderer` (a horizontal shelf of Shorts)

A single `reelShelfRenderer` represents *all* of that day's Shorts in one shelf. Expand it into multiple items:

```js
function parseReelShelf(rs) {
  return (rs.items || []).map(i => {
    const v = i.shortsLockupViewModel;
    if (!v) return null;
    const id = v.onTap?.innertubeCommand?.reelWatchEndpoint?.videoId
      || v.entityId?.split('/').pop()
      || '';
    return {
      title: v.overlayMetadata?.primaryText?.content || '',
      id,
      channel: v.overlayMetadata?.secondaryText?.content || '(Shorts)',
      runtime: 'Short',
      kind: 'short'
    };
  }).filter(Boolean);
}
```

Shorts URL: `https://www.youtube.com/shorts/${id}` (not `/watch?v=`).

### 5. Apply the date window

Resolve each section header to a real date based on today:

- `"Today"` → today
- `"Yesterday"` → today − 1
- Weekday name (`"Monday"` … `"Sunday"`) → the most recent past occurrence of that weekday. YouTube only uses weekday names for days 2–7 ago, so this is always within the last week.
- Full date (`"Apr 25"`, `"Jan 3"`) → that calendar date in the most recent past year.

Sections come in document order, newest first. Walk them and keep those whose date is `>= today − (days − 1)`. Stop walking once you hit an older section.

### 6. Fallback: DOM scrape (only if `ytInitialData` is unavailable)

This should be rare. If after spoofing visibility and waiting ~15s `window.ytInitialData` is still undefined or empty, fall back to scrolling the page and reading the rendered DOM. Use `javascript_tool` to scroll the last `ytd-item-section-renderer` into view, wait, and repeat until two consecutive scrolls produce no new sections. Then read items from `yt-lockup-view-model` (with `ytd-video-renderer` and `ytd-reel-shelf-renderer` as alternates) — the per-element schema mirrors the JSON described in section 4.

### 7. Format the response

Present the result grouped by section, with the section label as a small header and each video as a single line containing title, channel, and runtime. Wrap the title in a Markdown link to the YouTube URL so the user can click through. Keep view counts out of the main list — they're rarely what the user cares about — but offer to include them if they ask. Example shape:

```
**Today**
1. [How to Make Claude Code Your AI Engineering Team](https://www.youtube.com/watch?v=...) — Y Combinator (21:50)
2. ...

**Yesterday**
1. ...
```

End with a brief one-sentence observation about the day's viewing if there's an obvious theme (e.g. "Heavy on AI coding workflows today"), but only if it's actually useful — don't manufacture a pattern out of three unrelated videos.

## Edge cases

- **Empty Today section**: YouTube still renders a `Today` heading even if the user hasn't watched anything yet today. Treat sections with zero items as empty rather than erroring.
- **Shorts vs. long-form**: The history page has tabs for All / Videos / Shorts / Podcasts / Music. Default to All. If the user asks for a specific category, click the matching tab via its accessibility ref before reading; the category filter applies to both the JSON payload and the DOM fallback.
- **History paused**: If the user has YouTube watch history paused, the page shows a "Your watch history is off" message and `sectionListRenderer.contents` is empty or contains a `messageRenderer` instead of `itemSectionRenderer` entries. Report this clearly — there is nothing to extract.
- **Non-English UI**: Section labels may be localized ("Hoy", "Aujourd'hui", localized weekday names). If literal string matching fails, fall back to positional reasoning — first section is today, second is yesterday, sections 3–7 are the prior weekdays in order — and mention the localization in the response so the user knows you weren't strict about matching.

## What not to do

- Don't attempt to delete entries, clear history, or click "Pause watch history" — those are destructive actions outside the scope of this skill, and they require explicit user confirmation through other channels, not this skill.
- Don't trust the rendered DOM as the source of truth for how much history exists. It virtualizes aggressively and frequently shows only 1–2 sections even when the underlying JSON has dozens. Always check `ytInitialData` first.
- Don't report only what the DOM happens to have rendered. An earlier version of this skill did exactly that and silently dropped weeks of relevant history from its answer.
