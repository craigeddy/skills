# Claude Code Skills

A collection of custom skills I've built for [Claude Code](https://claude.ai/code). Each skill teaches Claude a repeatable workflow — drop the `SKILL.md` into your Claude skills directory and it becomes available as a slash command.

---

## Skills

### `servicebus-gap-report`
**Compare two Azure Service Bus namespace exports and find what's missing.**

Given two XML files exported from [Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer), this skill identifies Topics and Subscriptions present in a source namespace (e.g. staging) but absent from a target namespace (e.g. production). Results are presented in two tables — topics entirely absent, and topics that exist but have missing subscriptions. Optionally posts the report to a Slack channel.

- [skills/servicebus-gap-report/SKILL.md](skills/servicebus-gap-report/SKILL.md)
