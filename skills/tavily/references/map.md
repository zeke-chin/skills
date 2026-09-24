# tavily map

Discover URLs on a website without extracting content. Faster than crawling.

Requires authentication. Run the command directly; on an authentication error
(exit code 3), follow [cli-setup.md](cli-setup.md#troubleshooting).

## When to use

- You need to find a specific subpage on a large site
- You want a list of URLs before deciding what to extract or crawl

## Quick start

```bash
# Discover URLs
tvly map "https://docs.example.com" --json

# Natural-language filtering
tvly map "https://docs.example.com" --instructions "Find API docs and guides" --json

# Filter by path
tvly map "https://example.com" --select-paths "/blog/.*" --limit 500 --json

# Deeper map
tvly map "https://example.com" --max-depth 3 --limit 200 --json
```

## Options

| Option | Description |
|--------|-------------|
| `--max-depth` | Levels deep (1-5, default: 1) |
| `--max-breadth` | Links per page (default: 20) |
| `--limit` | Max URLs to discover (default: 50) |
| `--instructions` | Natural language guidance for URL filtering |
| `--select-paths` | Comma-separated regex patterns to include |
| `--exclude-paths` | Comma-separated regex patterns to exclude |
| `--select-domains` | Comma-separated regex for domains to include |
| `--exclude-domains` | Comma-separated regex for domains to exclude |
| `--allow-external / --no-external` | Include external links |
| `--timeout` | Max wait (10-150 seconds) |
| `-o, --output` | Save the JSON response to a file |
| `--json` | Structured JSON output |

## Map + extract pattern

Find the right page with `map`, then `extract` it. This is often cheaper than
crawling a whole site:

```bash
# Step 1: find the authentication docs
tvly map "https://docs.example.com" --instructions "authentication" --json

# Step 2: extract the page you found
tvly extract "https://docs.example.com/api/authentication" --json
```

## Tips

- **Map is URL discovery only** — use `extract` or [crawl](crawl.md) for content.
- **Map + extract beats crawl** when you only need a few pages from a large site.
- **Use `--instructions`** for semantic filtering when path patterns are not enough.
