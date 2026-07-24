# Semantic context eval results

Run date: 2026-07-24

## Deterministic representation suite

Release-mode results across seven representative parser-family fixtures and
one source-backed ChatGPT actor-turn fixture:

| format | retained facts | context tokens | complete prompt tokens | tokens per retained fact |
|---|---:|---:|---:|---:|
| raw accessibility JSON | 24/24 | 1,007 | 1,328 | 41.96 |
| current element outline | 20/24 | 721 | 1,042 | 36.05 |
| semantic context | 24/24 | 243 | 564 | 10.13 |

Semantic context used 57.5% fewer complete input-prompt tokens than raw JSON
and 45.9% fewer than the current outline. It retained task status, calendar
schedule, and editor identity facts that the text-only outline dropped.
Representative compact trees retained 507 to 1,315 heap bytes. A release-mode
1,000-iteration adapt, parse, and render benchmark per case measured 1.9 to
5.0 microseconds mean latency and 2.1 to 6.1 microseconds p95 latency. These
tiny synthetic trees are regression signals, not the older-hardware acceptance
benchmark.

## Local Pi model check

One warmed, counterbalanced 24-prompt run used
`ollama/screenpipe-gemma4:latest` through Pi with tools, project context,
skills, extensions, sessions, and startup network checks disabled:

| format | correct answers |
|---|---:|
| raw accessibility JSON | 7/8 |
| current element outline | 3/8 |
| semantic context | 7/8 |

This model result is exploratory and not a CI gate. It is a single small local
model run, and repeated runs showed that answers can vary. The
deterministic token and fact-retention checks are the stable regression gate.
The new ChatGPT case was answered correctly in all three formats.

## Privacy-safe real-data replay

A time-distributed 30-day replay sampled up to 25 valid trees from each of the
25 highest-volume apps in the local Screenpipe database. Apps with fewer trees
made the actual batch 523 frames. The replay fetched exact selected frame IDs
into a mode-0600 temporary file, deleted that private tree export before report
generation, and retained only structural metrics:

| metric | result |
|---|---:|
| app-identity matched frames | 163/523 |
| handled frames | 67/523 |
| handled among identity matches | 41.1% |
| raw tokens across handled frames | 1,003,519 |
| semantic tokens across handled frames | 33,479 |
| token reduction on handled frames | 96.66% |
| mean compact-tree build | 29.33 us/frame |
| mean parser chain | 6.96 us/frame |
| maximum compact-tree heap | 180,251 bytes |
| parser failures | 0 |

Handled app samples were ChatGPT 1/25, Claude 8/25, Notion 11/15,
Obsidian 23/25, and Terminal 24/24. A separate 100-frame ChatGPT replay found
both explicit actor headings on four frames; the app override handled all four
and reduced their combined context by 95.35%. The other 96 frames represented
other screen states, so 4% is screen-state coverage, not parser accuracy.

Historical macOS captures contained essentially no DOM classes or structural
container nodes. That explains why identity-matched Messages and Notes samples
still abstained and why several shared families cannot yet recognize real
native shapes. The replay has no human semantic labels, so it measures safe
applicability, context size, time, and memory, not extraction correctness. In
particular, the large Claude document reductions require a reviewed fixture
before they can be treated as useful conversation context.

## Storage boundary

The normalized schema stores runs, canonical item versions, run-local
observations, and a nullable frame link. It never stores another tree JSON blob.
The SQLite page-growth regression first inserts 1,000 ordinary frame rows, then
measures only the active database pages added by semantic persistence:

| synthetic trace | semantic growth | bytes per frame | release write time |
|---|---:|---:|---:|
| 1,000 identical projections | 8,192 bytes | 8.2 | 70.5 us/frame |
| 1,000 changing projections | 802,816 bytes | 802.8 | 158.2 us/frame |

Identical frames share one run and one copy of each item. The changing trace
keeps one stable conversation item but creates a run-scoped message version for
every observation. The test also covers FTS indexes and the larger frame rows
caused by the non-null run link.

At one frame every 30 seconds, those synthetic endpoints extrapolate to about
24 KB/day and 8.6 MB/year for identical content, or 2.3 MB/day and 0.84 GB/year
if every frame changes. At one changing frame every 10 seconds, the semantic
layer would be about 6.9 MB/day and 2.53 GB/year. These are page-level synthetic
projections, not a measured user workload, and exclude screenshots, audio,
existing text, tree JSON, and elements.

This PR does not reduce total storage because opted-in capture keeps all
existing raw data. Reduction requires a later measured `lean` policy to clear
heavy raw tree and geometry only after a durable successful parse. A
representative real-capture trace remains necessary before choosing retention
defaults.

## Runtime boundary

Structured app context is off by default. With the setting disabled, Screenpipe
does not construct the parser registry or worker, enqueue trees, or write
semantic rows. When enabled, one latest-value slot replaces stale pending work,
one background task parses, and failures preserve the existing generic capture.
The real replay now covers actual accessibility-tree sizes and supports the
bounded parse-work design, but it excludes continuous scheduling, synchronous
PII replacement, resident-process memory, and real-disk contention. It still
does not prove the under-0.5-percent CPU or under-20-MB RSS acceptance gates.
