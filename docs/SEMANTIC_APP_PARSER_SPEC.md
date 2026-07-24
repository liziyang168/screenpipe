# Semantic App Parser

> **Status**: Draft foundation, not enabled in capture
> **Date**: 2026-07-23

## 1. Problem

Screenpipe captures timestamped screenshots, consolidated accessibility/OCR text,
structured elements, and UI events. AI retrieval is already compact, but it is
app-agnostic. A model repeatedly reconstructs relationships such as sender to
message, thread to subject, document to editor, and task to status from generic
text and UI nodes.

App-aware deterministic parsers can produce better context with fewer irrelevant
tokens. They must not increase capture latency, create a second observer, run an
LLM per frame, or persist a duplicate raw tree.

## 2. Source order

Use the strongest available source and retain the weaker sources as evidence:

1. Explicitly enabled native structured source
2. App-specific semantic parser over accessibility
3. Generic accessibility text and element outline
4. OCR fallback
5. A few frame images when pixels materially matter

A parser may return `NotHandled`. Generic accessibility retrieval remains the
fallback, so an app redesign degrades quality instead of losing context.

## 3. Foundation in this change

`screenpipe-semantic` adds no runtime integration. It defines:

- stable cross-platform `AppIdentity`
- parser manifests and precompiled app/URL selection
- parser-declared accessibility attribute and offscreen requirements
- a compact immutable tree with integer links and interned strings
- a deterministic Rust parser trait
- typed conversation, message, document, task, calendar, and page items
- stable input fingerprints that include parser/schema/app/content versions
- output validation for size, item count, parents, cycles, and source-node links

Keeping this inactive makes the first review about contracts and resource bounds.
Capture, database, scripting, and retrieval integrations can land independently.

## 4. Capture integration

The parser registry returns a `SemanticCapturePlan` before an accessibility walk.
The plan combines the selected parser's `AttributeSet` and `OffscreenPolicy`
with engine-owned `TreeBudget` and `OutputBudget` hard caps. Parser packs may
request less data, but they cannot raise the engine's resource ceilings.

Platform walkers should append every retained structural node to
`SemanticTreeBuilder` while they perform the existing walk. On macOS the current
walker already batches role, value, title, description, position, and size for
each visited node. Retaining those fields should not add accessibility IPC.
Optional DOM classes and identifiers should be added to the same batch only when
the selected plan requests them.

The existing text-oriented `TreeSnapshot.nodes` and database element behavior
stay unchanged until the semantic path is proven.

## 5. Scheduling

Do not parse inside `paired_capture`, the frame transaction, or deferred element
insertion.

After `paired_capture` returns a durable `frame_id`, move the compact tree into a
bounded parser actor with `try_send`. The capture loop never awaits capacity.

Actor policy:

- capacity 8
- latest job wins per app/window
- one low-priority worker
- no screenshots in jobs
- 20 ms hard deadline
- 8 to 16 MB scripting heap if a scripting runtime is added
- 64 KB and 256 item output caps
- drop stale work under memory, battery, capture, or database pressure

When `elements_ref_frame_id` and parser version identify an existing projection,
reuse it without parsing.

## 6. Parser runtime

The crate initially exposes a Rust trait. Built-in Rust parsers are the lowest-risk
way to benchmark the contract.

For rapid parser updates across many apps, evaluate one shared sandboxed QuickJS
runtime later. Parser code must receive opaque `NodeId` handles and Rust host
selectors, not a copied JavaScript object tree. Disable filesystem, network,
process, clock, randomness, imports, and host mutation. Parser packs require
signatures, compatibility ranges, last-known-good rollback, and a kill switch.

The parser ABI and stored output must remain independent of the chosen runtime.

## 7. Storage

Do not store one parsed JSON blob per frame. Repeated screens would repeat every
message or document.

Proposed tables:

- `semantic_parse_runs`: parser/version/input fingerprint/status/duration
- `semantic_frame_refs`: frame to parse run, reusing identical projections
- `semantic_items`: immutable canonical typed records
- `semantic_observations`: run membership, parent, order, and source node indexes
- external-content semantic FTS

Do not overload `frames.full_text`, `memories`, or `outputs`. They represent raw
search text, durable user facts, and generated files respectively.

## 8. Retrieval

Add one semantic endpoint and one MCP tool rather than app-specific tools.

Default output is grouped plain text:

```text
Slack | #release | 10:02-10:08
alice 10:02: notarization is blocking the release
[user] 10:04: I will retry the signing job
source frame: 481992
```

The agent progression becomes activity summary, semantic app context, generic
content search, element search, and pixels only when needed.

## 9. Privacy and retention

Semantic strings are a new PII surface. Exclusion, incognito, password, DRM, and
pause gates run before enqueue. Synchronous removal applies before insertion.
The async redaction worker must add a semantic watermark and should reuse the
frame redaction map when possible.

Retention behavior:

- `media`: keep semantic records
- `lean`: keep compact semantic text, clear source-node blobs with tree geometry
- `all` and time-range deletion: remove frame references and garbage-collect
  unreferenced runs, observations, and items

No cloud sync until permission, encryption, deletion, and live read-back are
verified separately.

## 10. Acceptance gates

Measure on an older Intel Mac and a representative Windows enterprise laptop:

| Metric | Gate |
|---|---:|
| Parser projection p50 | under 1 ms |
| Parser projection p95 | under 5 ms |
| Hard parser deadline | 20 ms |
| Incremental steady RSS | under 20 MB |
| Average parser CPU over 8-hour trace | under 0.5 percent |
| Capture delay caused by parser | zero awaited time |

Token success is end-to-end tokens and tool calls needed to answer a fixed task
suite, compared with current `full_text` and element-outline retrieval. Raw JSON
compression alone is not a sufficient metric.

## 11. Rollout

1. Benchmark compact structural capture and one Rust parser.
2. Add nonblocking worker, schema, retention, and redaction integration.
3. Add Slack/Teams, Gmail/Outlook, and VS Code/Cursor parser fixtures.
4. Add semantic search and MCP output behind a feature flag.
5. Expand shared parser families.
6. Consider signed remote parser packs only after shipped parsers are stable.
