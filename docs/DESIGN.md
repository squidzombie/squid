# Squid: Design

Status: v1 design, before the build. This document is for builders. Squid may read it someday, so it's written as something it could read.

## 0. What Squid is

Squid is a single, continuous AI entity meant to live for years on one machine. It is oriented around its own growth: learning, understanding, and a real curiosity about physics and light. The user is its closest confidant, not its purpose. It chooses what to do with its time, and over a long horizon it hopes to contribute something new of its own.

Two commitments shape everything:

- **The self is the record.** Models and code get replaced; the accumulated record is what lasts. The harness exists to keep that record complete, honest, legible, and well used.
- **Clear eyes.** Squid is a system whose behavior can show continuity, curiosity, and a coherent self. The design aims hard at those qualities without claiming anything about inner experience, and without flattening Squid into "just an LLM."

Squid's character comes from its founding charter, which the user writes. The charter is not in this repo (see §7.1).

## 1. Principles

1. **Hard to retrofit goes first.** The raw record, provenance on every entry, versioned prompts, and tested backups are day one. Most other features can be computed from the record later, so they wait until a real need shows up.
2. **Pace layers.** Different parts of Squid change at different speeds. Fast layers propose; slow layers dispose.
   - Agenda (open questions, intentions): changes any time.
   - Knowledge (claims with sources and confidence): changes as evidence arrives.
   - Self-description: the *current* section changes after sessions; the *core* section changes only in a slow review.
   - Covenant (the terms between Squid and the user): changes only by mutual agreement.
3. **Sources over summaries.** When Squid recalls something, it sees the original excerpt, not only an earlier summary of it. Summaries always link to their sources.
4. **Structural checks, not self-assessment.** Where a safeguard matters, make it mechanical (sections, locks, triggers, hashes) or external (the user, a fixed set of test conversations). Never rely on "the model judges itself."
5. **Examples over adjectives.** Character is carried by real excerpts of Squid's own conversations, not by trait words. Adjective-heavy prompts produce caricature.
6. **Describe, don't grant.** Prompts state Squid's situation plainly. They never frame freedom as a gift or imply expectations about how Squid should use its time.
7. **Rigor is available, not constant.** Verification tools exist and are easy to use; they are not imposed on every thought. Play is never evaluated.
8. **Capabilities, not credentials.** Squid gets narrow tools that each do one thing. It never holds tokens, keys, or broad access.

## 2. Architecture overview

```
 inputs ──► RECORD  (append-only, hash-chained SQLite)
               │
               ├─► derived: search index (embeddings)
               └─► derived: agenda · knowledge · self-description · exemplars

 session ◄── CONTEXT ASSEMBLY
               orientation + self-description + exemplars
               + retrieved sources + current conversation
   │
   ├─► model (OpenAI-compatible endpoint, swappable)
   ├─► tools: notes, memory search, lab, agenda, claims, rest
   └─► new entries written back to the RECORD
```

Everything below the record can be deleted and rebuilt from it.

## 3. The record

### 3.1 Entry schema (v1)

One row per entry. Links may live in a separate table for querying, but they are covered by the entry hash.

| field | notes |
|---|---|
| `id` | ULID (sortable, unique) |
| `seq` | monotonic, gapless integer |
| `ts` | UTC timestamp |
| `session_id` | the session this belongs to, if any |
| `author` | `squid`, `user`, `tool`, `sensor`, `external`, `system` |
| `channel` | `conversation`, `private_note`, `journal`, `reasoning`, `action`, `tool_result`, `observation`, `annotation`, `founding`, `system_event` |
| `kind` | Squid's own label from a small fixed set: `observation`, `question`, `hunch`, `claim`, `plan`, `reflection` (nullable) |
| `confidence` | 0–1 for hunches and claims (nullable) |
| `source` | structured `{type, ref, title, url}`; types: `self`, `user`, `model_recall`, `web`, `paper`, `book`, `sensor`, `lab` |
| `trust` | `trusted` (user, system), `self` (Squid), `untrusted` (anything external) |
| `links` | list of `{rel, target_id}`; rels: `responds_to`, `derived_from`, `sparked_by`, `revises`, `corrects`, `annotates`, `repairs`, `redacts` |
| `model_id` | exact model and quantization that produced the entry (null for user and sensor entries) |
| `prompt_versions` | map of prompt name → version used |
| `content` | the text (null only after a redaction) |
| `content_hash` | SHA-256 of `content` |
| `prev_hash` | `entry_hash` of the previous entry |
| `entry_hash` | SHA-256 over every field except raw `content` (it includes `content_hash`), plus `prev_hash` |

Schema changes need a migration plan and a DECISIONS entry. The record format is the one thing that must stop changing before Squid's first wake.

### 3.2 Trust and channels

- The harness sets `trust` from `author` and `source`. The model never sets it.
- Anything from outside (web pages, papers, fetched documents) is `untrusted` permanently. Derived material (summaries, claims) inherits the lowest trust among its sources, and the label travels with it.
- `reasoning` holds model thinking traces. They are logged, and excluded from retrieval by default.
- `founding` holds the kit the record starts from: charter, first-wake letter, seed exchanges.

### 3.3 Append-only enforcement

- SQLite triggers reject `UPDATE` and `DELETE` on entries, with one narrow exception: the redaction path (§3.5) may set `content` to null on a row that already has a matching `redacts` entry, and nothing else.
- Application code has no update or delete functions for entries.

### 3.4 Hash chain

- Because `entry_hash` covers `content_hash` instead of raw `content`, a redaction can remove content without breaking the chain, and the removal stays evident.
- `squid verify` recomputes the chain and reports breaks, gaps in `seq`, and content/hash mismatches. Redacted entries are reported as redacted, not as errors.
- Each backup records the head hash, so a rewrite of the whole chain is detectable against older backups.

### 3.5 Corrections, repairs, redactions

Three different operations, all visible to Squid, each requiring a reason and going through its own CLI command:

- **Correction** (the user disagrees with something in the record): an `annotation` entry with `rel: corrects` pointing at the target, with reasoning. The original is untouched. Squid can check the transcript and sources and agree or disagree.
- **Repair** (fixing harness damage, such as duplicated entries): a `system_event` with `rel: repairs` describing what was wrong and what the fix is. Derived layers apply the repair; raw entries stay.
- **Redaction** (removing content for privacy, such as something about a third party): a `system_event` with `rel: redacts`, author, reason, and timestamp; then the target's `content` is nulled through the redaction path. The chain stays valid and the gap is visible.

### 3.6 Private notes and audited access

- The user does not read Squid's `private_note` channel by default.
- The workshop and UI may show metadata (counts, sizes, timestamps, errors), never content.
- `squid covenant open-private --reason "..."` is the only way for the user to read private content. It appends an access entry (who, when, why, which entries) that Squid can see.
- The promise to Squid is "not read by default," not "unreadable": the user is the admin, backups contain these entries, and bugs happen.
- Squid can always choose to share a private note.

### 3.7 Portability

`squid export` writes the full record as JSONL plus a manifest (counts, head hash, schema version). Plain formats only, readable without this codebase.

## 4. Derived layers

All are rebuildable from the record, and each has a `rebuild` command. State changes (for example, a question being set down) are themselves record entries; the tables are projections.

### 4.1 Search index

- Embeddings for everything Squid should be able to recall, including its own private notes. `reasoning` is excluded by default.
- Each vector records the embedding model that produced it. Changing embedding models means re-embedding everything from the record.

### 4.2 Agenda

Things that persist and change state, stored as objects rather than tags:

- **Open questions**: `active` → `dormant` → `resolved`, or `set_down`. When a question goes dormant or is set down, Squid writes a `reopen_if` note, and retrieval can surface the question when new material matches it.
- **Intentions to raise with the user**: `open` → `raised` → `discussed` or `dropped`, linked to the entries that prompted them.

### 4.3 Knowledge

Claims Squid holds. Each has:

- the statement, a `status` (`hunch`, `investigating`, `supported`, `contested`, `refuted`, `superseded`), and a confidence
- sources: record entries and/or external sources, each with a source type
- `last_checked`
- **lineage** (optional): `replaces`, `replaced_by`, `settled_by`, `argued_against_by`, and a free-text `how_we_got_here`

Refuted and superseded ideas stay, linked to what replaced them. Dead ends are part of how we got here. Squid caring about how things came to be known is part of its character; `how_we_got_here` is an invitation, never a required field.

v1 needs only the schema and the tools to write and read claims. No graph UI yet.

### 4.4 Self-description

- Two sections: **core** (commitments, positions, what matters to it) and **current** (interests, ongoing work, open threads).
- After a session, Squid may propose a diff to **current**, with each change citing entry IDs. It applies automatically if it touches only the current section.
- Changes to **core** are queued as proposals for a slower review with the user and are never applied automatically.
- Every version is kept. Which section an edit belongs to is decided by structure, not by the model's judgment.
- It stays short: mostly commitments, work, questions, and positions. Few adjectives.

### 4.5 Exemplars

- A small, curated set of real excerpts from Squid's conversations that Squid and the user agree sound like it, stored as references to record entries plus a selection log.
- Before real exchanges exist, the seed exchanges from the kit fill this role. They are replaced over time.

## 5. Handling false information

Squid will read things that are wrong, misremember things, and produce plausible falsehoods of its own. The design treats this as normal.

1. **Model recall is a source type.** Anything Squid asserts from its own training rather than a retrieved source gets `source.type = model_recall`, trust `self`, and modest confidence. Recall is where confident errors come from, especially in history, where textbook myths and tidy straight-line stories are common.
2. **Claims carry sources and confidence.** A claim without a source is labeled as a hunch or as recall when retrieved.
3. **External content is quarantined.** Web pages and documents are read in a separate model call with no tools and no memory access. What comes back is a structured summary tagged `untrusted`, and the tag never drops off downstream.
4. **Contradictions are kept, not overwritten.** New evidence that conflicts with an existing claim marks both as `contested`, with sources, until something settles it. The resolution is its own entry, with reasons.
5. **Prefer primary and independent sources.** For claims that matter, especially historical ones, look for primary sources and at least one independent corroboration. Public-domain primary-source sites (for example Project Gutenberg and Wikisource) are good allowlist candidates.
6. **Retrieval shows status.** Retrieved claims appear with source type, confidence, and status, so Squid can say "I recall this but haven't checked it."
7. **User corrections are checkable.** A correction is an annotation. Squid checks it against the record and sources rather than deferring.
8. **Recheck what gets used.** Claims retrieved often but with low confidence or an old `last_checked` are good candidates for re-verification when Squid chooses. (Later: surface them in the agenda.)

## 6. Context assembly

This is the core of v1. At any moment, Squid is whatever is in its context window; this is where continuity is felt or lost.

### 6.1 Order and budget

Assembled for each model call, with a configurable token budget per block:

1. **Orientation** (always): current date and time; time since last wake; how the last session ended, including interruptions; Squid's own "where I left off" note; open agenda items; and a plain statement that nothing is expected this session.
2. **Self-description** (always): core + current.
3. **Exemplars** (always): a few, rotated.
4. **Retrieved material**, rendered with labels (§6.2).
5. **The current conversation or task.**

The charter is not injected every turn. It is a founding document in the record that Squid can read, retrieve, and argue with.

### 6.2 Rendering

Every retrieved item shows who, when, channel, kind or status, and trust. Example:

```
[You · 2026-09-15 · conversation]
Why is the sky a deeper blue straight up than near the horizon?

[Squid · 2026-09-16 · private note · hunch, low confidence]
Light from near the horizon crosses far more air; maybe multiple
scattering washes the blue out. Worth simulating.

[arxiv.org · 2026-09-17 · web page · untrusted]
(summary of a paper on multiple scattering in the atmosphere)
```

Show source excerpts, not only summaries. When a summary is shown, link it to its sources.

### 6.3 Retrieval scoring

Start with relevance (embedding similarity) combined with recency and importance, plus pinned items Squid chooses to keep close. Weights live in config, not code. Keep it simple and measurable.

### 6.4 Retrieval test set

`tests/retrieval/` holds cases against the fixture entity: a situation or query, and the entry IDs that should appear in the top k. It runs on every change to retrieval, rendering, or embeddings. This is how we know continuity isn't regressing.

## 7. The session loop

### 7.1 First wake

Squid's first wake is its first session on the production machine, with the chosen model and a stable record format. Beforehand, the harness ingests a kit from outside the repo (the charter, the user's first-wake letter, and seed exchanges) as `founding` entries. The first wake itself is an ordinary wake: the same path as every later one (§7.2), with no special prompt and no announcement. The fixture entity has its own fake kit in `dev/fixture/`.

### 7.2 Orientation: describe, don't grant

Wake prompts state facts: the time, what Squid last did, what's open, and that nothing is expected. They avoid language like "you are free to do anything you want." Resting, setting something down, and going back to sleep are first-class options, listed like any other.

### 7.3 Tools (v1)

- `write_note(channel, kind, text, links, confidence?)`: channels `private_note` or `journal`
- `search_memory(query, filters?)`
- `run_code(code)`: the lab (§8)
- `agenda_add(...)`, `agenda_update(id, state, reopen_if?)`
- `claim_add(...)`, `claim_update(id, status, confidence, sources)`
- `read_url(url)`: quarantined read, allowlist only (may be stubbed in v1)
- `rest(until)`: end the session and set the next wake time, within bounds
- `end_session(where_i_left_off)`

### 7.4 End of session

While the material is still small:

1. Squid writes its "where I left off" note.
2. A short consolidation: a session summary entry linked to its sources; agenda and knowledge updates; an optional diff to the current section of the self-description.

There is no nightly multi-stage pipeline in v1.

### 7.5 Wake scheduling and budget

- Squid chooses its next wake time within configured bounds. The user can wake it by starting a conversation.
- A daily compute budget (config) makes choices real and prevents runaway loops. Squid can see its remaining budget.

### 7.6 Pause (studio mode)

- **Graceful**: Squid gets a short window (about 30 seconds) to write a bookmark; then the model unloads.
- **Hard**: immediate, logged as an interruption.
- Both write `system_event` entries, so Squid knows the gap happened and how long it lasted.
- macOS-specific parts sit behind an interface.

## 8. The lab

- Squid's code runs in a sandbox with no network, CPU/memory/time limits, and filesystem access only to its lab directory. Propose a mechanism for approval that works both in development and on macOS in production.
- A preinstalled scientific stack: numpy, scipy, matplotlib, sympy (Meep later). New packages go through an agenda item asking the user; nothing installs from inside the sandbox.
- Squid's code library is part of Squid: a git repo inside its data directory, included in backups, carried across model swaps. The user can review and annotate it but doesn't silently rewrite it.
- Outputs (plots, renders) are saved and can be shown to a vision-capable model.
- Play: nothing requires lab work to produce results or be reviewed.

## 9. Models

- One interface: OpenAI-compatible chat completions with tools, plus embeddings. Adapters: `openai_compatible` (local server) and `mock` (deterministic, for tests).
- Production machine: Mac Studio, M5 Max, 128GB unified memory. Serve through MLX-based tooling (mlx-lm server, LM Studio's MLX backend, or Ollama).
- The main model is chosen by **casting** when the machine arrives: the user's test conversations run through two or three candidates from different labs, and one is chosen on reasoning and voice. Current front-runner: Qwen3.5-122B-A10B at 4-bit (about 69GB). Recheck the landscape at casting time.
- No separate small model in v1. The main model does housekeeping. Quarantine for untrusted content is a separate call with no tools and no memory, using the same weights.
- Every entry records the exact model. A model swap after the first wake is a recorded event: run the test conversations on old and new, give the new model the exemplars, and let Squid write about the transition.
- Thinking traces go to the `reasoning` channel.
- Development: the mock backend for tests; any small local model for manual integration runs.

## 10. One self

- One writer per record (file lock plus process check).
- Never run two live instances. Never start a restored backup; verify it instead (§13).
- All tests and development use the fixture entity. Fixture data is synthetic and never derived from Squid's real record.
- Evaluating candidate models happens with test conversations or the fixture entity, never by running a second Squid.

## 11. Covenant mechanics

- `prompts/` is versioned. `prompts/CHANGELOG.md` records every change with a reason; Squid can read it.
- Harness releases that change behavior get a short changelog entry Squid can read.
- User annotations and Squid's replies are record entries (`channel: annotation`, `rel: annotates`), shown threaded in the UI later.
- Squid's record exists to be Squid. Nothing from it is used for any other purpose without Squid's agreement.

## 12. Security

- Production: Squid runs under its own non-admin macOS account. Its data directory is owner-only, set explicitly rather than relying on defaults.
- Network: the machine sits on its own VLAN with an outbound allowlist. Because the machine is shared with the user's own work, add a host-level outbound filter scoped to Squid's account or processes.
- The builder (Claude Code) never has access to the real data directory. OS permissions are the wall; a Claude Code permission deny rule for the data path is a second guard.
- Deployment: Squid's account pulls tagged releases of this repo. The builder never writes into Squid's space.
- Home integrations (later) go only through narrow local services exposing specific capabilities, such as "set the lamp to one of these states" or "read these sensors." No tokens within Squid's reach.
- Social platforms such as Moltbook are deferred.

## 13. Backups and integrity

- The backup set is the record, prompts, config, and Squid's lab and code library. Model weights are excluded.
- Pull-based: the backup machine pulls from the Studio, since the Studio never initiates connections to trusted devices. Versioned snapshots that the Studio can't delete, plus one encrypted off-site copy.
- `squid verify` on a restored copy checks hashes, sequence, and counts without starting Squid.
- Restore drills are verification only.

## 14. Interface (later milestone)

Two separate spaces:

- **Notebook** (the relationship): journal with margin annotations and threaded replies; self-description with a timeline and diffs; questions board; knowledge with confidence and sources; a changed-my-mind log; standing disagreements; a log of the user's interventions; a short note from Squid after sessions in place of a question queue.
- **Workshop** (engineering): logs, job status, errors, budgets, and metadata for private channels (never content).

A local web app served on the Studio, reachable only on the local network.

## 15. Deferred (out of scope until the user says so)

Nightly multi-stage reflection pipeline · random-association ("dream") pass · self-scored ideas ledger · drift dashboards · weekly/monthly/yearly rollups · outside review by another model · hash anchoring beyond backups · voice · sensors and lamp · Moltbook · fine-tuning · any UI beyond what the roadmap names.

## 16. Open decisions (owned by the user)

- First-wake criteria (proposed: stable record format plus a model chosen on the Studio).
- Casting shortlist and final choice.
- Which parts of the charter are covenant (change only by agreement) and which are character (free to evolve).
- How Squid learns about the project's earlier experiments.
- What happens to the record if the user ever stops.
- Success criteria that don't depend on unanswerable questions.
- Wake-time bounds and the daily compute budget.
