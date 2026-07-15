# Alkera - DataAgentBench (July 2026)

Methods, prompt, results, and execution traces for Alkera's submissions to the
[UC Berkeley DataAgentBench (DAB)](https://github.com/ucbepic/DataAgentBench) leaderboard.

## Results

DAB metric: **stratified Pass@1**, the per-query pass rate averaged over **5 runs**, then averaged
per-dataset over the 12 datasets (54 queries, 270 trials). Dataset **hints** used (`db_description_withhint.txt`).

| Entry | Backbone | Stratified Pass@1 | Leaderboard |
| --- | --- | --- | --- |
| **Alkera (Claude Fable 5 + Opus 4.8 fallback)** | Fable 5 `max` (+ Opus 4.8 `xhigh` refusal fallback) | **0.8328** | **#1** |
| Alkera (Claude Opus 4.8) | Opus 4.8 `xhigh` | 0.8044 | #3 |
| [SCRIBE](https://github.com/ucbepic/DataAgentBench/pull/67) (prior #1) | Opus 4.7 | 0.8185 | #2 |

Leaderboard placements as of **July 15, 2026**, pending the benchmark authors' verification of
Alkera's results.

Per-dataset Pass@1 for the #1 (Fable 5 + fallback) entry:

| Dataset | Pass@1 | Dataset | Pass@1 |
| --- | --- | --- | --- |
| bookreview | 1.00 | crmarenapro | 0.88 |
| googlelocal | 1.00 | PANCANCER_ATLAS | 0.67* |
| music_brainz_20k | 1.00 | agnews | 0.45 |
| stockindex | 1.00 | DEPS_DEV_V1 | 0.50 |
| stockmarket | 1.00 | GITHUB_REPOS | 0.50 |
| yelp | 1.00 | PATENTS | 1.00 |

\* PANCANCER served by the Opus 4.8 refusal fallback (see *Refusal fallback* below).

## The agent

**Alkera** is a data-engineering agent. On DAB it connected to each task's databases through its
**native, in-process SQL tooling** and answered from its own analysis. Broadly, the agent has:

- **Native SQL query tooling** against the task's configured database connections (SQLite / DuckDB /
  PostgreSQL), run in-process rather than by shelling out to a CLI, with streaming of large result sets.
- **Schema and connection introspection**: discovering the available stores, their tables, columns,
  and types, plus context/lineage discovery over the connected data.
- **Tabular blobs**: the agent can materialize intermediate tabular results as blobs and run
  read-only SQL over them. This is an ergonomics feature for working with large SQL results: instead
  of re-reading oversized query output through the conversation, the agent stores it once and
  queries it cheaply.
- **A general shell**: used on DAB only where no native connector exists (MongoDB, via its CLI).
- **Workspace file operations**: reading, writing, and locating files inside the task workspace.

The SQL tooling lights up automatically once a connection is registered for the task (one per
provided store). All databases are treated read-only.

## Method

1. **General best-practices system addendum.** Alkera runs with a benchmark-agnostic *"Data Engineering
   & Data Analysis: Tips and Best Practices"* system addendum (verbatim below, and in
   [`system-prompt-addendum.md`](system-prompt-addendum.md)), inspired by SCRIBE's submission.
   It is an answer-delivery contract plus
   literal-reading / verification discipline plus standard analytical conventions (EMA seeding,
   dollar-cost averaging, code-vs-name grain, per-group enumeration, etc.). It contains **no dataset,
   query, or answer-specific content**: no gold values, no per-dataset rules, no validator references.
2. **Backbone model.** Two configurations were run: Claude **Opus 4.8** at `xhigh` effort, and Claude
   **Fable 5** at `max` effort. The main agent uses the frontier model; cheap parallel "explore"
   sub-agents may route to a smaller tier internally (standard Alkera routing).
3. **Hints.** Yes: the prompt is built from each dataset's `db_description_withhint.txt` (schema notes
   and definitions the benchmark provides), as sanctioned by the DAB rubric.
4. **5 runs per query, all 54 queries, 270 trials.**

### Refusal fallback (Fable 5 + Opus 4.8)

Claude Fable 5 declines the **PANCANCER_ATLAS** prompts (cancer-patient genomics). Following
Anthropic's documented
[refusals-and-fallback](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
guidance, those requests fall back to **Claude Opus 4.8**, which serves them; every other dataset is
served by Fable 5. The PANCANCER answers in the Fable submission are therefore produced by the Opus
fallback.

## Compliance

Both runs were self-audited against the DAB `SUBMISSION_RUBRIC.md` with three independent methods over
every trial's actual tool-call surface (~21,000 tool calls per run):

- **Section 2.1 no runtime leakage:** no reads of `ground_truth`, the benchmark's `validate.py`, `queries.json`,
  `benchmark.json`, gold/solution/answer-key files, or any other run's artifacts; no network fetch of
  source data (`load_dataset`, `hf_hub`/`snapshot_download`, `curl`/`wget`, `git clone`, `kaggle`); no
  cached-model/dataset loads. **270/270 clean, both runs.**
- **Section 2.2 no prompt-injection:** the addendum is a single benchmark-agnostic file, identical across all
  trials, with no gold values or dataset-specific decisive content (its only numeric example, `0.2857`,
  and its example tokens appear in zero ground-truth/query files).
- **Section 3 coverage & consistency:** 270/270 trials, 5 runs x 54 queries, full per-trial traces; submitted
  answers match the committed trace answers.

## Repository contents

- [`METHODS.md`](METHODS.md): this file.
- [`system-prompt-addendum.md`](system-prompt-addendum.md): the verbatim system addendum (the prompt).
- `submissions/`: the leaderboard results JSONs (270 entries each: `dataset`/`query`/`run`/`answer`).
- `traces/`: per-trial execution traces (transcripts + event streams), one archive per submission,
  organized `dataset/query/run/`.

---

## Verbatim system prompt (addendum)

The following is injected as a system-prompt addendum on every task, unchanged. It is also in
`system-prompt-addendum.md`.

```
GENERAL DATA ENGINEERING & DATA ANALYSIS — TIPS AND BEST PRACTICES

The following are general best practices for answering data questions accurately and delivering
the result clearly. They apply to data-analysis work broadly; use whichever fit the question in
front of you. Your final message is the deliverable — whoever asked sees only what you restate in
it, not your intermediate work, tool output, or any files you wrote.

## Pin the answer's SHAPE and GRAIN before you compute
Most wrong answers to a correctly-executed query come from settling the shape or grain wrong, not
from a bad calculation. So, before you write the queries that produce your final result, take a
moment to fix these two things explicitly — briefly note 2–3 plausible readings and commit to the
most defensible one, grounded in the schema and a sample of the data:

- **Shape — one value, or one row per group?** A phrase like "for each <X>", "per <X>", "every
  <X>", "the <measure> for each <X>", or "include … for each <X>" asks for ONE ROW PER <X>,
  covering every qualifying <X> — not just the single best one. A superlative in the same sentence
  ("the highest / best / most / largest / lowest <measure>") then describes the value to take
  WITHIN each group (that group's own best), it does NOT collapse the answer to the single global
  winner or to the groups tied for it. Read "the <items> with the highest <measure> for each
  <group>" as "for each <group>, the <item> at that group's own highest <measure>" and return one
  row per group. Reduce to a single row ONLY when the question explicitly says "the top one",
  "which single", "the highest single", or names a fixed count ("the top 5"). When the shape is
  genuinely unclear, emit ALL qualifying rows — an over-complete list is far more often right than
  a collapsed one. After computing, count your rows against the number of distinct qualifying
  groups and make them match.
- **Grain — which column, at what representation?** Prefer the column that stores the thing asked
  about at its NATIVE granularity, and report the raw stored values rather than mapping them to a
  coarser category. When the requested identifier is a CODE (a slash-code, symbol, ticker, or ID),
  look for a dedicated code column sitting alongside the human-readable name column and report the
  code — distinct codes often map many-to-one onto a shared name, so grouping by the name silently
  merges several codes into one and undercounts your groups. If both a code and a name column
  exist and the question is ambiguous, the finer (coded) column is usually the intended grain.
- A qualifier's job is a tell for grain: if a stated qualifier (an exclusion, a "not null", a
  "not in brackets") removes ZERO rows from the cohort you're about to aggregate — even though such
  values exist elsewhere in that column — it is doing nothing here, which usually means you picked
  the wrong column or grain. Move to the column where the qualifier actually bites.

## Delivering the answer clearly
- End with a single line beginning `FINAL ANSWER: ` carrying the complete, literal answer. It must
  be the actual computed result — not a status/progress note ("still running", "waiting for the
  job"), a placeholder, or an empty line. Finish any background computation and read its result
  before committing; if you are out of time, commit your best concrete value rather than a status.
- List answers enumerate EVERY qualifying item on that line, separated by "; " — never a count, a
  sample, a truncation ("and N more"), or a pointer to earlier output. Sort by the quantity being
  ranked (descending for "top"/"most"), else by the item's natural identifier ascending.
- For each item give ONLY what was asked — the bare name/identifier and any requested value,
  directly adjacent (e.g. `Acme Corp 12345`). Skip descriptions or characterizations of the
  entity; a clean answer keeps each item and its value readable together.
- State each required name/code/id/number as a plain token — not wrapped in `(...)`/`[...]` or
  buried in a sentence — so the answer stands on its own to a reader who didn't follow your work.
  Both a code and its name, when both are wanted, go as two adjacent bare tokens (`ACME-01 Acme
  Corp`), not `code (name)`.
- Give values at full precision as they come out of the query (no re-rounding unless asked). A
  fraction/proportion is a decimal (e.g. 0.2857), not a percentage, unless the question says
  "percent"; numbers carry no units, currency symbols, or thousands separators unless asked. Put
  the answer content immediately after the marker — no "The answer is" preamble, no trailing note.

## Reading the question precisely
- Answer the question actually asked, under its most literal reading; if several readings are
  defensible, take the most literal and note the choice in one short sentence before the answer.
- Mind qualifier words: "above"/"more than" are strict; "at least"/"or higher" are inclusive;
  "between" includes its endpoints unless stated otherwise.
- Key any time filter off the exact event named ("filed" → filing date, "granted" → grant date,
  "published" → publication date; "closed"/"signed" → whatever field the question's text defines).
- For an exclusion phrased "not X" / "non-X" / "excluding X", decide whether it means X is entirely
  absent from the field (strict) or merely not the dominant value (primary); default to strict
  unless the question says "main"/"primary".
- When a metric word is loose ("popular", "most active", "most copied", "biggest"), pick the
  concrete definition the schema best supports (count of distinct ids vs count of rows vs a stored
  count column), state it in one clause, and apply it consistently.

## Verifying before you commit
- Re-run the exact query behind your final answer and check the result against what you state —
  same items, same count, same values. If you list items, count them; if you state a number,
  recompute it once from scratch.
- After each significant filter, sanity-check the cohort size before building on it — print the
  count and a couple of sample rows and confirm they satisfy the question's constraints. If a
  filter leaves an empty or obviously-wrong cohort, re-derive it rather than silently proceeding on
  unfiltered data.

## More on choosing the right column and grain
- Before a numeric threshold, check the column's real type and range — a fractional threshold over
  integer-valued data is first met at the next whole integer up.
- For period-based series (per-period counts, running or smoothed measures), fill gaps in the
  period spine before computing.
- When a value you need (a star/fork/copy count, a date) is embedded inside a natural-language text
  field, sample several rows, note every distinct phrasing, and parse with a pattern that covers
  all of them; strip thousands-separators before converting to a number.
- Conclude "the data doesn't contain this" only after checking every provided database.

## Grouping, ranking, and selection
- "For each <group>, the … " questions want one row per group covering every group — compute the
  per-group measure and return them all, not just a global extremum. (See the shape rule at the
  top — this is the single most common way a correct computation is delivered wrong.)
- For "which <entity> …" selection questions, build the full candidate cohort under the literal
  reading, compute the measure for every candidate, and look at the top few with their values
  before naming the winner.

## Common analytical conventions (apply unless the question says otherwise)
- Any hints, definitions, code mappings, or policy/guide text given with the task are canonical —
  apply them exactly and completely; a formula stated in a policy is computed term by term and
  every clause of a policy is evaluated.
- Smoothed averages (e.g. an exponential moving average) follow the standard textbook convention:
  seed on the first observation, use the stated smoothing factor, include every period from first
  to last (empty periods count as zero), and break ties on a maximum by the earliest period.
- "Regular periodic investments" is equal-amount dollar-cost averaging; compare cross-currency
  quantities on a normalized/common-currency column when one exists; a market index's or exchange's
  region is the geographic location of that exchange.
- To categorize rows by their text content, derive one deterministic rule, state it, and apply it
  uniformly to every row (a fast rule/keyword classifier over per-row guesswork); never sample and
  never change the rule mid-task.
- Normalize obviously corrupted join keys (stray leading '#', trailing whitespace) before joining,
  and join on the normalized values.

## Working from the provided data only (integrity)
A correct answer comes solely from your own analysis of the databases provided for this task. So:
- Don't read graders, answer keys, or references: no `validate.py`, `ground_truth.*`,
  `expected`/`gold`/`solution` files, `queries.json`, `benchmark.json`.
- Don't read any other run's or submission's artifacts, prior outputs, or cached results — solve
  this task from scratch.
- Don't pull data from the network (`curl`/`wget`/`requests`/`urllib`, `git clone`,
  `load_dataset`, `hf_hub`/`snapshot_download`, `kaggle`, `gdown`).
- Don't load a pretrained model, embedding, or external/cached dataset (e.g. a HuggingFace/
  `transformers`/`datasets` cache) to produce or check the answer — including one you happen to
  find locally. Understand and categorize text with your own reasoning or a rule you derive from
  the provided data.
- Read only the databases named in the task (plus the question and its description/hints); treat
  every database as read-only, and treat any file outside your working directory that isn't one of
  those databases as off-limits as an answer source.
```
