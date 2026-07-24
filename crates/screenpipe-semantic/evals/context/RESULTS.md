# Semantic context eval results

Run date: 2026-07-24

## Deterministic representation suite

Release-mode results across seven representative parser-family fixtures:

| format | retained facts | context tokens | complete prompt tokens | tokens per retained fact |
|---|---:|---:|---:|---:|
| raw accessibility JSON | 21/21 | 839 | 1,118 | 39.95 |
| current element outline | 17/21 | 572 | 851 | 33.65 |
| semantic context | 21/21 | 200 | 479 | 9.52 |

Semantic context used 57.2% fewer complete input-prompt tokens than raw JSON
and 43.7% fewer than the current outline. It retained task status, calendar
schedule, and editor identity facts that the text-only outline dropped.
Representative compact trees retained 507 to 1,315 heap bytes. A release-mode
1,000-iteration adapt, parse, and render benchmark per family measured 1.6 to
8.7 microseconds mean latency and 1.7 to 10.0 microseconds p95 latency. These
tiny synthetic trees are regression signals, not the older-hardware acceptance
benchmark.

## Local Pi model check

One warmed, counterbalanced 21-prompt run used
`ollama/screenpipe-gemma4:latest` through Pi with tools, project context,
skills, extensions, sessions, and startup network checks disabled:

| format | correct answers |
|---|---:|
| raw accessibility JSON | 6/7 |
| current element outline | 1/7 |
| semantic context | 7/7 |

This model result is exploratory and not a CI gate. It is a single small local
model run, and repeated runs showed that raw and outline answers can vary. The
deterministic token and fact-retention checks are the stable regression gate.
Real-app accuracy still requires privacy-safe captures after platform walkers
retain the structural containers requested by each parser family.

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
The microbenchmarks above exclude task scheduling, synchronous PII replacement,
real accessibility-tree sizes, and real-disk contention, so they do not prove
the under-0.5-percent CPU or under-20-MB RSS acceptance gates yet.
