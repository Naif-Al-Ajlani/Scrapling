# Building Scrapling-shaped tools with an AI

The goal here is not "make an AI write a scraper." It is to extract the reusable shape from
Scrapling and turn it into instructions specific enough that a model produces a coherent library
instead of a pile of functions.

Models are good at writing any single file in this codebase. What they get wrong, left to
themselves, is the shape: they invent a new return type per transport, put validation in three
places, make everything async or nothing async, and leave the extension points implicit. The
material below is aimed at those failures.

## Part 1 — The shape worth reusing

### When it applies

Scrapling's structure fits any tool with all five of these properties. If your problem has fewer
than four, use something simpler.

1. **Several transports, one result.** Data can arrive by more than one mechanism (HTTP, browser,
   file, API, queue), and the differences between them are an implementation concern the user
   should not have to care about.
2. **A domain object worth investing in.** There is one thing the user manipulates repeatedly — a
   parsed document, a record, a table, a message — and most of the value is in how good that
   object is to work with.
3. **Hostile or unreliable sources.** Rate limits, blocks, auth expiry, flaky networks, changing
   schemas. Retry, backoff, and recovery are features, not error handling.
4. **Work that scales from one item to many.** The same user should be able to write a five-line
   script and a 10,000-item job without switching libraries.
5. **Long-running jobs.** Something that runs for hours needs resumability, progress visibility,
   and a fast development loop.

### The five layers

```
  Layer 5   Surfaces        library API, CLI, agent/MCP server, framework adapters
  Layer 4   Orchestration   queue, concurrency, dedup, throttle, checkpoints, hooks
  Layer 3   Facade          the small set of public names, two usage modes each
  Layer 2   Engines         one per transport, private, all producing the Layer-1 type
  Layer 1   Core            the domain object, its collection type, its value types,
                            validation, storage contract — no network, no I/O
```

Imports point downward only. The one permitted upward reference is the Layer-2 module that defines
the "result" type as a subclass of the Layer-1 domain object — that single seam is what makes
"one return type" possible.

### The order to build in

Build order matters more than any individual decision, because each step constrains the next.

1. **The domain object and its collection type.** Nothing else until this is good. Write the code
   you wish users would write, then make it work. In Scrapling this is `Selector` / `Selectors`.
2. **The value types it returns.** Subclass the builtins so chaining survives. `TextHandler(str)`,
   `TextHandlers(list)`.
3. **The result type**, as a subclass of the domain object, carrying transport metadata. Decide
   its fields now; every engine will have to fill them.
4. **The adapter factory.** One class, one method per foreign type, producing the result type.
   Write it before the second engine exists, so the first engine does not embed conversion logic.
5. **One engine, end to end.** The simplest transport. Make it work with real inputs.
6. **The config struct and validator.** Only once you have seen which parameters the first engine
   actually needs.
7. **The second engine.** This is the test: if adding it forces changes to Layers 1–4, the seams
   are in the wrong place. Fix them now, not later.
8. **The facade.** Public names, two usage modes (one-shot and session/context-managed), lazy
   imports.
9. **Orchestration.** Queue, dedup, concurrency, throttle, checkpoints, hooks. Only after the
   facade is stable.
10. **Surfaces.** CLI, agent server, framework adapters. These are consumers of Layer 3 and 4 and
    should require no changes beneath them.

An AI told to "build a scraping library" will produce steps 5 and 8 and skip 1–4. Giving it this
sequence is most of the work.

## Part 2 — Specifications to hand a model

These are written to be pasted, one at a time, with your domain substituted. Each ends with a
constraint list, which is the part that does the work.

### Spec A — the domain object

```
Design the central object for <TOOL>. It wraps <UNDERLYING REPRESENTATION> and is what
users interact with in every code path.

Requirements:
- One class plus a collection subclass of `list` that maps every query method over its
  members and flattens the results.
- Query methods return the collection type, never a bare list.
- Text-like results return a `str` subclass that adds domain methods and preserves its own
  type across every inherited string method.
- Any property that is expensive to compute is a lazy property memoised into a __slots__
  field, not computed in __init__. This object will be created in bulk.
- __slots__ on both classes.
- Compile every regex, query, and translation table at module level, not per call.
- Provide aliases for the method names of <THE INCUMBENT LIBRARY> so users can paste
  existing code.

Constraints:
- No network calls. No file I/O except an explicitly injected storage backend.
- No imports from any other layer of this package.
- Raise a clear TypeError if the object is used in a way its underlying representation
  cannot support, rather than letting a confusing lower-level error surface.
```

### Spec B — the result type and the adapter factory

```
Define `<Result>` as a subclass of <the domain object from Spec A>, adding the metadata
every transport produces: <list them>.

Then define a `<Result>Factory` class with one classmethod per foreign response type we
support, each returning a `<Result>`.

Requirements:
- Every engine returns `<Result>`. No engine constructs it directly; all go through the
  factory.
- The factory owns every normalisation quirk: encoding detection, missing fields,
  paginated or partial responses, history reconstruction.
- Each quirk gets a comment naming the upstream issue or behaviour that forces it.
- Adding a new transport must mean adding one classmethod and nothing else.

Constraints:
- The factory does not fetch anything. It only converts.
- If a foreign type cannot supply a field, document the fallback in the code rather than
  silently defaulting.
```

### Spec C — configuration

```
Define the configuration for <engine family> as a msgspec Struct (or pydantic model) with
Annotated bounds for mechanical constraints and __post_init__ for cross-field rules.

Requirements:
- Three configuration layers: class-level defaults, instance/session config, per-call
  overrides.
- Per-call override merging must validate ONLY the keys the caller passed and merge those
  over the session config. Validating the full model would overwrite session values with
  model defaults. Write a test for exactly this.
- Mutually exclusive options raise at construction with a message naming both options and
  what to do instead.
- Precompute the model's default values once at import and skip validating any parameter
  equal to its default.
- Expose the parameter surface as a TypedDict used with Unpack[...] on every **kwargs
  method, so editors keep completion.

Constraints:
- Validation happens once. Nothing downstream re-checks.
- Unknown keys raise, with the message listing the accepted keys.
```

### Spec D — the engine

```
Implement the <transport> engine for <TOOL>.

Requirements:
- A private module under the implementation package, not part of the public API.
- Sync and async variants with identical method names. If a body would exceed ~40 lines in
  both, extract the shared logic rather than duplicating it.
- Resource acquisition through a context manager that guarantees cleanup in `finally` and
  handles every acquisition mode the engine supports.
- A retry loop that classifies errors into retryable and fatal via one shared predicate
  used by every engine.
- Every user-supplied callback is wrapped in its own try/except that logs and continues.
  User code must not be able to end the operation.
- Errors in this library's own contract raise immediately. Do not swallow them.

Constraints:
- Returns `<Result>` via the factory only.
- No configuration parsing; it receives a validated config object.
- Any tuning constant (flags, timeouts, blocked types) lives in a constants module with a
  comment per entry explaining what it is for.
```

### Spec E — orchestration

```
Implement the job runner for <TOOL>: it takes work items, runs them with bounded
concurrency, dispatches results to user callbacks, and survives interruption.

Requirements:
- Priority queue with content-addressed deduplication. The fingerprint hashes the
  canonicalised identity of the item plus the route it takes, not the raw input.
- Global concurrency limit plus optional per-<partition key> limit.
- Checkpointing: snapshot pending items and the seen set, serialise atomically
  (temp file + rename), and degrade to a fresh start if the file is corrupt.
- Work items may carry user callbacks. Serialise them by NAME and rebind by getattr on
  load, since bound methods do not survive pickling.
- Two-stage interrupt: first signal drains in-flight work and checkpoints; second cancels.
  Checkpoint before cancelling in both cases.
- Adaptive rate limiting fed by observed latency and failure signals, floored by any rate
  the source declares and capped by a configured maximum. A failure must never lower the
  delay.
- Development mode: cache every external result to disk keyed by fingerprint and replay it
  on later runs, so users can iterate on their own logic without re-hitting the source.
- Statistics as a dataclass with a to_dict(), covering counts, timings, per-partition
  volumes, and per-outcome tallies.
- Two output modes: collect into a list subclass with export methods, or stream items as
  they are produced.

Constraints:
- Extension is by overriding hooks on one class. No middleware registry, no settings
  module, no plugin discovery.
- Name the hooks after moments, not mechanisms: on_start, on_error, on_item, on_close,
  is_<bad-condition>, retry_<bad-condition>.
- One exception in a user callback costs that item only.
```

### Spec F — the agent surface

```
Expose <TOOL> as an MCP server.

Requirements:
- Tools mirroring the library's escalation path, each with a bulk variant that takes a list
  so an agent makes one call for N items.
- Session lifecycle tools (open/close/list) for anything expensive to create, plus a
  session id parameter on the operation tools.
- Tool annotations declaring read-only, destructive, and open-world properties honestly.
- An instructions block that teaches the model: which tool to start with, when to escalate,
  how to narrow output to cut tokens, and to close what it opens.
- Sanitise every payload before it reaches the model: strip content hidden from human
  readers (CSS-hidden, aria-hidden, template, off-screen), zero-width Unicode, and control
  characters. Expose this as an on-by-default flag on the CLI too.
- Bearer-token auth for any HTTP transport, constant-time comparison, host allow-listing,
  and a startup warning when running unauthenticated.

Constraints:
- Default output format is the compact one (markdown or a projection), not raw.
- Clamp any user-supplied size to the validator's own bound; never trust the caller's count.
- The server holds no state a restart cannot rebuild.
```

### Spec G — the agent skill

```
Package the documentation for <TOOL> as an agent skill.

Requirements:
- SKILL.md with front matter: name, one-sentence description written as trigger conditions
  ("use when the user asks to ...", "when <other tool> fails"), version matching the
  library version, license.
- A decision section first: which entry point for which situation, and the escalation order
  when the cheap one fails.
- Runnable examples for each entry point, using real endpoints where possible.
- references/ mirroring the documentation tree, so the agent reads locally instead of
  searching.
- A guardrails section: authorisation, rate limits, what not to collect.
- An explicit note about anything an automated security review would flag, explaining what
  the mechanism actually is.
```

## Part 3 — Worked transposition

Take the same skeleton to a different problem: **a library for pulling records out of third-party
SaaS APIs at scale** — the kind of thing built once per company and never well. Call it Harvest.

The problem has all five properties: several transports (REST, GraphQL, bulk export, webhook
replay), one domain object (a record), hostile sources (rate limits, token expiry, 429s), scale
from one call to a full backfill, and long-running jobs.

### The mapping

| Scrapling | Harvest | Notes |
|:--|:--|:--|
| `Selector` | `Record` | Wraps a decoded JSON document; exposes JSONPath, key lookup, projection, coercion, and schema-tolerant access |
| `Selectors` | `Records` | `list` subclass; every query method maps and flattens |
| `TextHandler` / `TextHandlers` | `Field` / `Fields` | `str` subclass with `.as_date()`, `.as_decimal()`, `.re()`; preserves type through string methods |
| `AttributesHandler` | `Meta` | Read-only `MappingProxyType` over record metadata |
| `Response(Selector)` | `Page(Record)` | Adds `status`, `rate_limit_remaining`, `cursor`, `has_more`, `request`, `raw` |
| `ResponseFactory` | `PageFactory` | `from_rest`, `from_graphql`, `from_bulk_export`, `from_webhook` |
| `css()` / `xpath()` | `path()` / `pick()` | Two query dialects over the same object |
| Adaptive relocation | Schema drift recovery | Store a field's structural fingerprint (path, sibling keys, value type, sample); when the path stops resolving after an API version change, score every candidate path by similarity and recover the field |
| `find_similar()` | `find_similar_records()` | Given one record, find others at the same nesting depth with a similar key set |
| `FetcherSession` | `RestSession` | Auth, base URL, default headers, retry policy, token refresh |
| `DynamicSession` | `GraphQLSession` | Query documents, variables, fragment reuse, cost-aware batching |
| `StealthySession` | `BulkSession` | Submit a job, poll for completion, stream the result file |
| `PagePool` | `ConnectionPool` | Bounded concurrent in-flight requests with state tracking |
| `ProxyRotator` | `CredentialRotator` | Same shape exactly: list plus pluggable strategy; a strategy is `Callable[[List[Credential], int], Tuple[Credential, int]]` |
| `is_proxy_error` | `is_retryable` | One predicate classifying 429/5xx/token-expired across all transports |
| `PlaywrightConfig` | `SourceConfig` | msgspec Struct with `Annotated` bounds, `__post_init__` cross-field rules |
| `Spider` | `Pipeline` | Class attributes for concurrency, rate limits, checkpoint dir; `parse()` becomes `transform()` |
| `Scheduler` | `WorkQueue` | Priority heap plus fingerprint dedup, where the fingerprint is `{source_id, endpoint, canonicalised params, cursor}` |
| `SessionManager` | `SourceManager` | Routes work items to the right configured source by `sid` |
| `AutoThrottle` | `QuotaThrottle` | Latency-driven, but also reads `X-RateLimit-Remaining` / `Retry-After` and backs off before hitting zero |
| `RobotsTxtManager` | `QuotaManager` | Per-source declared limits fetched or configured once and cached |
| `CheckpointManager` | Same | Pickle pending items and the seen set; serialise callbacks by name |
| `ResponseCacheManager` | Same | Development mode: cache API responses, replay while iterating on `transform()` |
| `ItemList` | Same | `to_json`, `to_jsonl`, `to_csv`, `to_parquet` |
| `CrawlStats` | `RunStats` | Requests, retries, quota consumed per source, records emitted/dropped, bytes, per-status counts |
| `LinkExtractor` | `CursorExtractor` | Pulls the next-page token out of a response by any of the common conventions (Link header, `next_cursor`, `nextPageToken`, offset arithmetic) |
| `CrawlSpider` / `SitemapSpider` | `PaginatedPipeline` / `IncrementalPipeline` | Generic protocols: follow cursors until exhausted; sync everything changed since a watermark |
| `ShopifySpider` | `StripePipeline`, `HubSpotPipeline` | Platform templates, admitted under the same rule: uniform structure across many independent tenants |
| MCP server | Same | `list_sources`, `query`, `bulk_query`, `describe_schema`, session lifecycle |
| `_sanitize_for_ai` | Same, plus PII redaction | API records contain more personal data than web pages; redaction is the analogue of hidden-element stripping |
| Scrapy decorator | Airflow / dbt / Dagster adapters | Meet users inside the orchestrator they already run |

### What the mapping teaches

Three things transfer unchanged and are worth noticing:

**The rotator is identical.** Proxies and API credentials are the same problem — a pool of
interchangeable identities, rotated by a pluggable strategy, with failures classified by a shared
predicate. The `ProxyRotator` code would need renaming and nothing else.

**Adaptive relocation generalises further than it looks.** The technique is: store a structural
fingerprint of the thing you located, and when the direct path fails, score candidates by
similarity across several weak signals and take the best above a threshold. That works for DOM
elements, JSON paths, spreadsheet columns, log fields, and database columns after a migration.
Scrapling's `_StorageTools.element_to_dict` plus `__calculate_similarity_score` is a template for
all of them.

**Development mode is transport-agnostic.** Cache the expensive external result by request
fingerprint, replay it while the user iterates on their own logic. Nothing about it is specific to
HTTP.

### Three more domains, briefly

| | Domain object | Engines | Orchestration | Adaptive layer |
|:--|:--|:--|:--|:--|
| **Document extraction** | `Document` wrapping a page tree; `Documents` collection | PDF (text layer), PDF (OCR), DOCX, HTML, scanned image | Batch queue over a directory tree, per-worker concurrency, checkpoints per file | Locate a field by stored layout fingerprint when the template changes |
| **Log and telemetry ingestion** | `Event` wrapping a parsed record; `Events` collection | File tail, S3 objects, Kafka, HTTP push | Backpressure-aware pipeline, watermarks instead of a queue | Field recovery when a producer renames or moves a field |
| **Database migration and reconciliation** | `Row` / `Rows` | JDBC, native drivers, CDC stream, CSV dump | Chunked ranges with resumable offsets, per-table concurrency | Column matching across schema versions by name, type, and value-distribution similarity |

The engines and the orchestration change. Layers 1 and 3 barely do.

## Part 4 — Instructions for the model

A compact brief to prefix any of the specs in Part 2.

```
You are building a Python library, not a script. Apply these rules throughout.

Structure
- Five layers, imports pointing downward only: core → engines → facade → orchestration →
  surfaces. The only upward reference permitted is the result type subclassing the core
  domain object.
- The public API is under 15 names. Everything else is underscore-prefixed or in a private
  package.
- Resolve public names lazily through a module-level __getattr__ registry so importing the
  package does not import optional heavy dependencies.
- Optional dependencies go in extras. Every guarded import re-raises ModuleNotFoundError
  with the exact install command.

The domain object
- Every transport returns the same type. If you are about to write a second return type,
  stop and subclass instead.
- Query methods return a list subclass that maps the same methods over its members.
- Text results are a str subclass that survives its own inherited methods.
- Expensive properties are lazy and memoised. __init__ does the minimum that makes the
  object valid.
- __slots__ on anything created in bulk.

Configuration
- Validate once into a struct at the boundary. Nothing downstream re-checks.
- Three layers: class defaults, session config, per-call overrides. Per-call validation
  covers only the keys passed.
- Type **kwargs with TypedDict + Unpack.

Failure
- One predicate classifies retryable errors, shared by every engine.
- User callbacks are wrapped individually: log and continue.
- The library's own contract violations raise immediately.
- Anything resumable is written to a temp file and renamed. Corrupt state degrades to a
  fresh start.

Extension
- Pure behaviour is a callable. Stateful behaviour is an ABC with abstract methods.
- Users extend by overriding hooks on one class. No middleware registry, no plugin
  discovery, no settings module.

Comments
- Comment the reason, not the mechanism. Every workaround names the upstream issue. Every
  non-obvious constant says what it is for. Every suppressed lint check says why.

Do not
- Add a dependency for something under 50 lines.
- Make everything async, or nothing async. Pick per layer and be consistent.
- Build the CLI or agent surface before the library API is stable.
- Ship a single-source template in a general-purpose library.
```

## Part 5 — Reviewing what the model produced

Fifteen checks. Each has a definite answer in the code, and each catches a specific failure that
models produce by default.

**Layers and boundaries**

1. Does `grep` show any import from a lower layer to a higher one? There should be exactly one
   upward reference, and it should be the result type's base class.
2. Does importing the package pull in the heavy optional dependencies? Check with
   `python -X importtime -c "import <pkg>"`.
3. Is the public API under 15 names, and does `__all__` match what the docs promise?

**The domain object**

4. Is there exactly one result type, or did a second transport grow its own? This is the most
   common failure.
5. Do query methods return the collection type, so `.a().b()` chains?
6. Does the str subclass survive `.strip()`, `.lower()`, `.replace()`? Test it directly.
7. Is anything expensive computed in `__init__`? Instantiate 10,000 objects and time it.

**Configuration**

8. Are the same defaults defined in more than one place?
9. Does passing one per-call override silently reset the others to model defaults? Write this
   test; it fails more often than not.
10. Do editors show completion for `**kwargs` methods?

**Failure handling**

11. Does an exception raised inside a user callback end the run? It should not.
12. Does a corrupt checkpoint or cache file crash startup? It should not.
13. Is there one retry-classification predicate, or one per engine?

**Surfaces**

14. Does content bound for a model get sanitised, and is that on by default?
15. Does the MCP instructions block teach a policy, or is it an empty string?

**Two smell tests that are not checks**

Read the five-line example from the README and ask whether it is the code you would actually
write. If it needs a setup block first, the facade is wrong.

Then read three comments at random. If they say what the line does rather than why it exists, the
model wrote comments to satisfy a rubric and the reasons are not recorded anywhere.
