---
name: ticker-tape
description: >
  Use this skill whenever the user wants to track, monitor, or get updates on the status of
  a regulation, bill, rule, or piece of litigation over time — as opposed to a one-time
  explanation. Trigger phrases include: "track this regulation", "what's the status of [X]",
  "update my tracker on [X]", "monitor [bill/rule/case]", "anything new on [X]", "what's changed
  with [X] law", "check in on [X]", or any request implying the user wants to follow a legal/
  regulatory development across multiple sessions rather than get a single static summary.
  Always use this skill when the goal is ongoing tracking, not just a one-time explainer.
---

# Ticker Tape
### (a regulatory tracker skill)

Builds and maintains a sourced, dated timeline of a regulatory, legislative, or litigation development — designed so each check-in only requires fresh research into what might be new, while a persistent artifact accumulates the full history across sessions without ever needing to be told what was already found before.

## Architecture note — read this before building anything

Claude cannot directly read what's already stored in a previous instance of this skill's artifact. Browser storage tied to an artifact only exists inside that rendered artifact's own code — it isn't something Claude can query from outside, and a new conversation has no memory of a prior one's research.

So the design does **not** rely on Claude remembering prior state. Instead:

1. Each time this skill runs, Claude does fresh research into the topic's current status (Step 2 below), with no assumption about what was found previously.
2. Claude assigns each development a **stable ID** (Step 3) so the same real-world event always produces the same ID, regardless of which session discovered it.
3. Claude hands this freshly-researched data to the tracker artifact.
4. The **artifact's own code** — not Claude — loads whatever is already stored for this topic, merges the new data in by ID (skipping anything already present, adding anything new), and writes the merged result back to storage.

This means the persistence lives in the artifact's merge logic, not in Claude's memory. It also means re-running this skill on a topic that hasn't changed is safe — nothing gets duplicated, because the IDs are stable.

---

## Step 1: Identify the topic and a stable topic key

Ask for or confirm a short, stable topic key — this is the lookup key the artifact uses in storage, so it needs to stay consistent across sessions for the same topic (e.g., `section-1033`, `eu-ai-act`, `ca-sb-53`). If the user just names a topic conversationally ("track the EU AI Act"), derive a reasonable slug yourself and confirm it: "I'll track this under `eu-ai-act` — let me know if you'd rather use a different key."

If the user is checking in on something they've tracked before, ask them to reuse the same key, or default to the obvious slug for that topic so a returning user lands on the same dashboard entry.

---

## Step 2: Research current status

Research the topic's current state using the sourcing discipline below — this matters more here than almost anywhere else, because the entire point of a tracker is accuracy about what's *currently* true, not what used to be true.

- **Pull from a single authoritative source per claim** (the agency's own site, the official bill-tracking site, a court's docket, the Federal Register) rather than assembling status from scattered search snippets, which often mix different dates without making that obvious.
- **Date-stamp everything.** Every status claim should carry the date it was true as of, not just "currently."
- **Distinguish settled fact from contested interpretation.** If parties disagree about what something means or what happens next, present that as live disagreement, not as one side's framing.
- **Look specifically for what changed**, not just the current snapshot — court orders, agency statements, new comment periods, amendments, or a change in any party's litigation position since whatever the topic's last known developments were.

---

## Step 3: Assign stable IDs to each development

For each distinct development (a ruling, a filing, an agency statement, a deadline passing, a new comment period, etc.), construct an ID from the date and a short slug of what happened — e.g., `2026-04-01-compliance-deadline-passes` or `2025-08-22-anprm-published`. This ID must be deterministic: if Claude (in this session or a future one) finds the same real-world event again, it should produce the same ID, so the artifact's merge logic recognizes it as already-known rather than creating a duplicate.

Do this for every development you find in Step 2, not just ones that seem new — the artifact handles deduplication; Claude's job is just to make the IDs consistent.

---

## Step 4: Build the data and hand it to the artifact

Read `references/tracker-artifact.md` for the full component template and data structure. Populate `DEFAULT_DATA` with:

- The topic key, a human-readable label, and a one-line current-status summary with its "as of" date
- The full list of entries you found in Step 2–3 (not just ones you think are new — the artifact will figure out what's actually new by comparing against what it already has stored)
- Any open/contested questions worth tracking (e.g., "will the agency narrow or fully vacate the rule")
- Sources for each entry (name + URL)

Save this as a `.jsx` artifact. The artifact's own code handles the merge against existing storage — Claude does not need to (and cannot) know what was already there.

---

## Step 5: Present a short conversational summary

After generating the artifact, give a brief plain-language summary of the current state — lead with what's changed or notable, not a full re-statement of history the user has likely already seen. Since Claude can't know for certain what the user saw last time, phrase this appropriately: describe what you found, rather than asserting confidently that something is "new since your last check." The artifact itself will visually distinguish genuinely new entries (ones that weren't already in storage) once rendered.

---

## Step 6: Disclaimer

Include a short note that this is a tracking summary, not legal advice, and that given how quickly procedural status can shift, anything time-sensitive (a compliance deadline, a filing deadline) should be confirmed against the primary source before being relied on.

---

## Notes & edge cases

- **Multiple topics**: a user may want to track several things over time. Use a distinct topic key per subject; the artifact supports multiple topics as tabs and can list all currently-tracked topics.
- **If research turns up nothing new**: still regenerate the artifact with the current findings. Re-running with identical data is safe and won't create duplicates or change anything visible — the merge is idempotent.
- **If the user disputes that something is "new"**: trust the artifact's merge logic over your own assumption. If an entry shows as new but the user recalls seeing it before, the most likely explanation is the ID didn't match a previous entry exactly (e.g., a slightly different date format) — not that the artifact's storage failed. Don't overwrite their account of history; just note the entries may have IDs that didn't dedupe cleanly.
- **Avoid presenting unsettled outcomes as settled.** Especially in live litigation or rulemaking, "currently" can mean something different two weeks later — emphasize the as-of date prominently rather than letting status read as permanent.
