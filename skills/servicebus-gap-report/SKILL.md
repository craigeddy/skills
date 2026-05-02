---
name: servicebus-gap-report
description: >
  Compare two Azure Service Bus namespace export XML files (from Service Bus Explorer)
  to identify Topics and Subscriptions present in a source namespace but missing from
  a target namespace. Use this skill whenever the user wants to compare, diff, or sync
  Service Bus topics/subscriptions between environments (e.g. staging vs production),
  or uploads Service Bus Explorer XML files and asks what's missing, different, or needs
  to be created. Also handles optionally posting the results to a Slack channel.
---

# Service Bus Gap Report

Compares two Service Bus Explorer XML exports and reports what Topics and Subscriptions
exist in the source but are missing from the target.

## Inputs

The user provides two XML files exported from Azure Service Bus Explorer:
- **Source** — the reference namespace (e.g. staging)
- **Target** — the namespace to check for gaps (e.g. production)

If not clear from context, ask which file is source and which is target.

## Default Exclusion Rules

Apply these exclusions unless the user overrides them:

**Exclude Topics** whose `Path` contains (case-insensitive):
- `dev`
- `masstransit/`

**Exclude Subscriptions** whose `Name` contains (case-insensitive):
- `craig`
- `test`
- `dev`
- `catchall`
- `catch-all`

Always confirm the exclusion rules with the user before running — they may want to
add or remove exclusions.

## Parsing Notes

- XML namespace: `http://schemas.microsoft.com/servicebusexplorer`
- Topic path is in `<Path>` child element
- Subscription name is in `<Name>` child element (not an XML attribute)
- Subscriptions are nested inside their parent `<Topic>` element

## Python Analysis Script

Use `bash_tool` to run this analysis inline. Adapt paths to match the actual uploaded file
locations (typically `/mnt/user-data/uploads/<filename>`).

```python
import xml.etree.ElementTree as ET

NS = 'http://schemas.microsoft.com/servicebusexplorer'

def tag(local):
    return f'{{{NS}}}{local}'

def get_text(elem, local):
    child = elem.find(tag(local))
    return child.text if child is not None else None

def is_excluded_sub(name, sub_exclusions):
    if name is None:
        return True
    lower = name.lower()
    return any(x in lower for x in sub_exclusions)

def is_excluded_topic(path, topic_exclusions):
    if path is None:
        return True
    lower = path.lower()
    return any(x in lower for x in topic_exclusions)

def parse_topics(file_path, topic_exclusions, sub_exclusions):
    tree = ET.parse(file_path)
    root = tree.getroot()
    topics = {}
    for topic in root.iter(tag('Topic')):
        topic_path = get_text(topic, 'Path')
        if not topic_path or is_excluded_topic(topic_path, topic_exclusions):
            continue
        subs = set()
        for sub in topic.iter(tag('Subscription')):
            name = get_text(sub, 'Name')
            if not is_excluded_sub(name, sub_exclusions):
                subs.add(name)
        topics[topic_path] = subs
    return topics

# --- Configure paths and exclusions ---
SOURCE_PATH = '/mnt/user-data/uploads/Staging-Events-Ferretly_Topics.xml'
TARGET_PATH = '/mnt/user-data/uploads/Events-Ferretly_Topics.xml'

TOPIC_EXCLUSIONS = ['dev', 'masstransit/']
SUB_EXCLUSIONS   = ['craig', 'test', 'dev', 'catchall', 'catch-all']

source = parse_topics(SOURCE_PATH, TOPIC_EXCLUSIONS, SUB_EXCLUSIONS)
target = parse_topics(TARGET_PATH, TOPIC_EXCLUSIONS, SUB_EXCLUSIONS)

# Find gaps
missing_subs   = {}   # topic exists in target, but missing some subs
absent_topics  = {}   # topic doesn't exist in target at all

for topic_path, source_subs in sorted(source.items()):
    if not source_subs:
        continue
    target_subs = target.get(topic_path, set())
    gap = sorted(source_subs - target_subs)
    if gap:
        if topic_path not in target:
            absent_topics[topic_path] = gap
        else:
            missing_subs[topic_path] = gap

# Print results
print(f"Source: {len(source)} topics, {sum(len(v) for v in source.values())} subs")
print(f"Target: {len(target)} topics, {sum(len(v) for v in target.values())} subs\n")

print(f"Topics entirely absent from target: {len(absent_topics)}")
for t, subs in sorted(absent_topics.items()):
    print(f"  {t}")
    for s in subs:
        print(f"    - {s}")

print(f"\nTopics with missing subscriptions: {len(missing_subs)}")
for t, subs in sorted(missing_subs.items()):
    print(f"  {t}")
    for s in subs:
        print(f"    - {s}")
```

## Output Format

Present results in two sections:

### 1. Topics Entirely Absent from Target
Topics that don't exist in the target at all. These require both Topic and Subscription creation.
Format as a table: `| Topic | Subscription(s) |`

### 2. Topics Existing in Target — Missing Subscriptions
Topics that exist but are missing one or more subscriptions.
Format as a table: `| Topic | Missing Subscription(s) |`

If there are no gaps, say so clearly.

### Caveats
After presenting results, flag any subscriptions whose names suggest they may be
environment-specific (e.g. containing `local`, `staging`, `local-`). Note these
should be verified before creating them in production.

## Optional: Post to Slack

If the user wants to post results to Slack:

1. Use `slack_search_channels` to resolve the channel name to an ID
2. Show the user the exact message text and ask for confirmation before sending
3. Use `slack_send_message` to post

Format the Slack message using Slack markdown (`*bold*`, `_italic_`, backticks for
topic/subscription names, bullet points with `•`).
