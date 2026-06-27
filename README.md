# Ticker Tape
### A regulatory tracker skill

A Claude Skill that builds and maintains a sourced, dated timeline of a regulatory, legislative, or litigation development — designed to accumulate history across sessions, not just produce a one-time snapshot.

## The problem this solves

Asking an AI assistant "what's the status of X regulation" repeatedly tends to produce a fresh, disconnected summary every time — useful in the moment, but with no sense of what's actually *changed* since you last checked, and no durable record of the development's history. This skill is built to behave more like a real tracker: each check-in adds to a permanent timeline rather than replacing it.

## The architectural problem, and how this skill solves it

Claude can't carry memory between separate conversations, and it can't directly read what's already stored in a previously-rendered artifact — that storage lives entirely inside the artifact's own browser-side code, not in anything Claude can query as a tool. So a naive design ("just remember what I found last time") doesn't work across sessions.

This skill solves that by moving the memory into the artifact itself, rather than relying on Claude to have continuity it doesn't actually have:

1. **Every time the skill runs, Claude does fresh research** into the topic's current state — it never assumes it knows what was found before.
2. **Each development gets a stable, deterministic ID** built from its date and a short slug (e.g., `2026-04-01-compliance-deadline-passes`). The same real-world event always produces the same ID, regardless of which session discovered it.
3. **The artifact, on every mount, loads whatever is already stored** for that topic and merges the freshly-researched entries in by ID — anything already present is left alone; anything new is added and visually flagged.
4. **The merged result is written back to storage**, so the permanent record grows over time even though Claude itself never "remembers" anything between sessions.

The practical effect: you can ask about the same topic next week, next month, or from an entirely new conversation, and the tracker will correctly show only the genuinely new developments highlighted — without Claude needing any continuity at all. The intelligence is in the merge logic, not in Claude's memory.

## How it works

### 1. Topic identification
Each tracked subject gets a short, stable key (e.g., `section-1033`, `eu-ai-act`) — this is the lookup key the artifact uses in storage, so it has to stay consistent across sessions for the same topic.

### 2. Research with sourcing discipline
The skill pulls from authoritative primary sources rather than assembling status from scattered search snippets (which often mix different effective dates without making that obvious), and date-stamps every claim. It distinguishes settled fact from contested interpretation, particularly important for anything still in active litigation or rulemaking.

### 3. Stable ID assignment
Every development found gets a deterministic ID. This is the linchpin of the whole design — get this wrong (e.g., inconsistent date formatting) and the merge logic will either create duplicates or fail to recognize something as new.

### 4. Handoff to the artifact
Claude builds the data object — current status, the full entry list (not just what it thinks is new; the artifact figures that out), and any open/contested questions — and saves it as a `.jsx` artifact. The merge and persistence happen entirely in the artifact's own code from there.

### 5. Multi-topic support
The artifact can track several subjects at once, presented as tabs, and on mount it also checks storage for any previously-tracked topics that weren't part of this session's research, so a dashboard of everything you've ever tracked stays intact even if a given session only refreshes one of them.

## What's in this repo

```
SKILL.md                          → the full skill definition (Anthropic Skills format)
references/tracker-artifact.md    → the React component template, including the actual merge-on-mount logic
```

## A note on reliability

This design depends entirely on ID consistency. If Claude's slugging is inconsistent between sessions (e.g., `2026-04-01-deadline` one time and `2026-04-01-compliance-deadline` another), the merge will treat the same event as two different ones. The skill instructs Claude to favor a consistent date-plus-short-slug pattern, but this is worth spot-checking the first few times you use it on a new topic — if you see an obvious duplicate, it's almost certainly an ID mismatch rather than a storage failure.

## Disclaimer

This is a tracking and summarization tool, not legal advice. Regulatory and litigation status can shift quickly; anything time-sensitive (a deadline, a filing date) should be confirmed against the primary source before being relied on.
