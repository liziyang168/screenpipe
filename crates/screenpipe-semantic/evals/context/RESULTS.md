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
Representative compact trees retained 507 to 1,315 heap bytes. In this small
release-mode run, adapt, parse, and render operations were each measured in
microseconds. These tiny synthetic timings are regression signals, not the
older-hardware acceptance benchmark.

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
