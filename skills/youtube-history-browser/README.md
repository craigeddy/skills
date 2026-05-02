# youtube-history-browser

A Claude skill that opens your YouTube watch history in your connected Chrome browser, reads it, and reports what you've watched — optionally over a multi-day window.

Ask Claude things like *"what did I watch on YouTube today?"*, *"show me my YouTube history for the past 3 days"*, or *"find that video about Kubernetes I watched yesterday on YouTube"* and the skill takes care of navigating, parsing, and summarizing.

## Prerequisites

The skill drives Chrome through the **Claude in Chrome** MCP, so before installing you need:

1. The Claude in Chrome browser extension installed and connected to your Claude account.
2. An active Chrome session signed in to YouTube. The skill won't try to log you in — if YouTube shows the sign-in prompt, the skill will stop and tell you.

If you don't have YouTube history enabled (i.e. you've paused it), the skill will report that too rather than make something up.

## Installation

You have two options depending on how you use Claude.

### Option 1: Install the packaged `.skill` file (Cowork or claude.ai)

Download `youtube-history-browser.skill` from the release / repo. In Cowork or claude.ai, install it through the skills manager UI — typically by dragging the file into the chat or selecting it from the skills settings page.

### Option 2: Copy the folder (Claude Code)

Clone this repo (or download the `youtube-history-browser/` folder) and copy it into your Claude Code skills directory:

```bash
# macOS / Linux
cp -r youtube-history-browser ~/.claude/skills/

# Windows (PowerShell)
Copy-Item -Recurse youtube-history-browser $env:USERPROFILE\.claude\skills\
```

Restart any active Claude Code session afterward so the skill is picked up.

## Verifying the install

Start a fresh Claude session and ask something like *"what have I watched on YouTube today?"* If the skill is installed and Claude in Chrome is connected, Claude should open `https://www.youtube.com/feed/history` in your browser and come back with a grouped list of videos.

If nothing happens, check that:

- Claude in Chrome shows at least one connected browser.
- You're signed in to YouTube in that browser.
- The skill's folder name and `SKILL.md` are intact (the frontmatter `name:` field has to match the folder name).

## Usage notes

The skill takes one optional input — number of days to look back. It defaults to today only. Phrasings like "this week", "the past 3 days", or "since Monday" are translated into a day count automatically; if it's truly ambiguous, the skill will ask before running.

It deliberately refuses to delete, clear, or pause your watch history. Those are destructive actions that need explicit confirmation in chat, and they're outside the scope of this skill.

## Files

```
youtube-history-browser/
├── SKILL.md   # the skill itself — instructions Claude follows
└── README.md  # this file
```

## Updating

Edit `SKILL.md`, then either drop the updated folder back into `~/.claude/skills/`, or repackage with the `package_skill.py` script from the `skill-creator` skill and reinstall the resulting `.skill` file.
