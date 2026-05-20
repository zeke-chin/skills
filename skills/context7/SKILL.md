---
name: context7
description: "Fetch up-to-date documentation and code examples for any library or framework using the ctx7 CLI. Use this skill whenever the user asks about a library, framework, SDK, API, CLI tool, or cloud service — even well-known ones like React, Next.js, Prisma, Express, Tailwind, Django, or Spring Boot. This includes API syntax, configuration, version migration, library-specific debugging, setup instructions, and CLI tool usage. Use even when you think you know the answer — your training data may not reflect recent changes. TRIGGER when: user asks how to use a library/framework/API; user mentions a specific library version or migration; user needs code examples for a library; user asks about SDK usage or configuration. Do NOT use for: refactoring, writing scripts from scratch, debugging business logic, code review, or general programming concepts."
---

# Context7 — Up-to-date Library Documentation

Use the `ctx7` CLI to fetch current documentation and code examples for any programming library or framework. This skill replaces manual web searches for library docs — it pulls directly from official documentation sources.

## Core Workflow

This skill provides **three core capabilities**. For any other `ctx7` functionality, require explicit user instruction first.

### 1. Authentication

```bash
ctx7 login          # Log in to Context7
ctx7 logout         # Log out
ctx7 whoami         # Check current login status
```

If a documentation query fails with an auth error, prompt the user to run `ctx7 login` first.

### 2. Resolve Library ID

Before querying docs, resolve the library name to a Context7-compatible library ID:

```bash
ctx7 library <name> [query]
```

**Examples:**
```bash
ctx7 library react "how to use hooks"
ctx7 library next.js "app router"
ctx7 library prisma "schema relations"
```

The output gives you an exact library ID (e.g., `/facebook/react`, `/vercel/next.js`). Use this ID in the next step.

### 3. Query Documentation

Use the library ID to fetch relevant docs and code examples:

```bash
ctx7 docs <libraryId> <query>
```

**Examples:**
```bash
ctx7 docs /facebook/react "useEffect cleanup function"
ctx7 docs /vercel/next.js "middleware redirect"
ctx7 docs /prisma/prisma "upsert with relations"
```

## Usage Pattern

For any library-related question:

1. **Resolve** the library ID: `ctx7 library <name> "<topic>"`
2. **Query** the docs: `ctx7 docs <library-id> "<specific question>"`
3. **Use** the returned documentation to answer the user's question

If you already know the library ID from a previous call in the same conversation, skip step 1.

## When NOT to Use

- Simple questions you can answer confidently from training data
- General programming concepts not tied to a specific library
- Debugging business logic or application code
- Code review or refactoring tasks

## Other Capabilities

`ctx7` has additional features (skills management, setup/install to other AI clients, etc.). Only use these if the user explicitly requests them. Do not invoke `ctx7 skills`, `ctx7 setup`, `ctx7 remove`, or other subcommands unless the user gives a clear, specific instruction to do so.
