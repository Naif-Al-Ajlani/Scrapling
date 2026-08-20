# What else this structure could build

An evaluation of tools that could be built on Scrapling's architecture without being about web
scraping. The question is not "what else could use a parser" — it is which problems are shaped
closely enough that the five layers, the one-return-type rule, the resumable crawl loop, and the
adaptive relocation idea all still earn their place.

Twenty candidates were considered. Five are worth building, seven are held back by one specific
thing, and four fail the test outright. The reasons matter more than the ranking.

## How the candidates were judged

### Structural fit — the five properties

From `03-building-similar-tools.md`. A candidate scores one point per property it genuinely has.
Fewer than four means the architecture is overkill.

1. **Several transports, one result.** Data arrives more than one way and the differences should
   not reach the user.
2. **A domain object worth investing in.** One thing manipulated repeatedly, where most of the
   value is in how good it is to work with.
3. **Hostile or unreliable sources.** Retry, backoff, and recovery are features rather than error
   handling.
4. **Scales from one item to many** without switching tools.
5. **Long-running jobs** needing resumability and a fast development loop.

### Three viability tests

**Adaptive analogue.** Does the signature idea — fingerprint the thing you located, and when the
direct path breaks, score candidates on several weak signals — have a real counterpart? Without
one, you are building a competent library with no reason for anyone to switch to it. This is the
test that separates the top five from the rest.

**Occupied ground.** What already exists, and is the gap a real one or a preference?

**Build cost.** How much has to be invented rather than transposed.

## The scorecard

| Candidate | Fit | Adaptive analogue | Ground | Verdict |
|:--|:--:|:--|:--|:--|
| Document field extraction | 5/5 | Field relocation after template drift — direct | Parsing tools strong, extraction weak | **Build** |
| Test and CI intelligence | 5/5 | Test identity across renames and moves | All SaaS, nothing library-first | **Build** |
| Mailbox mining | 5/5 | Transactional email template drift | Hosted APIs or raw transports, nothing between | **Build** |
| Scholarly harvesting | 5/5 | Entity resolution across sources | One client per source, no unification | **Build** |
| Dependency and supply chain | 5/5 | Package identity across ecosystems | Crowded | **Build, with eyes open** |
| SaaS connector framework | 5/5 | Schema drift recovery | dlt already library-first and good | Hold |
| Clinical data (FHIR/HL7/DICOM) | 5/5 | Vendor deviation mapping | Fragmented, specialist | Hold |
| Financial statement aggregation | 5/5 | Statement layout drift | Subset of document extraction | Hold |
| Industrial telemetry | 4/5 | Register-map drift across firmware | Vendor stacks dominate | Hold |
| Meeting transcript pipelines | 4/5 | Speaker identity across meetings | Vendors ship features fast | Hold |
| Geospatial / STAC harvesting | 4/5 | Weak — STAC already standardises | STAC tooling mature | Hold |
| Data reconciliation | 4/5 | Column matching across schema versions | Great Expectations, Soda, Datafold | Hold |
| Log and telemetry pipelines | 4/5 | Field recovery after producer renames | Vector, Fluent Bit, Benthos | No |
| Codebase indexing for agents | 2/5 | None | Saturated | No |
| Any single-transport client | 1/5 | None | — | No |
| Real-time stream processing | 2/5 | None | — | No |
| Interactive low-latency tooling | 1/5 | None | — | No |

## The five worth building

### 1. Document field extraction that survives template drift

**The problem.** Organisations pull fields out of invoices, bank statements, remittance advices,
lab reports, insurance forms, and shipping documents. The vendor changes the template and the
extraction breaks — usually silently, producing a wrong number rather than an error. The two
current answers are both unsatisfying: position-based template tools that shatter on any layout
change, or an LLM call per document, which costs money per page, gives no audit trail, and is not
reproducible.

**Why the shape fits.** Every property is present, and the adaptive layer is a near-direct
transposition rather than an analogy.

| Scrapling | This tool |
|:--|:--|
| `Selector` / `Selectors` | `Document` → `Page` → `Block` / `Line` / `Word`, each carrying geometry |
| `css()` / `xpath()` | `label()` (find by anchor text), `right_of()` / `below()` (spatial), `cell()` (table) |
| `Response(Selector)` | `Extraction(Document)` carrying source path, page count, OCR confidence |
| `ResponseFactory` | `DocumentFactory.from_pdf_text`, `.from_ocr`, `.from_docx`, `.from_xlsx`, `.from_html`, `.from_image` |
| `_StorageTools.element_to_dict` | Field fingerprint: anchor label text, relative offset, neighbouring labels, font size and weight, block role, page index |
| `relocate()` | Rescan and score candidates when the anchor no longer resolves |
| `find_similar()` | Find the other line items once you have located one |
| Response cache / development mode | Cache OCR output — the expensive step, and the one you re-run most while iterating |
| `Spider` | `Batch` over a directory tree or bucket prefix, resumable |

**What the adaptive layer buys.** A deterministic, auditable field locator. When it relocates a
field it can report the score and the signals that matched, which an LLM extraction cannot. That
is the difference between a tool finance will approve and one they will not.

**What is hard.** Scrapling gets its tree for free from lxml; here you have to build the document
tree yourself, and geometry-aware querying is a genuine design problem — reading order in
multi-column layouts, merged table cells, rotated scans. Budget most of the effort for the domain
object and almost none for the engines.

**Occupied ground.** unstructured.io and docling do parsing-to-structure well and treat field
extraction as somebody else's problem. Commercial IDP platforms do the fields but hosted, priced
per page, and closed. The gap is a local-first library where the extraction rule is a stored
fingerprint that heals itself.

### 2. Test and CI intelligence

**The problem.** Every engineering organisation past a certain size rebuilds the same thing badly:
which tests are flaky, which failures cluster, which are getting slower, which broke on which
commit. The data lives across CI provider APIs, junit XML artifacts, raw logs, and coverage
reports, in different shapes per provider.

**Why the shape fits.** The transports differ enormously — GitHub Actions, GitLab, Jenkins,
Buildkite, CircleCI all model a run differently, and a local directory of junit XML is a fourth
shape. One `Run` → `Job` → `TestCase` object across all of them is exactly the payoff the
architecture exists for. Backfilling a year of CI history is a resumable crawl with a frontier,
rate limits, and pagination.

**The adaptive analogue is the best of the five.** Test identity survives renames and moves.
`tests/test_api.py::test_login` becoming `tests/api/test_auth.py::TestAuth::test_login` is the same
test, and today every flake dashboard loses its history at that moment. Fingerprint it on name
tokens, path tokens, the assertion signature, the duration profile, and historical
failure correlation with other tests; when the exact identifier stops resolving, score candidates
and carry the history forward with a confidence figure.

Nobody does this well, and the failure is visible to anyone who has refactored a test suite.

**What is hard.** CI provider API differences are large and their rate limits are strict — GitHub's
in particular will shape the whole fetch strategy. Log parsing is per-framework and never finishes.

**Occupied ground.** BuildPulse, Trunk, Datadog CI Visibility, and the providers' own test insights
are all hosted SaaS. A self-hosted library a developer can adopt without procurement is a real
opening, and it is the candidate with the easiest distribution story: the audience is the same
people who install libraries.

### 3. Mailbox mining

**The problem.** A mailbox is the richest unstructured data source most people own, and the hardest
to work with programmatically. Extracting receipts for expenses, order confirmations for personal
finance, or a complete record for a legal hold means writing against IMAP, Gmail's API, and
Microsoft Graph separately, then parsing HTML bodies whose templates change every few months.

**Why the shape fits.** This needs the least invention of any candidate, because the message body
is HTML — the existing `Selector` works on it unchanged.

| Scrapling | This tool |
|:--|:--|
| `Fetcher` / `DynamicFetcher` / `StealthyFetcher` | `ImapSession`, `GmailSession`, `GraphSession`, `LocalSession` (mbox, eml, pst) |
| `Response` | `Message(Selector)` with `.thread`, `.parent`, `.replies`, `.attachments` |
| `ProxyRotator` | `AccountRotator` — same code, different name |
| `AutoThrottle` | Graph and Gmail quota backoff, reading their own headers |
| `Spider` | Mailbox backfill with checkpoints, because a full sync takes hours |
| `relocate()` | Relocate the total, the order number, the delivery date when the sender's template changes |

**What is hard.** OAuth setup is friction users feel immediately, so the getting-started path has to
be excellent. Privacy expectations mean local-first is not optional — no hosted component, ever.
PST parsing is unpleasant and can be deferred.

**Occupied ground.** Hosted email APIs solve the transports at a price and want your mail on their
servers. `imapclient` and the raw SDKs give you transports and nothing else. Nothing sits in
between with a real domain object and drift recovery.

### 4. Scholarly and bibliographic harvesting

**The problem.** Anyone doing bibliometrics, systematic reviews, or science-of-science work pulls
from OpenAlex, Crossref, PubMed, arXiv, DataCite, and Semantic Scholar, each with its own client
library, its own identifier scheme, and its own idea of what a "work" is. Reconciling them is
manual, per-project, and thrown away afterwards.

**Why the shape fits.** Citation-graph traversal *is* a crawl — a frontier, deduplication, politeness
limits, and a job that runs for hours. The spider layer transfers with almost no change. The
adaptive analogue is entity resolution: the same paper appears across sources under different
identifiers, and an author appears under name variants. Fingerprint on title similarity, author
set, year, venue, and identifier fragments, then score. That is the same weak-signal scoring as
`__calculate_similarity_score`, applied to metadata instead of DOM nodes.

**What is hard.** Each source's politeness policy differs and some require a contact address in the
user agent; getting that wrong gets you blocked. Full-text PDF handling overlaps with candidate 1
and should be left out of scope at first.

**Occupied ground.** `pyalex`, `habanero`, and Biopython's Entrez wrapper each cover one source
well. The unification is the gap, and the audience — research software engineers — is small but
adopts open tools readily and cites them.

### 5. Dependency and supply-chain inventory

**The problem.** Answering "what do we actually depend on, transitively, across every repo and
ecosystem, and which of it is vulnerable" means walking npm, PyPI, Maven, crates, and NuGet
registries, parsing five lockfile formats, and joining against advisory databases.

**Why the shape fits.** Transitive resolution across a large organisation is a resumable graph
crawl with registry rate limits. The adaptive analogue is package identity across ecosystems and
renames — the same project published under different names, or renamed mid-history — fingerprinted
on repository URL, maintainer set, description similarity, and file hashes.

**Why "with eyes open".** This is the most crowded space on the list. Syft, Grype, OSV-Scanner,
Dependency-Track, and several commercial products are already here, and SBOM formats are being
standardised, which erodes the multi-transport pain that justifies the architecture. Build it only
if the identity-resolution angle is genuinely the product rather than a feature.

## Held back by one thing each

**SaaS connector framework.** Perfect structural fit — it is the worked example in
`03-building-similar-tools.md`. Held back because `dlt` already occupies exactly this position:
library-first, Python, resumable, well designed. Differentiation would rest on schema-drift
recovery alone, which is thin ground for a whole project. Better as a contribution to what exists.

**Clinical data (FHIR, HL7v2, DICOM).** Structurally excellent — three genuinely different
transports, a real domain object, hostile sources in the form of vendor deviations from spec, and
long backfills. Held back by regulatory burden and the difficulty of getting realistic test data.
A specialist's multi-year project, not a first build.

**Financial statement aggregation.** Strong fit and a real drift problem in statement layouts, but
it is a subset of candidate 1 wearing a compliance jacket. Build the document tool first; this
becomes a template pack on top of it.

**Industrial telemetry (Modbus, BACnet, OPC-UA).** Good fit, and register-map drift across firmware
versions is a legitimate adaptive analogue. Held back by the audience: plants buy vendor stacks,
and open-source adoption in that world is slow enough that the feedback loop needed to get the
domain object right may never close.

**Meeting transcript pipelines.** One return type across Zoom, Teams, Meet, and local audio is
genuinely valuable, and speaker identity across meetings is a fair adaptive analogue. Held back
because the vendors ship native features quickly and most of the transports are a single API each,
which weakens property 1 over time.

**Geospatial and STAC harvesting.** Scores well on scale and resumability — huge downloads,
resumable transfers, long jobs. Held back because STAC already standardises the interface, which
removes most of the pain the architecture solves, and the adaptive layer has no obvious counterpart.

**Data reconciliation between systems.** Column matching across schema versions is a good adaptive
analogue. Held back by an active field — Great Expectations, Soda, dbt tests, Datafold — where the
differentiator would have to be the matching itself, which is narrower than a product.

## What fails, and why the failure is instructive

**Codebase indexing for agents.** Scores 2/5. There are no hostile sources, so retry and throttle
machinery is dead weight; jobs are minutes, not hours, so checkpointing is dead weight; and the
space is saturated. It looks adjacent because it involves parsing and traversal, which is exactly
the trap — parsing is not the property that makes this architecture worth its cost.

**Any "better client for X".** Scores 1/5 on the first property, and that single failure removes
the factory, the facade's two usage modes, and most of the reason the layers exist. If there is
one transport, write one clean module.

**Real-time stream processing.** The orchestration layer assumes bounded work with a frontier that
eventually empties — the crawl loop exits when the queue is empty and nothing is in flight. A
never-ending stream wants backpressure, windowing, and watermarks instead. Wrong orchestration
shape, and retrofitting it would mean rewriting the layer that carries most of the value.

**Interactive, low-latency tooling.** Validation at the boundary, layered configuration, and lazy
imports are paid for by long jobs where the cost amortises to nothing. A tool that must respond in
under a second pays the same complexity for no return.

## Reusable directly, not just as a pattern

Scrapling is BSD-3-Clause, so these can be lifted with attribution rather than reimplemented. Each
is small, self-contained, and depends on little:

| Module | Lines | What transfers |
|:--|--:|:--|
| `engines/toolbelt/proxy_rotation.py` | 104 | Identity rotation with pluggable strategy — rename `proxy` to whatever your identities are |
| `core/storage.py` | 155 | Storage ABC plus a thread-safe SQLite implementation, unchanged |
| `spiders/checkpoint.py` | 90 | Atomic pickle checkpointing with corrupt-file tolerance |
| `spiders/throttle.py` | 101 | Latency-driven backoff with `Retry-After` handling |
| `spiders/cache.py` | 93 | Fingerprint-keyed response cache for development mode |
| `spiders/scheduler.py` | 91 | Priority queue with fingerprint dedup and checkpoint snapshots |
| `spiders/result.py` | 215 | `ItemList` exporters (JSON, JSONL, CSV, XML) and the stats dataclass |
| `core/custom_types.py` | 345 | `TextHandler` / `TextHandlers` — useful in any tool that returns text |

That is roughly 1,200 lines of the orchestration and utility layers already written and tested.
The work in any of these candidates is the domain object and the engines.

## If you build one, week one

1. **Write the README example before any code.** Five lines, real inputs, the output you want. If
   it needs a setup block first, the API is wrong and no amount of layering will fix it.
2. **Build the domain object against a corpus you already have.** Not a synthetic fixture — real
   invoices, a real mailbox export, a real year of CI runs. The domain object is the whole product
   and it can only be judged against real mess.
3. **Design the fingerprint format before the storage backend.** What you record about a located
   thing determines what you can recover later. Getting it wrong means a migration.
4. **Add the second transport in week one, not month three.** It is the only test of whether the
   seams are in the right place, and fixing them early is cheap.
5. **Ship the CLI before the agent surface.** The CLI forces the facade to be usable by someone who
   has not read the source. The MCP server is a consumer of that same facade and gets easier once
   it is right.
