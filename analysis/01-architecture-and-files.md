# Architecture and file-by-file walkthrough

## The layer map

```
                       surfaces
   cli.py   core/shell.py   core/ai.py   integrations/scrapy.py
        |          |             |               |
        +----------+------+------+---------------+
                          |
                     spiders/            (orchestration: queue, engine, sessions,
                          |               checkpoints, throttle, templates)
                          |
                     fetchers/           (public facade: Fetcher, DynamicFetcher,
                          |               StealthyFetcher + their session classes)
                          |
                     engines/            (private implementation: curl_cffi engine,
                          |               two Playwright engines, toolbelt)
                          |
                     parser.py           (Selector / Selectors)
                          |
                      core/              (types, custom types, storage, translator,
                                          utils — no network, no I/O beyond SQLite)
```

Imports only ever point downward. `core/` never imports `engines/`; `parser.py` never imports
`fetchers/`. The one deliberate exception is `engines/toolbelt/custom.py`, which imports
`parser.Selector` to define `Response` as a subclass — that is the seam where fetching meets
parsing, and it is the only place the two halves touch.

`engines/` is private by convention: everything under it is either underscore-prefixed
(`_browsers/`) or reachable only through `fetchers/`. The package's public names are re-exported
from `scrapling/__init__.py` and `scrapling/fetchers/__init__.py`, both of which use lazy imports
so that `import scrapling` does not pull in Playwright or curl_cffi.

## Package root

### `scrapling/__init__.py`

Version, author, and a lazy import registry. `_LAZY_IMPORTS` maps eight public names to
`(module_path, class_name)` pairs, and module-level `__getattr__` (PEP 562) resolves them on first
access. `TYPE_CHECKING` imports keep static analysers happy without paying the runtime cost, and
`__dir__` is overridden so autocomplete still works.

The point is startup time. `from scrapling import Selector` should not import Playwright.

### `scrapling/py.typed`

Marker file (PEP 561) telling type checkers the package ships inline annotations.

### `scrapling/parser.py` (1,381 lines)

The centre of gravity. Defines `Selector`, `Selectors`, and the backwards-compatibility aliases
`Adaptor` / `Adaptors` at the end of the file.

`Selector.__init__` (line 80) takes either `content` (str/bytes) or `root` (an existing
`HtmlElement`). Content goes through `lxml.html.HTMLParser` with `recover=True`,
`huge_tree=True`, `remove_blank_text=True`, and comment/CDATA stripping controlled by flags. Null
bytes are stripped before parsing, and empty input becomes `<html/>` rather than an error.

The class deliberately wraps rather than inherits `lxml.html.HtmlElement`, and the docstring
explains why: `HtmlElement` is not picklable, which breaks reference-holding operations. Both
`Selector.__getstate__` and `Selectors.__getstate__` raise `TypeError` outright rather than
letting a confusing lxml proxy error surface later.

Key regions of the file:

- **Text-node handling** (line 195). `_is_text_node()` checks `issubclass(type(element),
  _ElementUnicodeResult)` — faster than checking lxml's three boolean properties. Every public
  method starts with this check, because `::text` and `::attr()` selections return string results
  that must not be treated as elements.
- **Lazy properties** (lines 258–341). `tag`, `text`, and `attrib` are computed on first access
  and cached into `__slots__` fields. The comment above them says computing these eagerly was the
  original design and moving them to lazy properties "made the library performance test sky rocket
  multiple times faster".
- **Conversion helpers** (lines 206–248). `__element_convertor` and `__elements_convertor` build
  child `Selector` objects while propagating url, encoding, adaptive flag, storage handle, and
  parser flags. The bulk version hoists those seven values into locals before the generator, to
  avoid repeated attribute lookups per element.
- **Traversal** (lines 384–460). `parent`, `children`, `siblings`, `below_elements`, `next`,
  `previous`, `iterancestors`, `find_ancestor`, `path`. `next`/`previous` skip `HtmlComment` nodes
  in a loop.
- **Selection** (lines 566–695). `css()` translates to XPath via the cached translator and
  delegates to `xpath()`. When adaptive mode is on and the selector contains a comma, it splits on
  commas with `cssselect.parse` first, so each sub-selector gets its own adaptive identifier.
- **Filters** (lines 696–905). `find_all()` accepts tag names, iterables of tag names, dicts of
  attributes, compiled regex patterns, callables, and keyword attributes — including the
  `class_` / `for_` aliases for Python reserved words. It builds a CSS selector string from tags
  and attributes and runs one query, then applies regex and callable filters to the results, on
  the reasoning that "it's easier and faster to build a selector than traversing the tree".
- **Adaptive machinery** (lines 508–564 and 875–905). `relocate()` walks every element in the
  tree and scores it against a stored fingerprint dict; `__calculate_similarity_score` averages
  `SequenceMatcher` ratios over tag, text, attribute keys and values, individual `class`/`id`/
  `href`/`src` attributes, the ancestor tag path, parent name/attributes/text, and sibling tags.
  `save()` and `retrieve()` proxy to the storage backend.
- **Similarity search** (lines 1013–1077). `find_similar()` narrows candidates first by
  grandparent/parent/self tag path and exact ancestor depth (`count(ancestor::*) = N`), then
  scores attribute similarity. Ignores `href` and `src` by default because they vary between
  otherwise-identical items.
- **Text search** (lines 1078–1190). `find_by_text()` and `find_by_regex()` iterate elements that
  have non-whitespace text (via a precompiled XPath) and short-circuit on `first_match`.

`Selectors` (line 1192) is a `List[Selector]` subclass whose `css`, `xpath`, `re`, `re_first`
map over members and flatten, plus `search`, `filter`, `first`, `last`, `length`, and the
parsel-compatible `get` / `getall` / `extract` / `extract_first`.

Module-level precompiled XPath objects (`_find_all_elements`, `_find_all_elements_with_spaces`,
`_find_all_text_nodes`) are compiled once at import.

### `scrapling/cli.py` (698 lines)

Click-based CLI with four command groups: `install`, `shell`, `extract`, and `mcp`. Entry points
are declared in `pyproject.toml` as `scrapling` and `scrapling-mcp`.

The Click import is wrapped in a try/except that raises a `ModuleNotFoundError` naming the
install extra to use — the pattern for every optional dependency in the codebase.

`_common_http_options`, `_common_browser_options`, and `_data_options` are decorator factories
that apply a shared list of `option(...)` decorators, so the four HTTP commands and the two
browser commands stay in sync without repetition. `__http_command` is the shared body.

`--ai-targeted` is worth noting: it sets `main_content_only=True` for the extraction path and
defaults `block_ads=True` for browser commands. The agent skill instructs models to always pass
it.

## `scrapling/core/` — primitives with no network dependency

### `core/_types.py`

Re-exports the typing surface used across the package plus a few domain aliases: `ProxyType`,
`SUPPORTED_HTTP_METHODS`, `SelectorWaitStates`, `PageLoadStates`, `extraction_types`,
`FollowRedirects`, and `SetCookieParam` (copied from Playwright's internal structure so the
package does not import Playwright just for a type).

Centralising imports here means every module does `from scrapling.core._types import ...` rather
than mixing `typing` and `typing_extensions` imports across 40 files.

### `core/custom_types.py` (345 lines)

`TextHandler(str)` overrides every string method that returns a string so the return value is a
`TextHandler` again, keeping chains alive. On top of that it adds `clean()` (whitespace
normalisation via a `str.maketrans` table plus a precompiled consecutive-space regex, optionally
unescaping HTML entities), `json()`, `sort()`, `re()`, and `re_first()`.

`re()` has three overloads: `check_match=True` returns a bool for cheap existence tests,
otherwise it returns `TextHandlers`. `replace_entities`, `clean_match`, and `case_sensitive`
control the details. Note the workaround comment in `json()`: orjson has an open issue with `str`
subclasses, so the value is passed through `str()` first.

`TextHandlers(List[TextHandler])` maps `re`/`re_first` across members with flattening and slices
back into itself.

`AttributesHandler(Mapping)` wraps element attributes in a `MappingProxyType` — described in the
docstring as "the fastest read-only mapping type" — converting string values into `TextHandler`
on construction. Adds `search_values()` (generator over exact or partial value matches) and
`json_string`.

### `core/storage.py` (155 lines)

`StorageSystemMixin(ABC)` defines the contract for adaptive storage: `save(element, identifier)`
and `retrieve(identifier)`, plus two cached helpers — `_get_base_url()` (uses `tld` to extract the
registrable domain so data is scoped per site) and `_get_hash()` (sha256 with the byte length
appended to reduce collisions).

`SQLiteStorageSystem` is the default implementation, decorated with `@lru_cache(1, typed=True)`
**on the class**, which turns it into a per-argument-set singleton. It opens SQLite with
`check_same_thread=False`, sets `PRAGMA journal_mode=WAL`, and guards writes with an `RLock`. The
schema is a single `storage` table with `UNIQUE (url, identifier)` and `INSERT OR REPLACE`
semantics.

`docs/development/adaptive_storage_system.md` documents this as a user extension point and shows
a Redis implementation.

### `core/translator.py` (134 lines)

An adapted copy of parsel's translator. `TranslatorMixin.xpath_pseudo_element` dispatches on the
pseudo-element name to `xpath_{name}_simple_pseudo_element` or
`xpath_{name}_functional_pseudo_element`, which is how `::text` and `::attr(name)` get added to
cssselect. `XPathExpr` carries two extra flags (`textnode`, `attribute`) through joins and appends
`/text()` or `/@attr` in `__str__`.

The module docstring states the reason plainly: matching parsel/Scrapy selector syntax so users do
not have to learn a new dialect, unlike BeautifulSoup's soupsieve. `css_to_xpath` is wrapped in
`@lru_cache(maxsize=256)`.

### `core/mixins.py`

`SelectorsGeneration` generates CSS and XPath selectors for an element by walking up to the root,
using `#id` when available and `:nth-of-type(n)` when a tag repeats among siblings. Inspired by
Firefox devtools' `css-logic.js`. A commented-out block explains that class names were removed
from generated selectors because some sites share identical classes across unrelated elements.

The comment at the top of the class notes it is a mixin whose methods reach into `Selector`
attributes through `self`.

### `core/utils/_utils.py`

- `setup_logger()` under `@lru_cache(1)`, the singleton-by-cache trick again.
- `LoggerProxy` + a `ContextVar` holding the current logger, with `set_logger`/`reset_logger`
  returning and consuming a token. This is how a spider swaps in its own named logger for the
  duration of a crawl without touching global state (`spider.py`, `__run`).
- `flatten`, `_is_iterable`, `clean_spaces` (cached, 128 entries).
- `_StorageTools.element_to_dict()` — the element fingerprint format used by both the storage
  system and the similarity scorer. Captures tag, cleaned attributes, stripped text, the ancestor
  tag path (built recursively), parent name/attributes/text, sibling tags, and child tags.

### `core/utils/_shell.py`

`_CookieParser` (wraps `http.cookies.SimpleCookie`) and `_ParseHeaders`, which splits a list of
`"Key: Value"` strings into a header dict and a cookie dict. Shared by the CLI and the curl
parser.

### `core/shell.py` (680 lines)

Three things live here.

`CurlParser` builds an `argparse` parser covering the curl flags that Chrome DevTools emits
(`-X`, `-H`, `-d`, `--data-raw`, `-b`, `--compressed`, `-L`, `-x`, ...) and converts a pasted curl
command into a `Request` namedtuple, or executes it directly through `Fetcher` via
`convert2fetcher`. `NoExitArgumentParser` overrides `error` and `exit` so a bad command raises
instead of killing the shell.

`CustomShell` embeds IPython with a prepared namespace: wrapped `get`/`post`/`put`/`delete`/
`fetch`/`stealthy_fetch` functions that record the last response into `page` / `response` /
`pages`, plus `uncurl`, `curl2fetcher`, `view` (opens the page in a browser via a temp file), and
`help`. `create_wrapper` sets `__signature__` explicitly so IPython introspection and
autocompletion work — the signatures come from `_shell_signatures.py` because `Unpack[TypedDict]`
kwargs are not introspectable at runtime.

`Convertor` handles output conversion for the CLI and MCP server:
- `_convert_to_markdown` via markdownify.
- `_strip_noise_tags` drops `script`, `style`, `noscript`, `svg` from a deep copy of the tree.
- `_sanitize_for_ai` drops elements matched by `_HIDDEN_XPATH` (inline `display:none`,
  `visibility:hidden`, `opacity:0`, zero font-size/height/width, `aria-hidden="true"`, and
  `<template>`), then strips zero-width Unicode characters and C0 control characters from every
  text and tail. This is prompt-injection defence, and it is the reason `--ai-targeted` exists.
- `_extract_content` yields markdown, html, or text depending on the requested type;
  `write_content_to_file` picks the type from the file extension.

### `core/_shell_signatures.py`

Runtime-accessible parameter dictionaries mirroring the TypedDicts in
`engines/_browsers/_types.py`, used by `_unpack_signature` to rebuild real `inspect.Signature`
objects for the shell. A maintenance duplication accepted in exchange for a usable REPL.

### `core/ai.py` (1,042 lines)

The MCP server. `ScraplingMCPServer` holds a dict of live browser sessions keyed by ID and exposes
ten tools:

| Tool | Backing |
|:--|:--|
| `open_session`, `close_session`, `list_sessions` | `AsyncDynamicSession` / `AsyncStealthySession` lifecycle |
| `get`, `bulk_get` | `FetcherSession` |
| `fetch`, `bulk_fetch` | `AsyncDynamicSession` |
| `stealthy_fetch`, `bulk_stealthy_fetch` | `AsyncStealthySession` |
| `screenshot` | either browser session, returns an `Image` content block |

Design decisions worth copying:

- Every response goes through `_translate_response`, which runs `Convertor._extract_content` and
  then strips control characters again before building the `ResponseModel`.
- Bulk variants exist so an agent makes one tool call for N URLs. `_page_pool_size` clamps the
  page pool to the validator's own upper bound (50) rather than trusting the batch length.
- Tools carry `ToolAnnotations`: fetch tools are `read_only_hint=True, open_world_hint=True`;
  session tools are `destructive_hint=False`; `list_sessions` is closed-world.
- The `instructions` string is a numbered policy that teaches the model the escalation ladder
  (`get` → `fetch` → `stealthy_fetch`), tells it to close sessions, and tells it to use
  `css_selector` to cut token spend.
- `_StaticTokenVerifier` uses `hmac.compare_digest` for bearer auth on the streamable-HTTP
  transport, `_transport_security` enables DNS-rebinding protection when allowed hosts are given,
  and `serve()` logs a warning when HTTP is used without a token.

## `scrapling/engines/` — transports

### `engines/constants.py`

Four tuples of Chromium flags: `EXTRA_RESOURCES` (resource types dropped when
`disable_resources` is on), `HARMFUL_ARGS` (passed to `ignore_default_args`, including
`--enable-automation`), `DEFAULT_ARGS` (speed and noise reduction), and `STEALTH_ARGS` (a long
list with a link to peter.sh's switch reference). Each entry that is non-obvious carries a
comment explaining what it defeats.

### `engines/static.py` (786 lines)

The curl_cffi HTTP engine, structured as three classes plus two clients.

`_ConfigurationLogic(ABC)` holds every default (impersonate target, proxies, timeout, headers,
retries, redirect policy, TLS verification, HTTP/3 flag, selector config, proxy rotator) and two
methods:

- `_merge_request_args()` layers per-call kwargs over session defaults, resolves a random browser
  when `impersonate` is a list, and passes unrecognised keys straight through to curl_cffi while
  skipping a known set of browser-only parameters. That skip-set is how a `Request` carrying
  browser kwargs can be routed to an HTTP session in a spider without error.
- `_headers_job()` sets a Google referer, and when `impersonate` is off, fills in generated
  browser headers without overwriting anything the user supplied.

`_SyncSessionLogic` and `_ASyncSessionLogic` are near-identical twins implementing
`__enter__`/`__exit__` (or the async pair) and `_make_request`, which is the retry loop: pull a
proxy from the rotator if there is one, build args, call curl, hand the result to
`ResponseFactory.from_http_request`; on `CurlError`, classify with `is_proxy_error` and retry with
backoff.

`FetcherSession` is the public wrapper that can act as either a sync or async context manager,
building the matching logic class from its own `__slots__` on entry.

`FetcherClient` / `AsyncFetcherClient` (lines 769–786) are the sessionless mode. They subclass the
logic classes and set `__enter__`/`__exit__` to `None` and the session attribute to a `_NO_SESSION`
sentinel, so `_make_request` detects the sessionless case and creates a throwaway curl session per
call. The comments explain the reason: curl_cffi caches impersonation state across a session, and
does not support async requests without one. It works, but it is the least readable construct in
the codebase.

### `engines/_browsers/_types.py`

TypedDicts for every parameter surface: `RequestsSession`, `GetRequestParams`,
`DataRequestParams`, `PlaywrightSession`, `PlaywrightFetchParams`, `StealthSession`,
`StealthFetchParams`, plus the `ImpersonateType` alias. These feed `Unpack[...]` annotations on
methods that take `**kwargs`, which is what gives editors completion on kwargs-only APIs.

### `engines/_browsers/_validators.py` (252 lines)

msgspec-based configuration validation. `PlaywrightConfig` is a `Struct` with ~35 fields and
`Annotated` constraints (`PagesCount` is 1–50, `RetriesCount` is 1–10, `Seconds` is ≥ 0).
`__post_init__` does what msgspec cannot: checks callables are callable, rejects `proxy` combined
with `proxy_rotator`, normalises proxies through `construct_proxy_dict`, validates the CDP URL
scheme, validates that `init_script` and `executable_path` are absolute existing files, and merges
the ad-domain list when `block_ads` is set. `StealthConfig` extends it with four stealth flags and
bumps the timeout floor to 60s when `solve_cloudflare` is on.

Two performance details: `models_default_values` is computed once at import from
`__struct_fields__`/`__struct_defaults__`, and `_filter_defaults` drops any parameter equal to its
default before calling `convert`, so validation only runs on what the user actually changed.

`validate_fetch()` implements per-call override semantics: it starts from the session's config,
validates only the keys the caller passed, and merges those back — the comment explains it must
not let validated defaults overwrite session values.

### `engines/_browsers/_page.py`

`PageInfo` — a slotted, generic dataclass holding a Playwright page, a `PageState` literal
(`ready`/`busy`/`error`), and a URL, with `mark_busy()`/`mark_error()`. `PagePool` holds a list of
them under an `RLock` with a hard `max_pages` cap, and exposes `pages_count`, `busy_count`, and
`cleanup_error_pages()`.

### `engines/_browsers/_base.py` (577 lines)

`SyncSession` and `AsyncSession` hold the shared browser lifecycle: `start`/`close`, context
initialisation (init script, cookies), `_get_page` (creates a page, sets timeouts, extra headers,
and installs a route handler when resources or domains are blocked), `_wait_for_page_stability`
(load → domcontentloaded → networkidle, each optional), `_create_response_handler` (captures the
main-frame document response into a one-element list, and optionally XHR/fetch responses matching
a regex), and `_page_generator`.

`_page_generator` is the acquisition context manager and handles two modes. With a proxy, it
builds a fresh browser context for that proxy, yields a page from it, and closes the context in
`finally`. Without one, it takes a page from the pool against the persistent context and closes
the page in `finally`. The async version additionally waits (polling at 50ms, capped at 60s) for
pool capacity.

`BaseSessionMixin` builds the Playwright options. `__validate_routine__` seeds context options
(dark colour scheme — the comment says it defeats creepjs's `prefersLightColor` check — and 2x
device scale) and browser options (`DEFAULT_ARGS` plus `ignore_default_args=HARMFUL_ARGS`), then
validates into the requested config model. `__generate_options__` fills in proxy, locale,
timezone, extra headers, user agent, flags, and channel. `DynamicSessionMixin` and
`StealthySessionMixin` specialise it; the stealth one adds fixed 1920×1080 screen and viewport,
permissions, `ignore_https_errors`, and conditional flags for WebRTC, WebGL, and canvas noise.

`StealthySessionMixin._detect_cloudflare` classifies a challenge as `non-interactive`, `managed`,
`interactive`, or `embedded` by looking for `cType: '...'` in the page source, falling back to a
CSS check for the Turnstile script tag.

### `engines/_browsers/_controllers.py` (401 lines)

`DynamicSession` and `AsyncDynamicSession`. `start()` picks one of three launch strategies: CDP
connection, plain `launch()` when a proxy rotator is configured (contexts are created per request
instead), or `launch_persistent_context()` with a user data dir. `fetch()` runs the sequence:
validate per-call params, loop over retries, acquire a page, attach the response handler, run
`page_setup`, navigate, wait for stability, run `page_action`, wait for `wait_selector`, sleep for
`wait`, build the `Response`.

Every user-supplied hook is wrapped in its own try/except that logs and continues.

### `engines/_browsers/_stealth.py` (576 lines)

`StealthySession` / `AsyncStealthySession`, identical in shape to the dynamic controllers but
launching through Patchright and adding `_cloudflare_solver`. The solver waits for network idle,
detects the challenge type, and then either waits out a non-interactive challenge or locates the
Turnstile checkbox — by iframe bounding box when the challenge iframe is reachable, otherwise by
CSS selector — and clicks at a randomised offset inside it with a randomised delay. It recurses if
the challenge is still present afterwards.

### `engines/toolbelt/custom.py` (307 lines)

`Response(Selector)` — the unified return type. Adds `status`, `reason`, `cookies`, `headers`,
`request_headers`, `history`, `meta`, `request`, and `captured_xhr` on top of everything a
`Selector` can do, overrides `body` to return bytes, and logs a one-line summary on construction.
`follow()` builds a spider `Request` for a linked URL, carrying over the session kwargs, the
callback, the priority, and the meta from the current request, and setting the referer chain.

`BaseFetcher` holds the class-level parser configuration shared by all fetchers, with
`configure(**kwargs)` guarded by a `parser_keywords` whitelist that raises on unknown keys, and
`_generate_parser_arguments()` returning the dict merged into each call's `selector_config`.

`StatusText` maps status codes to reason phrases through a `MappingProxyType` with a cached
`get()` — needed because Playwright sometimes returns an empty `status_text`.

### `engines/toolbelt/convertor.py` (325 lines)

`ResponseFactory` with four entry points: `from_http_request` (curl_cffi),
`from_playwright_response`, `from_async_playwright_response`, and the two history builders.

Details it handles that a naive adapter would miss: charset extraction from the `content-type`
header with a cached regex (Playwright does not expose encoding); falling back from
`final_response` to `first_response`; taking rendered content from `page.content()` for HTML and
raw `response.body()` otherwise; walking `request.redirected_from` backwards to build history,
with the comment explaining that calling `.text()` on a redirect response raises; and converting
captured XHR responses recursively with `collect_history=False`.

`_get_page_content` retries `page.content()` up to 20 times at 500ms intervals, referencing the
Playwright issue where it fails on Windows.

### `engines/toolbelt/fingerprints.py`

`get_os_name()` maps the platform to browserforge's OS names, returning all supported OSes when
unknown. `driven_browser_version()` reads the Chromium version out of the installed Playwright or
Patchright `browsers.json` so generated user agents match the browser actually being driven.
`generate_headers()` builds headers via browserforge, restricted to Chrome in browser mode and
widened to Chrome/Firefox/Edge in HTTP mode, then rewrites the Chrome version in the UA string to
match the driver.

### `engines/toolbelt/navigation.py`

`construct_proxy_dict()` accepts a proxy string or dict and returns Playwright's format,
validating the scheme and using msgspec to validate the dict form.
`create_intercept_handler` / `create_async_intercept_handler` close over a frozenset of blocked
domains and a set of blocked resource types, returning a route handler. `_is_domain_blocked` walks
the hostname's suffix chain (`a.b.c.net` → `b.c.net` → `c.net`) doing O(1) frozenset lookups, so
subdomain blocking costs no more than exact matching.

### `engines/toolbelt/proxy_rotation.py`

`ProxyRotator` — a slotted, lock-guarded list of proxies with a pluggable strategy. A strategy is
any `Callable[[List[ProxyType], int], Tuple[ProxyType, int]]`; `cyclic_rotation` is the default.
`is_proxy_error()` classifies an exception by substring-matching its message against a set of six
indicators, and is used identically by the HTTP and browser retry loops.

### `engines/toolbelt/ad_domains.py` (3,537 lines)

A frozenset of ~3,500 ad and tracker domains from Peter Lowe's list, with the source URL and the
exact query used to generate it in the module docstring. Imported lazily inside
`PlaywrightConfig.__post_init__` so it costs nothing unless `block_ads` is set.

## `scrapling/fetchers/` — the public facade

`fetchers/__init__.py` mirrors the root's lazy-import registry for nine names.

`fetchers/requests.py` defines `Fetcher` and `AsyncFetcher` as thin classmethod facades over two
module-level client instances, with `_merge_selector_config` layering `BaseFetcher`'s class
config under any per-call `selector_config`.

`fetchers/chrome.py` and `fetchers/stealth_chrome.py` define `DynamicFetcher` and
`StealthyFetcher`, whose `fetch`/`async_fetch` classmethods open a session, make one request, and
close it. `chrome.py` also accepts the legacy `custom_config` key as a fallback for
`selector_config`.

The pattern across all three: a one-shot classmethod for scripts, a session class for anything
repeated.

## `scrapling/spiders/` — the crawl framework

### `spiders/spider.py` (330 lines)

`Spider(ABC)` is the user-facing base class. Everything configurable is a class attribute:
`name`, `start_urls`, `allowed_domains`, `robots_txt_obey`, `development_mode`,
`concurrent_requests`, `concurrent_requests_per_domain`, `download_delay`,
`max_blocked_retries`, the five autothrottle settings, three fingerprint flags, and four logging
settings.

`__init__` builds a per-spider logger with a `LogCounterHandler` (counts messages by level, which
end up in the crawl stats), disables propagation to the root scrapling logger, and calls
`configure_sessions()`, wrapping any failure in `SessionConfigurationError`. A spider with zero
registered sessions is rejected at construction.

Hooks a user can override: `start_requests`, `parse` (abstract), `on_start`, `on_close`,
`on_error`, `on_scraped_item` (return `None` to drop an item), `is_blocked`,
`retry_blocked_request`, `configure_sessions`. That list is the entire extension surface — there
are no middlewares.

`start()` installs a SIGINT handler that requests a graceful pause on the first Ctrl+C and a
forced stop on the second, then runs the engine under `anyio.run` with an optional uvloop backend.
`stream()` is the async generator alternative, yielding items as they are scraped.

### `spiders/request.py` (174 lines)

`Request` carries url, session id, callback, priority, `dont_filter`, meta, retry count, and
arbitrary session kwargs.

`update_fingerprint()` builds a sha1 over `{sid, method, canonicalized url, body hex}` — using
`w3lib.url.canonicalize_url` so query-parameter order does not create duplicates — and optionally
folds in sorted kwargs and normalised headers. The result is cached on the instance.

`__getstate__`/`__setstate__` replace the callback with its name for pickling, and
`_restore_callback(spider)` rebinds it by `getattr` after a checkpoint load. That pair is what
makes the whole checkpoint system possible, since bound methods do not pickle cleanly across
runs.

`__lt__`/`__gt__` compare by priority for the heap; `__eq__` compares fingerprints and raises if
they have not been computed.

### `spiders/scheduler.py` (149 lines)

An `asyncio.PriorityQueue` of `(-priority, counter, request)` tuples — the negation makes higher
priority dequeue first, and the monotonic counter breaks ties without comparing requests. A
`_seen` set of fingerprints does deduplication unless `dont_filter` is set.

The `_pending` dict mirrors the queue so `snapshot()` can serialise pending work without draining
it, and `_inflight` tracks dequeued-but-unfinished requests so `complete()` can stop tracking
them. `restore()` reloads both from checkpoint data.

### `spiders/session.py` (149 lines)

`SessionManager` maps session IDs to `FetcherSession`, `AsyncDynamicSession`, or
`AsyncStealthySession` instances. `add(..., lazy=True)` defers startup until the first request
routed to that ID, guarded by a lock. `fetch(request)` routes by `sid`, unwraps `FetcherSession`
to its async client to call `_make_request` directly, attaches the request to the response, and
merges request meta into response meta.

### `spiders/engine.py` (473 lines)

`CrawlerEngine` is the loop. Construction wires up the optional subsystems — robots manager,
response cache, autothrottle — based on spider flags, and creates an `anyio.CapacityLimiter` for
global concurrency plus a dict of per-domain limiters.

`_process_request()` is the per-request pipeline:

1. robots.txt check (drop silently if disallowed), then resolve the domain's delay floor as the
   max of the spider's `download_delay`, robots `Crawl-delay`, and robots `Request-rate`.
2. Cache lookup by fingerprint when development mode is on.
3. Acquire the rate limiter, apply the autothrottle delay, record proxies into stats, fetch, time
   the request.
4. Cache the response on a miss.
5. Ask `spider.is_blocked(response)`; feed latency and outcome into autothrottle.
6. On a block, copy the request, bump retry count, lower priority, set `dont_filter`, drop the
   proxy kwargs, pass it through `spider.retry_blocked_request()`, and re-enqueue — up to
   `max_blocked_retries`.
7. Otherwise dispatch to `_run_callbacks`.

`_run_callbacks()` iterates the callback's async generator: `Request` results go back to the
scheduler after an offsite check, `dict` results go through `on_scraped_item` and land in the item
list or the stream, anything else logs an error. The whole loop is inside a try/except that logs
with a traceback and calls `spider.on_error`, so one bad callback cannot end the crawl.

`crawl()` is the outer loop. It restores a checkpoint if one exists, opens all sessions, prefetches
robots.txt for the seed domains concurrently, seeds the queue from `start_requests()` (skipped
when resuming), then spins: honour pause requests, save periodic checkpoints, exit when the queue
is empty and nothing is in flight, throttle task spawning to `concurrent_requests`, and
`tg.start_soon` each dequeued request. On exit it calls `on_close`, cleans up checkpoint files if
the run completed, and logs the full stats dict as JSON.

`_stream()` runs `crawl()` in a task group while yielding from a memory object stream with a
100-item buffer.

### `spiders/result.py` (215 lines)

`ItemList(list)` with `to_json`, `to_jsonl`, `to_csv`, and `to_xml`. The CSV writer unions keys
across items in first-seen order and JSON-encodes non-scalar values. The XML writer sanitises tag
names against `[^\w.-]`, preserves the original key in a `name` attribute when it had to rewrite
it, and strips characters XML cannot represent — a real concern given scraped text.

`CrawlStats` is a dataclass with 20+ counters (requests, failures, offsite, robots-disallowed,
cache hits/misses, bytes per domain, items scraped/dropped, per-status counts, per-session counts,
proxies used, autothrottle delays, log counts by level) plus `elapsed_seconds` and
`requests_per_second`.

`CrawlResult` bundles stats, items, and a `paused` flag, and is iterable.

### `spiders/throttle.py` (140 lines)

`parse_retry_after()` reads `Retry-After` as either seconds or an HTTP date.

`AutoThrottle` keeps a delay per domain. `record()` computes
`new_delay = max((current + latency/target_concurrency) / 2, latency/target_concurrency)`, then
applies a penalty on a bad response — either the server's `Retry-After` or double the current
delay — and clamps between the per-domain floor and `max_delay`. A block can never make the
spider faster.

### `spiders/cache.py` (~120 lines)

`ResponseCacheManager` writes one JSON file per request fingerprint under
`.scrapling_cache/{spider.name}/`, base64-encoding the body so binary content survives. Writes go
to a `.tmp` file and are renamed into place. The comments document a fixed bug: browser responses
store cookies as a tuple of dicts and HTTP responses as a flat dict, and collapsing the former was
silently dropping every cookie.

### `spiders/checkpoint.py` (~95 lines)

`CheckpointData` (pending requests + seen fingerprints) pickled to `checkpoint.pkl` with the same
temp-file-and-rename discipline. `load()` returns `None` and logs rather than raising if the file
is corrupt, so a bad checkpoint degrades to a fresh crawl.

### `spiders/robotstxt.py`

`RobotsTxtManager` caches a `protego.Protego` parser per domain, fetching `robots.txt` through the
spider's own session so proxies and impersonation apply. `prefetch()` warms the cache for all seed
domains concurrently before the crawl starts. A fetch or parse failure produces an empty parser,
so a missing robots.txt allows everything.

### `spiders/links.py` (300 lines)

`LinkExtractor`, modelled on Scrapy's. Accepts allow/deny regex (strings or compiled patterns,
individually or as iterables), allow/deny domains with subdomain matching, `restrict_css` and
`restrict_xpath` scoping, configurable tags and attributes, canonicalisation, whitespace
stripping, a 100+ entry `IGNORED_EXTENSIONS` set, and a `process` callable that can rewrite or
drop a URL.

`extract(response)` builds a single XPath union (`.//a/@href | .//area/@href`) rather than
querying per tag, and deduplicates via `dict.fromkeys` to preserve insertion order.
`matches(url)` applies the same filters to a bare URL, which is how `SitemapSpider` dispatches
without a response.

### `spiders/templates/`

- `crawler.py` — `CrawlRule` (dataclass of extractor + callback + priority + request processor)
  and `CrawlSpider`, whose `parse()` runs every rule's extractor over the response and yields
  followed requests.
- `sitemap.py` — `SitemapSpider`. Detects a robots.txt body and pulls `Sitemap:` directives out of
  it; parses `<sitemapindex>` into child sitemaps and `<urlset>` into URLs; recurses into children
  filtered by `sitemap_follow`; dispatches each URL through `rules()` with first-match-wins.
  Namespaces are handled with `etree.QName(...).localname` so any sitemap dialect works.
- `feed.py` — `XMLFeedSpider` (iterate nodes matching `itertag`, hand each to `parse_node` as a
  namespace-stripped deep copy) and `CSVFeedSpider` (`DictReader` rows into `parse_row`).
- `shopify.py` — `ShopifySpider`, which walks `/collections.json` and each collection's
  `products.json` and yields one item per variant, never touching HTML.
- `_utils.py` — `_decompress()`, which gunzips by content-type or magic bytes with a 64 MiB output
  cap to defend against gzip bombs.

`CONTRIBUTING.md` draws the line for what belongs here: platform templates are accepted when the
platform has a uniform structure across many independent domains; single-site scrapers never are.

## `scrapling/integrations/scrapy.py`

`convert_response()` turns a Scrapy response into a Scrapling `Response`, parsing `Set-Cookie`
from raw header lines because Scrapy's `to_unicode_dict` joins duplicate headers with commas and
corrupts them.

`scrapling_response` is a decorator usable bare or parameterised (`func is None` → return a
`partial`). It inspects the wrapped callback and emits a wrapper of the same kind — async
generator, coroutine, generator, or plain function — because Scrapy introspects the callback
itself, not just its return value. That detail is the whole reason the decorator is 130 lines
instead of 15.

## Tests (`tests/`, ~9,000 lines)

Mirrors the source layout: `parser/`, `fetchers/` (split into `sync/` and `async/`), `spiders/`,
`core/`, `cli/`, `ai/`, `integrations/`. Tests are organised into classes by behaviour
(`TestSchedulerEnqueue`, `TestSchedulerInit`), one assertion cluster per method, with docstrings
that state the expected behaviour.

`tox.ini` runs pytest three times to work around real constraints: browser tests serially,
asyncio tests serially (nested-loop problems on CI), and everything else with `-n auto`.
`pytest.ini` enables `--doctest-modules`, so docstring examples in the library are executed.

## Repository files

| File | Role |
|:--|:--|
| `pyproject.toml` | Static version (the comment explains it is static for Docker layer caching), 6 core dependencies, four optional extras (`fetchers`, `ai`, `shell`, `all`), two console scripts, mypy and pyright config |
| `tox.ini` | Test matrix 3.10–3.13 plus a `pre-commit` env |
| `pytest.ini` | Strict asyncio mode, doctest modules |
| `ruff.toml` | 120-char lines, `E`/`F`/`W` rules, Python 3.10 target |
| `.pre-commit-config.yaml` | bandit, ruff (lint + format), vermin |
| `.bandit.yml` | Ten skipped checks, each with a comment explaining why |
| `Dockerfile` | `uv`-based, dependency layer separated from source layer, browsers installed in one layer, entrypoint `scrapling` |
| `server.json` | MCP registry manifest declaring both the PyPI (`uvx`) and OCI package forms |
| `.readthedocs.yaml`, `zensical.toml`, `docs/` | Documentation site: guides, tutorials, API reference generated by mkdocstrings, and ten translated READMEs |
| `agent-skill/` | A packaged AgentSkill: `SKILL.md` plus `references/` mirroring the docs and four runnable examples, distributed as a zip and via skill registries |
| `AI_POLICY.md` | Requires disclosure of AI assistance in PRs; undisclosed AI output gets labelled and closed |
| `CONTRIBUTING.md` | PRs target `dev`, conventional commits, mandatory type-check and lint passes, no author names in code |
| `benchmarks.py` | Timed comparison against lxml, parsel, bs4, selectolax, pyquery, MechanicalSoup, AutoScraper |
| `.github/workflows/` | `tests.yml` (matrix + Playwright browser caching), `code-quality.yml` (bandit, ruff, vermin, mypy, pyright, all `continue-on-error` with a summary gate), `release-and-publish.yml` (PR title becomes the tag, PR body becomes the release notes, then publishes to PyPI via trusted publishing), `docker-build.yml` |
