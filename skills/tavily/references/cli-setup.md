# Tavily CLI setup and troubleshooting

Install, authenticate, update, and recover from errors in `tavily-cli`
(`tvly`). Search and extract support capped keyless access; map, crawl, and
research require authentication.

## Contents

- [Troubleshooting](#troubleshooting)
- [Install](#install)
- [Agent setup](#agent-setup)
- [Authentication](#authentication)
- [Keep the CLI current](#keep-the-cli-current)
- [Output and exit codes](#output-and-exit-codes)

## Troubleshooting

| Symptom | Action |
|---------|--------|
| `tvly: command not found` | [Install](#install), then retry the original command |
| Search or extract hits the keyless cap, interactive session | Run `tvly login` (browser OAuth), then retry the original command once |
| Search or extract hits the keyless cap, unattended or CI | Report the cap and the [authentication](#authentication) options; do not start an interactive flow |
| Map, crawl, or research returns exit code 3 / auth error | `tvly login`, or `tvly init --skip-skills` when guided verification is also useful; in unattended environments use a securely provided `TAVILY_API_KEY` or hand back to the user |
| Exit code 2 (bad input) | Check `tvly <command> --help`; make sure URLs are quoted |
| Exit code 4 (API or live-verification error) | Retry once; if it persists, report the error message verbatim |
| Extract exits 0 but a URL is missing | Inspect `failed_results`; retry that URL with `--extract-depth advanced` or report it |
| Unknown option or outdated behavior | `tvly update --check`, then `tvly update` |
| Unsure whether authenticated | `tvly --status --json` (only when needed, not before every command) |

Do not start a second login immediately after guided setup has completed.

## Install

```bash
curl -fsSL https://cli.tavily.com/install.sh | bash
```

Or manually: `uv tool install tavily-cli` / `pip install tavily-cli`.

## Agent setup

For agent setup, start keyless unless the user asks to sign in or the task
needs map, crawl, or research. If the installer did not already complete setup:

```bash
tvly init --skip-auth
```

This installs or updates the Tavily skills and verifies a live keyless search.
Do not look for an API key or authenticate before the first search or extract
request.

When authentication is requested or required, use guided setup:

```bash
tvly init

# Prefer to open the sign-in link yourself
tvly init --no-browser
```

`tvly init` reuses an existing credential, installs or updates the Tavily
skills bundled with that CLI release, and verifies a live search. Run
`tvly update` first when refreshing bundled skills. Use
`tvly init --skip-skills` when the skills are already installed and only
authentication or verification is needed.

## Authentication

Authenticate without full setup:

```bash
tvly login                         # browser OAuth (preferred when a user is present)
tvly login --no-browser            # print the sign-in link instead of opening it
tvly login --api-key tvly-YOUR_KEY
export TAVILY_API_KEY=tvly-YOUR_KEY
```

`--no-browser` only changes how the link is opened; the flow still returns to a
localhost callback on the machine running `tvly`. In remote sessions, make sure
that callback is reachable (SSH may require port forwarding).

In an unattended agent or CI environment, leave authentication to the user or
use a securely provided `TAVILY_API_KEY`, then resume the original command.

## Keep the CLI current

```bash
tvly update --check
tvly update
```

## Output and exit codes

Search, extract, crawl, map, and research support `--json`. Result-producing
commands support `-o` to save the JSON response; crawl also supports
`--output-dir` for one Markdown file per page. Setup, authentication, status,
and update commands expose `--json` where documented but do not support `-o`.

```bash
tvly search "react hooks" --json -o results.json
tvly extract "https://example.com/docs" -o docs.json
tvly crawl "https://docs.example.com" --output-dir ./docs/
```

| Exit code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | Setup or update failure |
| 2 | Bad input |
| 3 | Authentication error |
| 4 | API or live-verification error |

Run `tvly` without a subcommand for the interactive REPL (for humans, not
agents). Run `tvly --help` or `tvly <command> --help` for full option details.
