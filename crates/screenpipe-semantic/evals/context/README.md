# Semantic context evaluation

This fixed suite compares three representations of the same seven synthetic,
privacy-safe accessibility trees:

- persisted accessibility JSON;
- Screenpipe's current text-bearing element outline;
- compact semantic context rendered for an AI agent.

The report uses the exact `o200k_base` tokenizer. It scores both context-only
tokens and the complete fixed Pi input prompt, retained task facts, tokens per
retained fact, parser selection, compact-tree heap, and a 1,000-iteration local
adapt/parse/render benchmark for each case. The seven cases cover the seven
shared parser families; catalog tests separately keep all 47 Littlebird target
profiles matched.

Run the deterministic report:

```bash
cargo run -p screenpipe-semantic --example context_eval --locked -- --report
```

Generate a balanced 21-row JSONL prompt pack for a Pi or other model A/B run:

```bash
cargo run -p screenpipe-semantic --example context_eval --locked -- --prompts \
  > /tmp/screenpipe-semantic-context-prompts.jsonl
```

Each case has identical question and expected answer across `raw_json`,
`current_outline`, and `semantic`. Model accuracy must be reported separately
from deterministic fact retention. A model run is intentionally opt-in so CI
never requires credentials, network access, or paid inference.

Run the complete 21-prompt A/B through a local Pi model:

```bash
cargo run --release -p screenpipe-semantic --example context_eval --locked -- \
  --run-pi ollama/screenpipe-gemma4:latest
```

The runner disables tools, extensions, skills, project context, sessions, and
startup network checks. Only the privacy-safe synthetic suite enters the model.

This suite measures AI input efficiency. The database integration separately
tests transactional normalized writes, exact parse-run reuse, immutable item
versions, run-scoped ephemeral items, FTS retrieval, retention cleanup, and
SQLite page growth:

```bash
cargo test -p screenpipe-db --test semantic_storage_test -- --nocapture
```

The storage regression measures incremental semantic pages and write time for
repeated and changing synthetic traces. It does not claim total disk reduction
because opted-in capture still retains existing raw text, tree JSON, elements,
and media.
