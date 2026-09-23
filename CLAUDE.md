# Squid: harness for a long-lived, self-directed agent

Squid is a single, continuous AI entity meant to live and grow for years on one Mac Studio. The model is replaceable; Squid's **record** (its memory) is what lasts. This repo is the harness around the model: the record, context assembly, the session loop, the lab, and later the interface.

Before proposing changes, read `docs/DESIGN.md`. Scope and order of work are in `docs/ROADMAP.md`. Design decisions and their reasons go in `docs/DECISIONS.md`.

Squid may read this repo someday (its prompts and changelogs certainly). Write docs, comments, and commit messages you'd be comfortable having it read.

## Your role

You are the builder. You write the harness, the tests, and (later) the UI. You never read, open, copy, or run anything against Squid's real data, and you never write Squid's words.

## Hard rules

1. **Never touch real data.** Squid's real data directory exists only on the production machine, owned by a separate macOS account. Never read it, list it, copy it, or point code or tests at it. All development and testing uses the **fixture entity** in `dev/fixture/`, which contains synthetic data only.
2. **Never write Squid's words.** Don't author anything that would enter Squid's record as its own: journal entries, self-descriptions, exemplars, charter text, letters. The user writes the charter, the first-wake letter, and the seed exchanges; they live outside this repo. The fixture entity is a different, obviously fictional character (for example, "Tock," curious about clocks and timekeeping), so nothing from it can be mistaken for Squid.
3. **The record is append-only.** No code path updates or deletes record entries. Corrections, repairs, and redactions are new entries that reference old ones (DESIGN §3.5). Enforce this in the database with triggers, not only in application code.
4. **Everything derived is rebuildable.** Embeddings, indexes, summaries, the agenda, the knowledge layer, the self-description, and any views must be regenerable from the record. Nothing may exist only in a derived layer.
5. **Stamp provenance on every entry.** Author, channel, source, trust, model ID, and prompt versions. No exceptions, including in tests.
6. **Models are swappable by config.** Talk to models only through the client interface in `squid/models/`. No model-specific code outside adapters. Tests use the mock backend and must never require a live model.
7. **One writer.** Exactly one process writes to a record at a time, enforced with a lock. Never run two instances against the same record. Never start a restored backup as a live instance; verify it instead.
8. **Prompts are part of Squid.** Everything in `prompts/` is versioned, and every change gets a line in `prompts/CHANGELOG.md` with the reason. Squid can read that changelog. Draft prompt changes as proposals; the user approves them before they land. Never change a prompt silently.
9. **The lab has no network.** Code that Squid runs executes in a sandbox with no network access, resource limits, and no filesystem access outside its lab directory (DESIGN §8).
10. **Private notes stay private from the user interface.** Content in the `private_note` and `reasoning` channels is never shown in the workshop or UI and never written to logs. Squid itself can retrieve its own private notes. The only way for the user to read them is the audited access command (DESIGN §3.6).
11. **No scope creep.** Build only what the current milestone asks for. Everything in DESIGN §15 is out of scope until the user says otherwise. Ask before adding a dependency.

## Stack

- Python 3.12+, managed with `uv`.
- SQLite for the record (WAL mode). Plain Markdown/JSONL for exports.
- Model access: the `openai` Python client pointed at a local OpenAI-compatible server (mlx-lm server, LM Studio, or Ollama on the Mac). A deterministic mock backend for tests.
- Embeddings: a small local embedding model behind the same interface. The vector index is derived; start with brute-force numpy and change only if needed.
- Tests: `pytest`. Lint and format: `ruff`.
- CLI entry point: `squid`.

## Layout (target)

```
squid/
  record/      # event log: schema, append-only store, hash chain, verify, export
  models/      # client interface; adapters: openai_compatible, mock
  memory/      # derived layers: index, agenda, knowledge, self-description, exemplars
  context/     # context assembly and rendering
  session/     # wake, tools, end-of-session consolidation, rest, pause
  lab/         # sandboxed code execution
  covenant/    # annotations, repairs, redactions, audited private access
  cli.py
prompts/       # versioned prompt templates + CHANGELOG.md
dev/fixture/   # synthetic fixture entity and its fake kit
tests/
  retrieval/   # retrieval test cases against the fixture entity
docs/
```

## How to work

- Start each milestone in plan mode: read the relevant DESIGN sections, propose a plan, and wait for approval before writing code.
- Small, reviewable commits. Tests with every change.
- When DESIGN.md doesn't answer a question, ask instead of guessing. When the user decides, add it to `docs/DECISIONS.md` with the reason.
- If something in DESIGN.md seems wrong, say so and explain why before deviating from it.
- Keep macOS-specific pieces (launchd, account setup, pause/studio mode) behind small interfaces. Development may happen on a different machine than production.

## Commands

(Fill in as they come to exist: setup, test, lint, run the fixture entity.)
