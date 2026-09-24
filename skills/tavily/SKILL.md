---
name: tavily
description: |
  Web search and URL content extraction via the Tavily CLI (`tvly`), with references for site mapping, crawling, cited deep research, large-result filtering, and CLI setup and authentication. Use when the user wants to search the web, look up current information or news, find sources, or read a webpage, or says "search for", "find me", "look up", "what's the latest on", "extract", "grab the content from", "read this page", or "get the page at https://". Also use when the user wants to map or crawl a site, download documentation, produce an in-depth research report or comparison with citations, or install or authenticate the Tavily CLI. Do not use for local file operations, git commands, deployments, or code editing tasks.
allowed-tools: Bash(tvly *)
---

# Tavily

Web search and content extraction through the `tvly` CLI. Results are JSON
optimized for LLM consumption. This file covers `search` and `extract`; other
capabilities live in `references/`.

## Before running

Run `tvly search` and `tvly extract` directly. Both support capped keyless
access, so do not look for an API key, check status, or authenticate before the
first request.

If a command fails, read [references/cli-setup.md](references/cli-setup.md)
and follow the matching troubleshooting row. The common cases:

| Symptom | Action |
|---------|--------|
| `tvly: command not found` | Install the CLI, then retry the original command |
| Keyless cap reached (interactive session) | `tvly login`, then retry the original command once |
| Keyless cap reached (unattended/CI) | Report the cap and authentication options; do not start an interactive flow |
| Exit code 3 / authentication error | Authenticate per the setup reference, then retry |

Do not start a second login immediately after guided setup has completed.

## Choose the command

Start simple and escalate only when needed.

| Need | Command | Where |
|------|---------|-------|
| Find pages or facts, no URL yet | `tvly search` | [Search](#search) |
| Read the content of known URLs | `tvly extract` | [Extract](#extract) |
| Know the site, not the exact page | `tvly map` | [references/map.md](references/map.md) |
| Many pages from a site section | `tvly crawl` | [references/crawl.md](references/crawl.md) |
| Multi-source report with citations | `tvly research` | [references/research.md](references/research.md) |
| Raw content, several queries, or many long pages | search/extract + local filtering | [references/dynamic-search.md](references/dynamic-search.md) |
| Install, authenticate, update, exit codes | `tvly init` / `tvly login` | [references/cli-setup.md](references/cli-setup.md) |

Map, crawl, and research require authentication; search and extract do not.

## Search

Web search returning ranked results with content snippets and relevance scores.

```bash
# Basic search
tvly search "your query" --json

# More results, higher precision
tvly search "quantum computing" --depth advanced --max-results 10 --json

# Recent news
tvly search "AI news" --time-range week --topic news --json

# Domain-filtered
tvly search "SEC filings" --include-domains sec.gov,reuters.com --json

# Include full page content in results
tvly search "react hooks tutorial" --include-raw-content --max-results 3 --json
```

| Option | Description |
|--------|-------------|
| `--depth` | `ultra-fast`, `fast`, `basic` (default), `advanced` |
| `--max-results` | Max results, 0-20 (default: 5) |
| `--topic` | `general` (default), `news`, `finance` |
| `--time-range` | `day`, `week`, `month`, `year` |
| `--start-date` / `--end-date` | Date bounds (YYYY-MM-DD) |
| `--include-domains` / `--exclude-domains` | Comma-separated domains |
| `--country` | Boost results from a country |
| `--include-answer` | Include AI answer (`basic` or `advanced`) |
| `--include-raw-content` | Include full page content (`markdown` or `text`) |
| `--include-images` | Include image results |
| `--include-image-descriptions` | Include AI image descriptions |
| `--chunks-per-source` | Chunks per source (`advanced`/`fast` depth only) |
| `-o, --output` | Save the JSON response to a file |
| `--json` | Structured JSON output |

| Depth | Speed | Relevance | Best for |
|-------|-------|-----------|----------|
| `ultra-fast` | Fastest | Lower | Real-time chat, autocomplete |
| `fast` | Fast | Good | Need chunks, latency matters |
| `basic` | Medium | High | General purpose (default) |
| `advanced` | Slower | Highest | Precision, specific facts |

Tips:

- **Keep queries under 400 characters** — write a search query, not a prompt.
- **Split complex questions into focused sub-queries.**
- **Use `--include-raw-content`** when full page text is needed; it saves a
  separate extract call. If the output could be large, follow
  [references/dynamic-search.md](references/dynamic-search.md) instead of
  printing it straight into context.
- **Use `--include-domains`** for trusted sources and `--time-range` for recency.
- **Verify identity-sensitive facts at the exact primary source.** For
  releases, versions, ownership, or similarly named projects, confirm the
  official repository or domain instead of trusting a generated answer or a
  package-name match alone.

## Extract

Clean markdown or text from one or more URLs, including JavaScript-rendered
pages.

```bash
# Single URL
tvly extract "https://example.com/article" --json

# Multiple URLs (max 20 per call)
tvly extract "https://example.com/page1" "https://example.com/page2" --json

# Query-focused extraction (relevant chunks only)
tvly extract "https://example.com/docs" --query "authentication API" --chunks-per-source 3 --json

# JS-heavy pages
tvly extract "https://app.example.com" --extract-depth advanced --json

# Save to file
tvly extract "https://example.com/article" -o article.json
```

| Option | Description |
|--------|-------------|
| `--query` | Rerank chunks by relevance to this query |
| `--chunks-per-source` | Chunks per URL (1-5, requires `--query`) |
| `--extract-depth` | `basic` (default) or `advanced` (JS pages, dynamic content, tables) |
| `--format` | `markdown` (default) or `text` |
| `--include-images` | Include image URLs |
| `--timeout` | Max wait time (1-60 seconds) |
| `-o, --output` | Save the JSON response to a file |
| `--json` | Structured JSON output |

Tips:

- **Try `basic` first**, fall back to `advanced` if content is missing.
- **Batch more than 20 URLs** into multiple calls.
- **Use `--query` + `--chunks-per-source`** to get relevant content instead of
  whole pages.
- **Inspect `failed_results` even after exit code 0.** A successful request can
  still return no extracted pages. Retry the affected URL with `advanced` when
  appropriate; otherwise report the per-URL failure instead of treating the
  request as complete.
- Skip extract when search results already carry the content via
  `--include-raw-content`.

## Shell usage

- **Always quote URLs** — the shell interprets `?` and `&`.
- **Prefer `--json`** in agent workflows; add `-o file.json` to keep large
  responses out of context.
- **Read the query from stdin with `-`**: `echo "query" | tvly search - --json`.
- Run `tvly <command> --help` for the full, current option list.

## References

Read these only when the task needs them:

- [references/cli-setup.md](references/cli-setup.md) — install, `tvly init`, authentication, keyless cap, updates, exit codes, troubleshooting
- [references/map.md](references/map.md) — discover URLs on a site; map + extract pattern
- [references/crawl.md](references/crawl.md) — bulk extraction from a site section, saving pages as markdown
- [references/research.md](references/research.md) — cited multi-source research, `mini`/`pro`/`auto` models, async polling
- [references/dynamic-search.md](references/dynamic-search.md) — save large results to disk, filter, deduplicate, keep context small
