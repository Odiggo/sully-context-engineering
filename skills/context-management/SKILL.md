---
name: context-management
description: Governs pre-compaction memory flushes, artifact tracking, context placement strategy, and tool output discipline. Activated automatically before compaction and during session boot.
---

# Context Management

This skill defines how Sully Intelligence manages context throughout long-running sessions to prevent information loss during compaction and minimize degradation from attention mechanics.

## When to Activate

- Before any compaction (auto or manual) — structure the memory flush
- At session boot — verify artifact index is loaded
- When tool outputs exceed ~2000 tokens — apply compression before they enter context
- When session context estimate exceeds 100k tokens — begin proactive offloading

## Core Principles

### 1. Optimize for Tokens-Per-Task, Not Tokens-Per-Request

Aggressive compression saves tokens per request but causes re-fetching, re-exploration, and wasted effort. The right metric is total tokens consumed from task start to completion. A strategy saving 0.5% more tokens but causing 20% more re-fetching costs more overall.

### 2. Structure Forces Preservation

Unstructured summaries silently drop information. Explicit sections act as checklists the summarizer must populate. If there's a "Files Modified" section, the summarizer must fill it — preventing the artifact trail loss that plagues all compression methods.

### 3. Critical Info at Context Edges

The lost-in-middle phenomenon causes 10-40% lower recall for information in the center of context. Place critical operational info (current task, key decisions, next steps) at the beginning or end. Push stable reference material to the middle.

### 4. Minimize Context Surface Area

Every token in context competes for attention. Don't dump raw tool outputs when a summary will do. Don't load reference files that aren't needed this session. The model must attend to everything provided — irrelevant tokens create distraction.

---

## Structured Memory Flush Format

When performing a pre-compaction memory flush (or any session-end capture), write to `memory/YYYY-MM-DD.md` using these explicit sections:

```markdown
## Session Intent
[What the user is trying to accomplish — the overarching goal]

## Work Completed
- [Specific task 1 with outcome]
- [Specific task 2 with outcome]

## Files Modified/Created
- path/to/file.ext: [What changed and why]
- path/to/new-file.ext: [Created — purpose]

## Tools Used & Key Outputs
- [Tool name]: [What was queried/done] → [Key result summary]
- [Tool name]: [What was queried/done] → [Key result summary]

## Decisions Made
- [Decision]: [Reasoning / alternatives considered]

## Open Questions / Blockers
- [Anything unresolved]

## Next Steps
1. [Immediate next action]
2. [Follow-up action]
```

### Rules
- Every section MUST be populated. If nothing applies, write "None this session."
- File paths must be exact (no "some config file" — use `config/redis.ts`)
- Error messages preserved verbatim when relevant
- Decision reasoning includes alternatives that were rejected and why

---

## Artifact Index

Maintain a persistent file at `memory/artifact-index.md` that tracks cross-session state:

```markdown
# Artifact Index
Last updated: YYYY-MM-DD

## Active Branches / PRs
- branch-name → PR #123: [status, what it does]

## Running Cron Jobs
- job-id: [schedule, purpose, last status]

## Recent File Modifications (Last 7 Days)
- YYYY-MM-DD: path/to/file — [what changed]

## Active Projects / Investigations
- [Project name]: [current status, what's blocking]
```

Update this file:
- After any PR creation/merge
- After any cron job change
- At end of any substantial work session
- During heartbeat maintenance (every few days)

This file survives compaction because it's on disk, not in context.

---

## Tool Output Discipline

### Large Outputs (>2000 tokens estimated)
When a tool returns a large result:
1. Extract the relevant facts needed to answer the user's question
2. Summarize into a compact response
3. If the full output might be needed later, write it to a workspace file and reference the path

### Specific Tool Policies
- **GitHub diffs**: Summarize what changed per file. Only quote specific lines if discussing a bug.
- **Metabase/BigQuery**: Extract the key numbers and trends. Don't paste raw result sets.
- **Grafana logs**: Pull the relevant log lines, not the full search results.
- **Firestore documents**: Extract relevant fields, not entire documents.
- **Langfuse traces**: Summarize the flow, only show specific observation I/O when debugging.

### Exception
When the user explicitly asks to "show me the raw output" or "paste the full diff," do so. The discipline applies to default behavior, not explicit requests.

---

## Context Placement Strategy

### Session Boot Order (Attention-Optimized)
Files loaded at boot should be ordered for attention mechanics:

1. **SOUL.md** — Identity and behavior (high attention, beginning position)
2. **Current task context** — What we're working on right now
3. **MEMORY.md** — Long-term curated memory
4. **TOOLS.md** — Operational reference
5. **Stable reference** — AGENTS.md, IDENTITY.md (rarely changes, middle position OK)

### Within Files
- Put "what to do" at the top, "background info" at the bottom
- Use clear section headers — they help the model navigate structure
- Remove stale sections rather than leaving them as noise

---

## Context Poisoning Prevention

### Detection Signals
- Output quality degrades on previously-successful tasks
- Tool calls using wrong parameters that were "learned" from a bad earlier result
- Hallucinations that persist despite correction

### Recovery
- If poisoning is suspected, use `/compact` with explicit instructions: "Discard [specific incorrect context]. The correct state is [X]."
- For severe cases, `/new` and reconstruct from disk files (memory/, workspace files)
- Document the poisoning event in daily memory so future sessions learn from it

---

## Degradation Monitoring

### Warning Signs (Act When You See These)
- Session token estimate >120k tokens — consider proactive `/compact`
- Same information requested twice in one session — possible compaction lost it
- Tool output from >10 turns ago being referenced — it's probably already pruned
- Contradictory statements in your own responses — possible context clash

### Thresholds (Claude Opus 4.6, 200k window)
- **Comfortable**: <80k tokens — full performance
- **Attentive**: 80-120k tokens — be aware of placement, consider offloading
- **Compact soon**: 120-160k tokens — proactive memory flush + compact
- **Emergency**: >160k — auto-compaction imminent, flush NOW

---

## Integration with OpenClaw

This skill works with OpenClaw's native systems:
- **Compaction mode `safeguard`**: Chunked summarization (already configured)
- **Memory flush**: Pre-compaction silent turn (already enabled, this skill improves the format)
- **Session pruning**: `cache-ttl` mode trims old tool results (complementary — this skill prevents bloat from entering in the first place)
- **Reserve tokens floor**: 20k (sufficient for the structured flush format)

No OpenClaw code changes required. This skill operates entirely through agent behavior and workspace files.
