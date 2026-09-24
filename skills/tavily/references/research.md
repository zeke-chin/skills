# tavily research

AI-powered deep research that gathers sources, analyzes them, and produces a
cited report. Takes 30-120 seconds. For quick facts, use `tvly search` instead.

Requires authentication. Run the command directly; on an authentication error
(exit code 3), follow [cli-setup.md](cli-setup.md#troubleshooting).

## When to use

- You need comprehensive, multi-source analysis
- The user wants a comparison, market report, or literature review
- Quick searches are not enough — you need synthesis with citations

## Quick start

```bash
# Basic research (waits for completion)
tvly research "competitive landscape of AI code assistants"

# Pro model for comprehensive analysis
tvly research "electric vehicle market analysis" --model pro

# Stream results in real time
tvly research "AI agent frameworks comparison" --stream

# Save report to file
tvly research "fintech trends" --model pro -o fintech-report.json

# JSON output for agents
tvly research "quantum computing breakthroughs" --json
```

## Options

| Option | Description |
|--------|-------------|
| `--model` | `mini`, `pro`, or `auto` (default) |
| `--stream` | Stream results in real time |
| `--no-wait` | Return `request_id` immediately (async) |
| `--output-schema` | Path to JSON schema for structured output |
| `--citation-format` | `numbered`, `mla`, `apa`, `chicago` |
| `--poll-interval` | Seconds between checks (default: 10) |
| `--timeout` | Max wait seconds (default: 600) |
| `-o, --output` | Save the JSON response to a file |
| `--json` | Structured JSON output |

## Model selection

| Model | Use for | Speed |
|-------|---------|-------|
| `mini` | Single-topic, targeted research | ~30s |
| `pro` | Comprehensive multi-angle analysis | ~60-120s |
| `auto` | API chooses based on complexity | Varies |

Rule of thumb: "What does X do?" → `mini`. "X vs Y vs Z" or "best way to..." → `pro`.

## Async workflow

For long-running research, start and poll separately:

```bash
# Start without waiting
tvly research "topic" --no-wait --json    # returns request_id

# Check status
tvly research status <request_id> --json

# Wait for completion
tvly research poll <request_id> --json -o result.json
```

## Tips

- **Use `--stream`** to see progress during the 30-120 second run.
- **Use `--model pro`** for complex comparisons or multi-faceted topics.
- **Use `--output-schema`** to get structured JSON matching a custom schema.
- Read from stdin: `echo "query" | tvly research - --json`
