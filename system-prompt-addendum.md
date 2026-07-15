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
