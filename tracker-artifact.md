# Tracker Artifact Template

This file contains the React artifact template for the regulatory tracker dashboard. Unlike a typical artifact that's rebuilt fresh each time, this one is designed to **accumulate**: every time it's regenerated with new research, it merges that research against whatever is already stored, rather than replacing it.

## How the data flows

1. Claude researches the topic's current state (per `SKILL.md`) and builds a `DEFAULT_DATA` object with whatever it found, using stable IDs for each development.
2. The artifact mounts and, **for each topic in `DEFAULT_DATA`**, loads any existing stored data for that topic's key.
3. It merges: any entry whose ID already exists in storage is left as-is (not duplicated); any entry with a new ID is added and flagged as new-this-session for visual highlighting.
4. The merged result — not just the freshly-passed data — is written back to storage and rendered.

This means Claude never needs to know what was already tracked. The freshest research simply gets unioned into the permanent record by the artifact itself.

## Data structure

```
DEFAULT_DATA = {
  topics: {
    "topic-key": {
      label: "Human-readable name",            // e.g. "Section 1033 / Open Banking Rule"
      currentStatus: "One-line summary of where things stand",
      asOf: "2026-06-25",                       // date this status was confirmed accurate
      entries: [
        {
          id: "2026-04-01-compliance-deadline-passes",  // STABLE — same event = same id always
          date: "2026-04-01",
          headline: "Short headline of what happened",
          detail: "1-3 sentence plain-language explanation",
          sourceName: "Cozen O'Connor client alert",
          sourceUrl: "https://..."
        }
      ],
      openQuestions: [
        "Will the agency narrow the rule or vacate it entirely?"
      ]
    }
  }
}
```

### Mapping rules

- **id**: must be deterministic — built from the date plus a short slug, so the same real event always produces the same id across sessions. This is what makes merging safe.
- **entries**: include everything found in this session's research, not just what seems new. Let the artifact's merge logic determine novelty.
- **openQuestions**: short, currently-unresolved questions worth surfacing. These get unioned (deduplicated by exact text match) across sessions, not replaced.
- **currentStatus / asOf**: always reflects the latest research passed in — these two fields are not merged, they're overwritten, since "current status" should always show the freshest read.

---

## Full artifact code

```jsx
import { useState, useEffect } from "react";

const STORAGE_PREFIX = "regtracker:";

const DEFAULT_DATA = {
  // === REPLACE THIS OBJECT WITH REAL RESEARCH DATA ===
  topics: {}
};

function mergeTopic(stored, fresh) {
  const storedEntries = stored?.entries || [];
  const storedIds = new Set(storedEntries.map(e => e.id));
  const newOnes = fresh.entries.filter(e => !storedIds.has(e.id)).map(e => ({ ...e, isNew: true }));
  const keptOld = storedEntries.map(e => ({ ...e, isNew: false }));
  const mergedEntries = [...keptOld, ...newOnes].sort((a, b) => a.date.localeCompare(b.date));

  const storedQuestions = stored?.openQuestions || [];
  const mergedQuestions = Array.from(new Set([...storedQuestions, ...(fresh.openQuestions || [])]));

  return {
    label: fresh.label || stored?.label,
    currentStatus: fresh.currentStatus,
    asOf: fresh.asOf,
    entries: mergedEntries,
    openQuestions: mergedQuestions,
  };
}

export default function RegulatoryTracker() {
  const [topics, setTopics] = useState(null);
  const [activeTopic, setActiveTopic] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    (async () => {
      const merged = {};
      for (const [key, fresh] of Object.entries(DEFAULT_DATA.topics)) {
        let stored = null;
        try {
          const result = await window.storage.get(STORAGE_PREFIX + key);
          stored = result ? JSON.parse(result.value) : null;
        } catch {
          stored = null;
        }
        const mergedTopic = mergeTopic(stored, fresh);
        merged[key] = mergedTopic;
        try {
          await window.storage.set(STORAGE_PREFIX + key, JSON.stringify(mergedTopic));
        } catch {}
      }

      // Also pick up any previously-tracked topics not in this session's fresh data
      try {
        const listResult = await window.storage.list(STORAGE_PREFIX);
        if (listResult?.keys) {
          for (const fullKey of listResult.keys) {
            const key = fullKey.replace(STORAGE_PREFIX, "");
            if (!merged[key]) {
              try {
                const result = await window.storage.get(fullKey);
                if (result) merged[key] = JSON.parse(result.value);
              } catch {}
            }
          }
        }
      } catch {}

      setTopics(merged);
      const keys = Object.keys(merged);
      if (keys.length > 0) setActiveTopic(keys[0]);
      setLoading(false);
    })();
  }, []);

  if (loading) return <div style={{ display: "flex", alignItems: "center", justifyContent: "center", height: "100vh", fontFamily: "'DM Sans', sans-serif", color: "#94A3B8" }}>Loading tracker...</div>;
  if (!topics || !activeTopic) return <div style={{ padding: 40, fontFamily: "'DM Sans', sans-serif" }}>No tracked topics yet.</div>;

  const topicKeys = Object.keys(topics);
  const topic = topics[activeTopic];

  return (
    <div style={{ minHeight: "100vh", fontFamily: "'DM Sans', sans-serif", background: "linear-gradient(165deg, #0F172A 0%, #1E293B 100%)", color: "#E2E8F0", padding: "24px 16px" }}>
      <link href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&family=DM+Mono:wght@400;500&display=swap" rel="stylesheet" />

      <div style={{ maxWidth: 700, margin: "0 auto" }}>
        {topicKeys.length > 1 && (
          <div style={{ display: "flex", gap: 8, marginBottom: 20, overflowX: "auto" }}>
            {topicKeys.map(k => (
              <button key={k} onClick={() => setActiveTopic(k)} style={{
                fontSize: 12, fontFamily: "'DM Mono', monospace", padding: "6px 14px", borderRadius: 8, cursor: "pointer", whiteSpace: "nowrap",
                background: k === activeTopic ? "#334155" : "transparent",
                color: k === activeTopic ? "#E2E8F0" : "#64748B",
                border: k === activeTopic ? "1px solid #475569" : "1px solid #1E293B",
              }}>{topics[k].label || k}</button>
            ))}
          </div>
        )}

        <div style={{ fontSize: 11, fontFamily: "'DM Mono', monospace", color: "#64748B", letterSpacing: "0.1em", textTransform: "uppercase", marginBottom: 8 }}>Regulatory Tracker</div>
        <h1 style={{ fontSize: 26, fontWeight: 700, margin: "0 0 16px", letterSpacing: "-0.02em" }}>{topic.label}</h1>

        <div style={{ background: "#1E293B", border: "1px solid #334155", borderRadius: 12, padding: "16px 20px", marginBottom: 28 }}>
          <div style={{ fontSize: 11, fontFamily: "'DM Mono', monospace", color: "#64748B", marginBottom: 6, textTransform: "uppercase" }}>Current Status — as of {topic.asOf}</div>
          <div style={{ fontSize: 15, lineHeight: 1.5 }}>{topic.currentStatus}</div>
        </div>

        {topic.openQuestions && topic.openQuestions.length > 0 && (
          <div style={{ marginBottom: 28 }}>
            <div style={{ fontSize: 11, fontFamily: "'DM Mono', monospace", color: "#64748B", letterSpacing: "0.1em", textTransform: "uppercase", marginBottom: 10 }}>Open Questions</div>
            {topic.openQuestions.map((q, i) => (
              <div key={i} style={{ fontSize: 13, color: "#CBD5E1", padding: "8px 14px", marginBottom: 6, background: "#1E293B", border: "1px solid #334155", borderRadius: 8 }}>
                {"\u{2753}"} {q}
              </div>
            ))}
          </div>
        )}

        <div>
          <div style={{ fontSize: 11, fontFamily: "'DM Mono', monospace", color: "#64748B", letterSpacing: "0.1em", textTransform: "uppercase", marginBottom: 12 }}>Timeline ({topic.entries.length} entries)</div>
          {topic.entries.map(entry => (
            <div key={entry.id} style={{
              display: "flex", gap: 14, padding: "14px 16px", marginBottom: 8, borderRadius: 10,
              background: entry.isNew ? "#064E3B" : "#1E293B",
              border: entry.isNew ? "1px solid #047857" : "1px solid #334155",
            }}>
              <div style={{ minWidth: 90, fontSize: 12, fontFamily: "'DM Mono', monospace", color: entry.isNew ? "#6EE7B7" : "#94A3B8", paddingTop: 1 }}>{entry.date}</div>
              <div style={{ flex: 1 }}>
                <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                  <div style={{ fontSize: 14, fontWeight: 600, color: entry.isNew ? "#ECFDF5" : "#E2E8F0" }}>{entry.headline}</div>
                  {entry.isNew && <span style={{ fontSize: 9, fontFamily: "'DM Mono', monospace", padding: "2px 6px", borderRadius: 4, background: "#047857", color: "#ECFDF5" }}>NEW</span>}
                </div>
                <div style={{ fontSize: 13, color: "#94A3B8", marginTop: 4, lineHeight: 1.4 }}>{entry.detail}</div>
                {entry.sourceUrl && (
                  <a href={entry.sourceUrl} target="_blank" rel="noreferrer" style={{ fontSize: 11, color: "#38BDF8", textDecoration: "none", marginTop: 6, display: "inline-block" }}>
                    {entry.sourceName || "Source"} {"\u{2192}"}
                  </a>
                )}
              </div>
            </div>
          ))}
        </div>

        <div style={{ marginTop: 32, paddingTop: 16, borderTop: "1px solid #1E293B", textAlign: "center" }}>
          <div style={{ fontSize: 11, fontFamily: "'DM Mono', monospace", color: "#334155" }}>
            Entries marked NEW were added in this update. Ask Claude to check this topic again to refresh.
          </div>
        </div>
      </div>
    </div>
  );
}
```
