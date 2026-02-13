# OpenClaw Compaction Configuration Notes

## Current Config (as of 2026-02-12)
```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard"
      }
    }
  }
}
```

## What Each Setting Does

### `compaction.mode`
- `"default"` — Standard summarization
- `"safeguard"` — Chunked summarization for very long histories (CURRENT — keep this)

### `compaction.reserveTokensFloor`
- Default: 20000 tokens
- Minimum headroom reserved for prompts + next output
- Current: using default (20k) — sufficient for structured memory flush format
- Consider bumping to 25k if the structured flush is getting cut off

### `compaction.memoryFlush`
- `enabled`: true (default) — runs silent pre-compaction turn
- `softThresholdTokens`: 4000 — how far below compaction limit to trigger flush
- `prompt`: the message sent to the agent for the flush turn
- `systemPrompt`: extra system prompt for the flush turn

## Recommended Changes

### No immediate config changes needed
The current `safeguard` mode + default memory flush settings work well. The improvement comes from the agent's behavior (structured flush format, artifact tracking) not config changes.

### Future consideration
If structured flushes are getting truncated or compaction fires before flush completes:
```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "reserveTokensFloor": 25000
      }
    }
  }
}
```
