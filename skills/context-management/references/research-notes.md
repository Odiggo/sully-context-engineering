# Research Notes — Context Engineering

Sources: Factory Research (Dec 2025), RULER Benchmark, Netflix AI Summit 2025, Peking University MCE paper (2026).

## Key Findings Applied

### Compression Method Comparison (Factory Research, 36k+ messages)
| Method | Compression | Quality | Best For |
|--------|-------------|---------|----------|
| Anchored Iterative | 98.6% | 3.70/5.0 | Long sessions, file tracking |
| Regenerative | 98.7% | 3.44/5.0 | Clear phase boundaries |
| Opaque | 99.3% | 3.35/5.0 | Max compression, short sessions |

**Decision**: Use anchored iterative (structured sections) for Sully. 0.7% more tokens retained buys 0.35 quality points. Worth it when re-fetching costs matter.

### Artifact Trail Is Universally Weak
All methods scored 2.2-2.5/5.0 on artifact trail (file tracking). This requires specialized handling beyond general summarization — hence the artifact-index.md approach.

### Lost-in-Middle (Empirical)
- 10-40% lower recall for middle-positioned information
- U-shaped attention curve: beginning and end get reliable attention
- BOS token creates "attention sink" consuming budget
- Shuffled (incoherent) context sometimes outperforms coherent context — coherent text creates false associations

### Degradation Thresholds (Claude Opus 4.5 era, likely similar for 4.6)
- Degradation onset: ~100k tokens
- Severe degradation: ~180k tokens
- Context window: 200k tokens
- Implication: meaningful degradation can begin at 50% window utilization

### Context Distraction
- Even a single irrelevant document reduces performance (step function, not linear)
- Models cannot "skip" irrelevant context — must attend to everything
- Mitigation: don't load it in the first place (just-in-time retrieval > preloading)

### The Four-Bucket Framework
1. **Write**: Save context outside the window (scratchpads, files)
2. **Select**: Pull only relevant context in (retrieval, filtering)
3. **Compress**: Reduce tokens while preserving information (summarization)
4. **Isolate**: Split context across sub-agents/sessions

### Context Poisoning
- Enters via: tool output errors, incorrect retrieved docs, hallucinated summaries
- Compounds through feedback loops — each decision references poisoned content
- Recovery: truncate to before poisoning, explicit correction, or fresh session

## What We Chose NOT to Implement (and Why)

- **Opaque compression**: Sacrifices interpretability. Can't verify what was preserved. Bad for an agent that needs to debug its own context.
- **Probe-based evaluation pipeline**: Formal LLM-as-judge scoring per compaction is overkill. Would add latency and cost to every compaction. Monitor informally instead.
- **Full regenerative summaries**: Loses details across repeated compression cycles due to full regeneration rather than incremental merging. Anchored iterative is strictly better for long-running sessions.
- **KV-cache optimization**: OpenClaw handles this at the infrastructure level (session pruning + Anthropic cache TTL). Not an agent-level concern.
