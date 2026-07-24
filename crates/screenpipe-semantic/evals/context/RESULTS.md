# Semantic context eval results

Run date: 2026-07-24

## Deterministic representation suite

Release-mode results across seven representative parser-family fixtures and
one source-backed ChatGPT actor-turn fixture:

The registry now contains 21 parser implementations covering all 47 cataloged
Littlebird targets through shared families plus exact app overrides.

| format | retained facts | context tokens | complete prompt tokens | tokens per retained fact |
|---|---:|---:|---:|---:|
| raw accessibility JSON | 24/24 | 1,007 | 1,328 | 41.96 |
| current element outline | 20/24 | 721 | 1,042 | 36.05 |
| semantic context | 24/24 | 243 | 564 | 10.13 |

Semantic context used 57.5% fewer complete input-prompt tokens than raw JSON
and 45.9% fewer than the current outline. It retained task status, calendar
schedule, and editor identity facts that the text-only outline dropped.
Representative compact trees retained 507 to 1,315 heap bytes. A release-mode
1,000-iteration adapt, parse, and render benchmark per case measured 1.69 to
4.93 microseconds mean latency and 1.79 to 5.04 microseconds p95 latency. These
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

A time-distributed 90-day replay sampled up to 100 valid trees from each of the
50 highest-volume apps in the local Screenpipe database. Apps with fewer trees
made the actual batch 1,645 frames across 36 apps. The replay fetched exact
selected frame IDs into a mode-0600 temporary file, deleted that private tree
export before report generation, and retained only structural metrics:

| metric | result |
|---|---:|
| app-identity matched frames | 424/1,645 |
| handled frames | 201/1,645 |
| handled among identity matches | 47.41% |
| raw tokens across handled frames | 2,843,140 |
| semantic tokens across handled frames | 135,288 |
| token reduction on handled frames | 95.24% |
| mean compact-tree build | 19.37 us/frame |
| mean parser chain | 4.41 us/frame |
| maximum compact-tree heap | 180,251 bytes |
| parser failures | 0 |

Handled app samples were ChatGPT 6/100, Claude 19/62, Messages 43/67,
Notion 11/15, Obsidian 98/100, and Terminal 24/24. The Messages override used
exact native balloon/title identifiers already present in historical text-node
captures, handled 64.18% of its sampled screens, and reduced their context by
94.30%. The 100-frame ChatGPT sample found explicit actor headings on six
frames. These rates measure screen-state coverage, not parser accuracy.

Historical macOS captures contained essentially no DOM classes or structural
container nodes. The opt-in walker now keeps a bounded parser-only structural
sidecar in memory, including stable AX and DOM identifiers, while the persisted
raw tree stays text-only. New Slack, WhatsApp, task, and document overrides need
fresh captures with that structure, so this historical replay cannot measure
their applicability yet. Notes still safely abstains because the current
historical shape lacks its exact body marker. The replay has no human semantic
labels, so it measures safe applicability, context size, time, and memory, not
extraction correctness. In particular, the large Claude document reductions
require a reviewed fixture before they can be treated as useful conversation
context.

## Storage boundary

The normalized schema stores runs, canonical item versions, run-local
observations, and a nullable frame link. It never stores another tree JSON blob.
The SQLite page-growth regression first inserts 1,000 ordinary frame rows, then
measures only the active database pages added by semantic persistence:

| synthetic trace | semantic growth | bytes per frame | release write time |
|---|---:|---:|---:|
| 1,000 identical projections | 8,192 bytes | 8.2 | 72.1 us/frame |
| 1,000 changing projections | 802,816 bytes | 802.8 | 156.4 us/frame |

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
On one fresh Arc window, 30 warmed walks visited the same 38 nodes: the baseline
averaged 4.4 ms and semantic capture averaged 4.7 ms. Semantic capture retained
five transient structural nodes while persisted raw tree JSON remained exactly
4,987 bytes in both runs. This is a single-window development-machine check,
not the older-hardware acceptance benchmark.

The real replay now covers actual accessibility-tree sizes and supports the
bounded parse-work design, but it excludes continuous scheduling, synchronous
PII replacement, resident-process memory, and real-disk contention. It still
does not prove the under-0.5-percent CPU or under-20-MB RSS acceptance gates.
