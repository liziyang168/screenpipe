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

For a matched app, parser order is:

1. an app or version-specific override, only when needed
2. a shared UI-family parser with declarative app profiles
3. the existing generic accessibility projection outside this crate

`NotHandled` or a parser failure advances to the next candidate. `Empty` stops
the chain because it means the parser recognized a genuinely empty screen.

## 3. Foundation in this change

`screenpipe-semantic` adds no runtime integration. It defines:

- stable cross-platform `AppIdentity`
- parser manifests and precompiled app/URL selection
- ordered app-override and shared-family parser candidates
- merged capture requirements across fallback candidates
- fail-open parser execution with failure details for telemetry
- parser-declared accessibility attribute and offscreen requirements
- a compact immutable tree with integer links and interned strings
- a deterministic Rust parser trait
- typed conversation, message, document, task, calendar, and page items
- stable input fingerprints that include parser/schema/app/content versions
- output validation for size, item count, parents, cycles, and source-node links

Keeping this inactive makes the first review about contracts and resource bounds.
Capture, database, scripting, and retrieval integrations can land independently.

## 3.1 Parser coverage model

Screenpipe does not need a custom semantic parser for every installed app.
Every app keeps the current generic accessibility and OCR path. A parser is an
optional quality upgrade for screens where sender, thread, document, task, or
event relationships matter.

Prefer shared family implementations with small declarative profiles:

| Family | Shared extraction | Profile-specific details |
|---|---|---|
| Conversation | channel, sender, message, time, draft | message-list anchor, chrome exclusions, sender marker |
| Mail | subject, participants, body, thread order, draft | thread root, collapsed-message rules, compose labels |
| Editor | file, buffer, terminal, project | workbench marker, editor and terminal containers |
| Task | title, project, status, due date, assignee | board/list anchors and field labels |
| Calendar | event, time range, attendees, location | day/week view structure and event container |
| Page/document | title, author, body, selection | article/editor root and navigation exclusions |
| Terminal | command and output transcript | terminal root and prompt markers |

A family parser manifest can match multiple bundle identifiers, executables,
and URL patterns. Native and web versions may still need separate adapters when
their accessibility trees differ, but they should share extraction and output
code. App-only logic should be limited to selectors, stable labels, and known
quirks rather than duplicating the whole parser.

The family parser selects its profile from `ParseContext.app`, so a profile does
not require another runtime or another tree copy. Register an `App` parser only
when an app needs an algorithmic override that cannot fit the shared family.

Only matching candidates run. The registry caps a tree at four candidates; the
normal case is one family parser, or one app override plus one family parser.
This bounds failure-path CPU even if the registry eventually contains hundreds
of app definitions.

### Built-in catalog and reference family parsers

The built-in catalog covers the 47 app targets found in Littlebird 0.82.4. This
is an independent compatibility catalog of public app identities and URL
patterns. It does not copy Littlebird parser implementations.

| Family | Built-in profiles |
|---|---|
| Conversation | Antigravity, Antigravity IDE, ChatGPT, ChatGPT legacy, ChatGPT web, Claude, Claude macOS, ClickUp, ClickUp web, Cursor, Discord, Gemini desktop, Gemini web, Messages, Messenger, Microsoft Teams, Slack, WhatsApp, WhatsApp web, Windsurf |
| Mail | Gmail, Mail, Microsoft Outlook, Spark Desktop, Spark Mail Classic, Superhuman |
| Editor | Antigravity IDE, Cursor, VS Code, Windsurf, Xcode |
| Document | Antigravity IDE, Claude macOS, ClickUp, ClickUp web, Microsoft Outlook, Microsoft Word, Microsoft Word web, Notes, Notion, Obsidian, Pages, TextEdit, Xcode |
| Task | Antigravity, Asana, Asana web, ClickUp, ClickUp web, Microsoft To Do, OmniFocus, Todoist, Toggl |
| Calendar | Calendar, Fantastical |
| Terminal | Ghostty, iTerm2, Terminal, Warp |

Profiles may belong to more than one family because the same app can expose
different semantic surfaces. The registry still runs at most four matching
candidates for one tree.

`EditorFamilyParser` recognizes macOS AX, Windows UIA, and Linux AT-SPI role
aliases, then emits editor buffers and integrated terminal content as `Document`
items. `FamilyParser` supplies conservative conversation, mail, document, task,
calendar, and terminal implementations. Each implementation requires structural
markers before it emits output. App identity by itself always returns
`NotHandled`.

The parsers abstain on unrecognized surfaces and inaccessible editor buffers.
This preserves generic accessibility instead of emitting an empty or misleading
semantic result. Synthetic, privacy-safe fixtures cover every family, and the
editor fixtures cover all three platforms. They are contract fixtures, not proof
that current platform walkers retain every required structural node. Capture
integration remains a separate measured milestone.

### Real-capture replay checkpoint

`CapturedAccessibilityNode` and `adapt_captured_accessibility_tree` can now
replay Screenpipe's existing `accessibility_tree_json` through the compact
arena without committing private capture data. The `replay` example reports
only node and attribute counts, parser selection, output size, heap estimates,
and timings. It never prints semantic content.

A read-only replay over nine recent local target-app frames on 2026-07-24
confirmed the intended fail-open behavior:

- Obsidian and Terminal produced bounded document output.
- ChatGPT, Claude, Mail, Notion, Notes, Messages, and WhatsApp returned
  `NotHandled` rather than guessing.
- The largest sampled tree had 584 retained nodes. Debug-build adaptation was
  under 2 ms and parsing was under 1.4 ms for every sample.
- Most sampled macOS trees had no class names and many depth gaps because the
  current stored tree keeps text-emitting nodes, not every structural
  container.

This is evidence that runtime activation must retain parser-requested
structural containers during the existing walk. Loosening family parsers around
flat text would increase false relationships and is not an acceptable shortcut.
Raw frames and extracted text remain local and are not test fixtures.

## 4. Capture integration

The parser registry returns a `SemanticCapturePlan` before an accessibility walk.
The plan combines every matching fallback candidate's `AttributeSet` and
`OffscreenPolicy` with engine-owned `TreeBudget` and `OutputBudget` hard caps.
This prevents an app override from abstaining after capture omitted an attribute
needed by its family fallback. Parser packs may request less data, but they
cannot raise the engine's resource ceilings.

Platform walkers should append every retained structural node to
`SemanticTreeBuilder` while they perform the existing walk. On macOS the current
walker already batches role, value, title, description, position, and size for
each visited node. Retaining those fields should not add accessibility IPC.
Optional DOM classes and identifiers should be added to the same batch only when
the selected plan requests them.

`TreeSnapshot` now carries a native bundle identifier or executable alongside
the display name so parser selection does not depend on localized app names.
The current text-oriented node list can be adapted for offline replay, but it is
not sufficient for conversation, mail, task, and calendar relationships until
the requested structural containers are retained.

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

This change does not add a migration, database table, background writer, or
semantic write path. Parsed output exists only in memory in tests, replay, and
evaluation. Current `frames`, `elements`, FTS, screenshot, and retention behavior
is unchanged, so this PR reduces prompt tokens but does not reduce disk use.

Today a stored accessibility frame can contain `accessibility_text`, a derived
`full_text`, `accessibility_tree_json`, and normalized `elements`. Exact content
dedup can skip an eligible capture, and `elements_ref_frame_id` can share an
anchor frame's normalized elements, but a stored frame can still repeat its text
and tree JSON. Semantic persistence must not add another per-frame JSON copy.

Do not store one parsed JSON blob per frame. Repeated screens would repeat every
message or document. The normalized persistence contract is:

- `semantic_parse_runs`: parser/version/input fingerprint/status/duration
- `semantic_frame_refs`: frame to parse run, reusing identical projections
- `semantic_items`: immutable, canonical, versioned typed records
- `semantic_observations`: run membership, parent, order, and source node indexes
- external-content semantic FTS

Required keys and constraints:

- `semantic_parse_runs.input_fingerprint` is unique. It includes parser ID,
  parser version, schema version, app identity, and source content hash.
- `semantic_frame_refs.frame_id` is unique, so one frame resolves to at most one
  selected semantic projection.
- Every item has an `entity_fingerprint` and a `version_fingerprint`.
  `semantic_items.version_fingerprint` is unique and immutable.
- Stable and derived items may reuse the same exact canonical version across
  runs. Ephemeral items include the parse-run fingerprint in their entity key,
  so equal message text or position-based keys from different screens never
  merge.
- Only stable identity supports authoritative cross-run entity history. A
  derived entity key is an approximate grouping hint even when exact-value
  reuse is safe.
- Parent, order, parser-local ID, and source-node indexes live only in
  `semantic_observations`. Moving an item within a screen must not duplicate its
  canonical value.
- A changed title, body, actor, time, status, or metadata value creates a new
  immutable item version while preserving the entity fingerprint.

`semantic_item_storage_keys` implements this contract without I/O. It also
domain-separates entity and version hashes and scopes them by parser and app.
The storage adapter must bind the 32-byte values as BLOBs, not hex text.

Do not overload `frames.full_text`, `memories`, or `outputs`. They represent raw
search text, durable user facts, and generated files respectively.

### Raw and parsed retention

Keep both forms only while they serve distinct purposes:

- Before activation and in the metrics-only shadow phase: write no semantic
  data.
- During the measured rollout: keep existing raw/generic evidence and normalized
  semantic records so parser recall and fallback can be compared.
- After a successful parse is durable and the retention/redaction gates pass,
  `lean` mode may keep `full_text` plus semantic items while clearing heavy tree,
  geometry, element, and source-node evidence.
- If parsing abstains or fails, preserve the current generic accessibility/OCR
  path. Never delete the only usable representation.

Storage reduction is therefore a later retention outcome, not an automatic
consequence of parsing. The schema PR must report SQLite page-level bytes for the
same fixed trace in current full, current lean, semantic full, and semantic lean
modes after checkpoint and compaction. It must separately report media bytes,
database bytes, reused parse runs, reused item versions, and parse failures.

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

The crate now includes a privacy-safe fixed context suite covering all seven
shared parser families. It compares persisted accessibility JSON, the current
text-bearing element outline, and semantic plain text with the exact
`o200k_base` tokenizer. Regression tests require every semantic case to retain
all scored task facts while using a smaller complete prompt than both
baselines. An opt-in Pi runner sends the balanced 21-prompt pack to a local or
configured model with tools, project context, skills, extensions, and sessions
disabled. Model accuracy is reported separately from deterministic fact
retention and is never a network or credential requirement for CI.

The synthetic suite verifies the representation contract, not real-app parser
recall. Real-capture evaluation remains gated on retaining parser-requested
structural containers in the platform walk; otherwise most current stored
trees cannot express sender/message, task/status, or event/schedule
relationships.

## 11. Rollout

1. Retain parser-requested structural containers in the existing platform walk
   and replay every family against privacy-safe real-tree fixtures.
2. Add the nonblocking shadow worker with metrics only and no database writes.
3. Add schema, retention, and redaction integration after shadow resource gates
   pass.
4. Add semantic search and MCP output behind a feature flag.
5. Measure token reduction and parser resource use on representative traces.
6. Tighten profiles only from privacy-safe real-tree fixtures when a shared
   parser abstains or emits the wrong structure.
7. Consider signed remote parser packs only after shipped parsers are stable.
