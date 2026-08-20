# Scrapling codebase analysis

A study of Scrapling v0.4.14 (`scrapling/`, ~9,000 lines of library code plus ~9,000 lines of
tests): what each file does, which design decisions hold it together, and how to hand those
decisions to an AI so it can build a comparably shaped tool for a different problem.

Written against commit `cb00f1b` on `main`. Line references point at that revision.

## The documents

| File | What it covers |
|:--|:--|
| [`01-architecture-and-files.md`](01-architecture-and-files.md) | Layer map and a file-by-file walkthrough of every module, test directory, and repo-level file |
| [`02-design-patterns.md`](02-design-patterns.md) | The 30-odd recurring techniques, each with the code that demonstrates it and the cost it carries |
| [`03-building-similar-tools.md`](03-building-similar-tools.md) | The transferable skeleton, the spec templates to feed a model, a worked transposition to a different domain, and a review checklist |

## Summary of what the code does

Scrapling is four products in one package, stacked so that each layer only depends on the one
beneath it:

1. **A parser.** `Selector` wraps an `lxml` element tree and exposes CSS, XPath,
   BeautifulSoup-style filters, text search, and regex search behind one object. Results are
   `Selectors` (a list subclass) and `TextHandler` (a str subclass), so every call chains.
2. **Fetchers.** Three transports — TLS-impersonating HTTP via `curl_cffi`, Chromium via
   Playwright, and hardened Chromium via Patchright — all returning the same `Response` object,
   which is itself a `Selector`. Learning the parser means you have already learned every
   fetcher's return value.
3. **A spider framework.** A Scrapy-shaped async crawler on `anyio`: priority queue with request
   fingerprinting, per-domain concurrency limits, robots.txt compliance, autothrottle,
   pause/resume checkpoints, and a response cache for iterating on parse logic offline.
4. **Agent and operator surfaces.** An MCP server, a CLI, an IPython shell, a Scrapy decorator,
   and a packaged agent skill that ships the documentation to coding agents.

## What makes it worth studying

The parser and the crawler are not novel on their own; parsel and Scrapy have covered that ground
for a decade. Three things here are less common and carry most of the transferable lessons.

**One return type across every transport.** `Response` subclasses `Selector`, and every engine
funnels through a single `ResponseFactory` that adapts foreign response objects
(`curl_cffi.Response`, two flavours of Playwright `Response`, and Scrapy's `Response`) into it.
The user-visible API surface stays constant as the transport changes underneath.

**Adaptive relocation.** The parser can store an element's structural fingerprint and, after the
site's markup changes and the original selector returns nothing, rescan the page and score every
candidate element by similarity to recover the same element. It is a fallback path baked into
`css()` and `xpath()` rather than a separate tool.

**The AI surface is designed, not bolted on.** Content served to a model is stripped of
CSS-hidden elements, `aria-hidden` nodes, `<template>` blocks, zero-width characters, and control
characters, because those are the standard carriers for prompt injection in scraped pages. The
MCP server ships tool annotations and an instruction block that teaches the model an escalation
ladder. The repo also ships a versioned agent skill and an MCP registry manifest.

The third document turns all of this into a build procedure rather than a description.
