# Claude Code Skills

A collection of custom skills I've built for [Claude Code](https://claude.ai/code). Each skill teaches Claude a repeatable workflow — drop the `SKILL.md` into your Claude skills directory and it becomes available as a slash command.

---

## Skills

### `servicebus-gap-report`
**Compare two Azure Service Bus namespace exports and find what's missing.**

Given two XML files exported from [Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer), this skill identifies Topics and Subscriptions present in a source namespace (e.g. staging) but absent from a target namespace (e.g. production). Results are presented in two tables — topics entirely absent, and topics that exist but have missing subscriptions. Optionally posts the report to a Slack channel.

- [skills/servicebus-gap-report/SKILL.md](skills/servicebus-gap-report/SKILL.md)

### `readwise-inbox-triage`
**Walk through your Readwise Reader inbox one article at a time and decide what stays.**

Fetches unread articles from your Readwise Reader inbox, presents each one with a 2–3 sentence summary and a relevance take tailored to your interests, then moves documents based on your decision (keep, save for later, or archive). Runs in configurable batch sizes and wraps up with a session summary. Requires the Readwise MCP.

- [skills/readwise-inbox-triage/SKILL.md](skills/readwise-inbox-triage/SKILL.md)

### `youtube-history-browser`
**See what you've watched on YouTube today, yesterday, or over the past N days.**

Opens your YouTube watch history in your connected Chrome browser, parses the grouped-by-day video cards, and returns a clean linked list of titles, channels, and runtimes. Supports flexible look-back windows ("last 3 days", "this week") and closes with a one-line observation if there's an obvious theme to your viewing. Requires the Claude in Chrome MCP with an active YouTube session.

- [skills/youtube-history-browser/SKILL.md](skills/youtube-history-browser/SKILL.md)
