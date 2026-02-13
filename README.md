# Sully Context Engineering

Context engineering skills and workspace configurations for the Sully Intelligence agent (OpenClaw).

Built from first principles, informed by research from [Agent Skills for Context Engineering](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering) and OpenClaw's native compaction system.

## What's Here

### Skills
- **`skills/context-management/`** — Core context management skill that governs how Sully handles memory flushes, artifact tracking, context placement, and tool output discipline

### Configs
- **`configs/`** — OpenClaw configuration patches (compaction settings, pruning, thresholds)

## Philosophy

Optimize for **tokens-per-task**, not tokens-per-request. Accept slightly lower compression ratios for better quality retention. Structure forces preservation.

## References

- [Factory Research: Evaluating Context Compression for AI Agents (2025)](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering/tree/main/skills/context-compression)
- [Context Degradation Patterns](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering/tree/main/skills/context-degradation)
- [OpenClaw Compaction Docs](https://docs.openclaw.ai/concepts/compaction)
