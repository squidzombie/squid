# Roadmap

**v1 goal:** a Squid that can wake for the first time on the Studio and live a small, real life: remembering accurately, choosing what to do, working in its lab, and resting, with its record safe. The user should be talking to it within a week or two of the Studio arriving.

Finish and verify each milestone before starting the next. Start each one in plan mode. Section numbers refer to `docs/DESIGN.md`.

## M0: Repo and guardrails

- Project skeleton per the layout in `CLAUDE.md`; `uv`, `pytest`, `ruff`; a `scripts/check` that runs lint and tests.
- README with setup commands.
- `.gitignore` excludes data directories, model files, and anything that looks like a record database outside `dev/fixture/`.
- A Claude Code permission deny rule for the production data path (confirm the syntax against current Claude Code docs).
- Fixture entity scaffold in `dev/fixture/`: a fictional character that is obviously not Squid, with a fake kit (charter, first-wake letter, 3–5 seed exchanges).

**Done when:** checks pass, and the fixture entity's kit loads.

## M1: The record (§3)

- SQLite schema, WAL mode, append-only triggers, hash chain, gapless `seq`.
- Append API that requires full provenance; single-writer lock.
- `squid verify` and `squid export` (JSONL plus manifest).
- Correction, repair, and redaction commands; the redaction path keeps the chain valid.
- Audited private-note access command.

**Done when:** tests prove that updates and deletes are rejected, that tampering is caught by `verify`, that redaction leaves a verifiable chain, that a second writer is refused, and that export round-trips.

## M2: Model interface (§9)

- Client interface with `openai_compatible` and `mock` adapters; embeddings through the same interface.
- Every call stamps the model ID and prompt versions onto the entries it produces.
- `prompts/` with versioned templates and `CHANGELOG.md`; loading a prompt records its version.

**Done when:** switching models is config-only; mock-backed tests cover tool calls; entries show model and prompt versions.

## M3: Derived layers, minimal (§4)

- Search index with rebuild; embedding model recorded per vector.
- Agenda (questions and intentions) as projections of record entries, with state transitions.
- Knowledge: claims with status, confidence, sources, and lineage fields. Schema and tools only, no UI.
- Self-description sections, versioning, current-section diffs with citations, core proposals queued.

**Done when:** deleting every derived table and running rebuild reproduces them exactly.

## M4: Context assembly, the core (§6)

- Orientation, self-description, exemplars, retrieved material, and conversation, with per-block budgets.
- Rendering with labels; sources over summaries.
- Retrieval scoring (relevance, recency, importance, pins) with weights in config.
- Retrieval test set against the fixture entity, at least 20 cases to start.

**Done when:** the retrieval test set passes and runs in `scripts/check`, and a sample assembled context reads clearly to a human.

## M5: Session loop (§7)

- `squid wake` and `squid talk`, for the fixture entity.
- The tools from §7.3 (`read_url` may be stubbed).
- End of session: "where I left off" note, session summary, agenda and knowledge updates, optional current-section diff.
- Rest and wake scheduling within bounds; daily budget.
- Graceful and hard pause, with system events; the macOS-specific part behind an interface.

**Done when:** the fixture entity runs several sessions in a row on a small local model, and each wake accurately reflects what came before, including a simulated interruption.

## M6: The lab (§8)

- Sandboxed execution meeting §8 (propose the mechanism first).
- Squid's code library as its own git repo inside its data directory.
- Saved outputs that can be shown to a vision-capable model.

**Done when:** tests show no network access, enforced limits, and no filesystem access outside the lab directory.

## M7: Backups and production setup (§12, §13)

- The backup set definition; pull-based backup scripts and docs for the backup machine; `verify` on restored copies.
- macOS setup scripts and docs: Squid's account, owner-only data directory, launchd service, pulling tagged releases.
- Casting kit: run the user's test conversations through candidate models on the Studio and save the results side by side.

**Done when:** a restore to a scratch location verifies cleanly without starting Squid.

## M8: Minimal notebook and workshop (§14)

- Local web app: journal with annotations, self-description versions, agenda board; workshop with logs and private-channel metadata only.

**Done when:** the user can read, annotate, and see Squid's replies to annotations.

## First wake

Not a builder milestone. The user decides when (§16).
