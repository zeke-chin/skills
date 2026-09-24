# Dynamic search: keep large results out of context

Keep large raw web payloads on disk and return only the evidence needed for the
task. Use this when using `--include-raw-content`, combining several queries,
or extracting multiple long pages. Do not use it for a simple lookup that a
normal `tvly search --json` can answer directly, and use
[research](research.md) for end-to-end cited synthesis.

Search and extract support capped keyless access. For missing CLI, keyless cap,
or authentication errors, follow [cli-setup.md](cli-setup.md#troubleshooting).

## Workflow

1. Search broadly without raw content and inspect titles, URLs, scores, and
   snippets.
2. Fetch full content only for the best sources.
3. When raw output could be large, save it with `-o` and filter the file before
   printing anything to the model context.
4. Preserve source URLs beside every extracted fact.

When the user restricts evidence to official or named domains, validate the
hostname of every selected URL during local filtering. `--include-domains`
narrows the search but is not proof that every returned result belongs to an
allowed host. If full-page extraction is unavailable, label conclusions as
search-snippet evidence instead of implying that the page body was verified.

Keep the process in one turn when the relevant sources and filters are already
known. Use another turn only when the first search changes what should be
extracted.

Create a unique temporary task directory before saving evidence so concurrent
agents do not overwrite one another. Python's `tempfile.mkdtemp()` is available
when `mktemp` is not permitted. Reuse that directory for all raw and filtered
artifacts from the task.

## Small result: filter a direct JSON response

For a small search response, a pipe is enough:

```bash
tvly search "query" --max-results 5 --json | python3 -c '
import json, sys
data = json.load(sys.stdin)
for result in data.get("results", []):
    score = result.get("score") or 0
    title = result.get("title") or ""
    print(f"[{score:.2f}] {title}")
    print(result.get("url", ""))
    print(result.get("content", "")[:300])
'
```

Do not discard stderr. Authentication failures, keyless-cap messages, and API
errors are actionable and must remain visible.

## Large result: save first, then filter

Use the CLI's file output so raw page content does not pass through the tool
response:

```bash
tvly search "query" \
  --include-raw-content markdown \
  --max-results 8 \
  --json \
  -o /tmp/tavily-search-results.json
```

Then print only bounded evidence:

```bash
python3 -c '
import json
from pathlib import Path

data = json.loads(Path("/tmp/tavily-search-results.json").read_text())
for result in data.get("results", []):
    title = result.get("title") or ""
    url = result.get("url") or ""
    print(f"## {title}")
    print(f"URL: {url}")
    print((result.get("raw_content") or result.get("content") or "")[:1200])
    print()
'
```

Adjust the filtering logic to the question. Prefer relevant paragraphs or
fields over fixed character slices when the target information is known. Aim
for roughly 150-600 tokens per source unless a table or code block genuinely
requires more.

## Targeted extraction

When search identifies the right URLs, extract only those pages:

```bash
tvly extract "https://example.com/article" \
  --json \
  -o /tmp/tavily-extract-results.json
```

For topic-focused pages, let Tavily reduce the response before local filtering:

```bash
tvly extract "https://example.com/docs" \
  --query "authentication API" \
  --chunks-per-source 3 \
  --json \
  -o /tmp/tavily-extract-results.json
```

## Multiple queries

For multi-angle research, run a small set of focused searches, deduplicate by
URL, and rank before extracting. Use `subprocess.run(..., capture_output=True,
text=True)` when orchestrating commands in Python. Check `returncode`; if a
command fails, surface its stderr and stop or retry deliberately. Never use a
blanket `except Exception: continue` that hides missing evidence.

## Response shapes

`tvly search --json` returns `query`, optional `answer`, `results`, and
`response_time`. Each result commonly contains `url`, `title`, `content`,
`score`, and optional `raw_content`.

`tvly extract --json` returns `results`, `failed_results`, and
`response_time`. Each successful result commonly contains `url`,
`raw_content`, and optional images.

Treat fields as optional and use `.get()` while filtering. Inspect
`failed_results` instead of assuming every requested URL succeeded.

Use `jq` only for short filters when Python is unavailable:

```bash
tvly search "query" --json | jq '[.results[] | {title, url, score, content}]'
```
