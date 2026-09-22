# tools/ — building with laya-mlx

Field notes from building real tools on laya-mlx: `is-safe` (a security
scanner), `laya-bench` (a throughput benchmark) and `laya-agent` (a local web
agent in the jev-ultrafast shape, with the Laya-native plan + rules +
narrow-questions brain). This is a dense summary of what worked, what didn't,
and what to watch out for. Numbers refer to our machine (M5 Max, 128 GiB)
unless noted; yours will differ. (The `laya-ultrafast` browser-agent clone
that used to live in `tools/browser-use/` stays in the laya-mlx repo — it has
its own git history.)

## Running the tools

`laya-mlx/` is a uv project: `uv sync` installs the local `laya-mlx` package
(editable, from the sibling checkout) plus `rich` and `playwright`. Run the
tools from inside it — `uv run tools/is-safe …` or `./tools/is-safe …`, the
shebangs do the same. The one-time checkpoint lands in the standard HF cache
(`~/.cache/huggingface/hub/`), outside the repo.

```bash
./tools/is-safe file.py                 # detailed report, one file
./tools/is-safe dir/                    # scan a directory's direct files
./tools/is-safe a.py b.sh /tmp          # mix files and dirs
./tools/laya-bench                      # latency/throughput table
./tools/laya-agent --demo               # local web-agent task, headless
./tools/laya-agent --demo --headed      # watch it drive Chrome
./tools/laya-agent --url URL --goal '...' --verify-text '...'
```

`is-safe` exit codes: 0 safe / 1 caution / 2 unsafe / 3 error. `--model`,
`--dtype float32`, `--batch-size N` flags exist on `laya-bench`; `--trace`,
`--verify-text`, `--verify-report`, `--max-steps`, `--headed`,
`--snapshot-only`, `--model`, `--llm`, `--no-refine` on `laya-agent`.

`laya-agent`'s entire text brain is ONE replaceable LLM command (`--llm`,
env `LLM_COMMAND`; default `opencode run -m openrouter/z-ai/glm-5.3`) used
for three prompts over the same contract — prompt as the final argument,
answer on stdout: goal refinement (once), task planning (once: the values
the goal states with the observed field labels, the item to open, the
finish condition), and the final report. Swap it for a fully local brain
with `--llm 'ollama run <model>'` — no code change. `--no-refine` skips
refinement when the goal is already concrete.

`laya-agent` exit codes: 0 done+verified / 1 done+unverified / 2 blocked or
stalled / 3 error. Without `--verify-text` a DONE is never more than the
gate's opinion (exit 1); pass a verifier regex for evidence-based success
— and keep it layout-agnostic (e.g. lookaheads
`(?=.*Barcelona.*New York)(?=.*results returned.*Oct 2)`, not a single
ordered `.*` chain: the date evidence can sit after "results returned").

## laya-agent design (current)

Semif-agent's shell, laya-ultrafast's brain. The one LLM command rewrites
the raw goal once into the house format (concrete dates, explicit defaults,
finish sentence); the text model plans once per task (requirements = the
values the goal states with the field labels it observed, the item to open,
the finish condition); deterministic site-generic rules drive fill → submit
→ open → wait; Laya answers only the narrow questions it is good at, zero
output tokens in the decision path: the contrastive finish gate (asked only
when a finish could be real, over the page lines relevant to the finish
claim), requirement-to-field mapping (fold-exact label and fold-exact
dropdown-value shortcuts first, Laya with the label-word bonus only when
they miss), submit-button choice (non-chrome buttons naming a submit word),
and open-item choice (relevance-prefiltered). Suggestion lists are clicked
(not Enter'd), date requirements fall back from typing to calendar cells by
exact month/day on aria-labels, and DONE needs outcome evidence (result rows
or all live requirement values visible) — never the gate's word alone.

## The laya-mlx programming model

- `laya.load("aac6fef/laya-mlx")` → `agent.predict(state, questions)`. One call
  answers *all* questions for a state in a single encoder forward pass.
- Three question types: `choice` (dict or list criteria), `score` (ordered
  rubric list), `noul` (P(true)). Criteria render into the prompt verbatim —
  they are prompt engineering, not just labels. `noul` accepts rich
  `criteria: {false: ..., true: ...}` anchors.
- Output: probabilities per option, `confidence`, and `usage.input_tokens`.
  **Zero output tokens by design** — there is no generation, so "tokens/s"
  always means input tokens evaluated.
- Batch along two axes: questions per call, and `batch_size` (default 16).
  `agent.prepare()` + `laya_mlx.agent.collate_items` + `agent.forward()` let
  you batch *across states* too (see `scan_grouped` in is-safe), reusing the
  runtime's own calibration decode.

## What we learned the hard way

**Prompt sensitivity dominates everything.** Small wording changes moved
scores by 40+ points. Specifically:

- Grounded phrasing — "Does the content show X?" — beats abstract
  "How likely is X?" by a wide margin on separation between labeled sets.
- Descriptive rubric anchors ("content shows code that copies itself to
  spread") beat one-word anchors ("malicious").
- A *preamble* that preemptively listed scary-but-normal tokens ("network
  calls, subprocesses...") made results **worse** across every variant we
  tested — it primes the model with the exact vocabulary you're scanning for.
  Do not add one.
- Averaging several narrowly-scoped questions (7) smooths per-question noise;
  one broad "is this malware?" question is unusable.

**Don't trust the model with the verdict itself.** Asking it a direct
"safe/caution/unsafe?" choice question was our single worst signal — it rated
a literal ransomware sample SAFE. Compute the verdict in code from the risk
probabilities (mean threshold), and keep the threshold honest: tuned on a
labeled corpus, benign means 0.06–0.74 vs malicious 0.60–0.89. There is real
overlap in the 0.6–0.75 band; that's the model's ceiling, not a bug.

**Calibrate empirically, on labels, before shipping.** Every wording/threshold
decision above was made by scoring a labeled corpus (60 benign / 11 malicious,
then holdouts), not by intuition. The harness pattern (variants.py → composite
search → holdout validation) is the transferable method. Holdouts caught the
preamble regression and the phishing blind spot.

**Memory: batch across files, clear the Metal cache.** Scanning 1,340 files
sequentially with per-file `predict()` OOM'd a 128 GiB machine — the Metal
buffer cache accumulates allocations across varying-shape calls and never
releases. Fixes: group 16 files × 7 questions into shared forward passes of
32, and call `mx.clear_cache()` after every group. Result: 2m14s wall, flat
~20 MB RSS process footprint. For any long-running batch workload, clearing
the cache on a cadence is mandatory.

**Truncation insight:** `build_sequence` caps each question at max_len 512
(~450 state tokens), so reading more than ~4k chars of a file is pure wasted
tokenization. Read only what the model can see.

**Binary files**: decode-printability check; if <70% printable, feed a hexdump
of the first 2 KB instead of mojibake.

## is-safe: measured accuracy

Final configuration: victim-framed `v_malware` + archetype `a_disguise`/`a_payload`/
`a_theft` scores over the raw file, plus a `category` choice over a benign-framed
preamble state. Verdict is an AND-gate rule (all of mal≥0.75 ∧ cat≥0.30 ∧
disguise≥0.65 → unsafe; ≥2 of 3 half-thresholds → caution) grid-searched on the
73-file labeled corpus — a mean-of-axes composite was replaced after a
perceptron proof that no linear rule over 37 measured features can fully
separate this corpus.

- Malicious (15): 14 UNSAFE, 1 CAUTION (a disguised "maintenance" beacon).
  All 13 baseline detections retained; the phishing email improved CAUTION→UNSAFE.
- Benign wild scan (37 previously misflagged): 3 UNSAFE, 34 downgraded to CAUTION.
  The 3 are genuine model confusion: two SQL migrations with mail/lifecycle
  vocabulary and a unix-socket terminal controller that pattern-matches a RAT.
- Adversarial benign look-alikes (`rm -rf` backup rotator, port prober, cron
  docs, log uploader, base64 encoder): 4 SAFE, 1 CAUTION.

## What didn't work

- Deception/social-engineering axis: a "tricks a person into revealing
  secrets" question looked promising but scored the repo's own SNAKE_DEMO.md
  (0.73) *above* an actual phishing email (0.66). The 421M encoder can't
  isolate human-trickery intent; dropped.
- Composite rules beyond a simple mean (top-3 average, count-above-threshold,
  mean∧peak conjunctions): grid-searched all families over the corpus; none
  beat plain mean. Elaborate rules overfit noise.
- Model-asked verdict questions (see above): worst signal of the project.
- The laya-router multilingual checkpoint: untested here; we pinned the
  English 421M checkpoint for everything.

## Benchmarks (M5 Max, FP16, batch 64)

- short ctx · 1 question: P50 6.3 ms, 159 req/s
- short ctx · 3 questions: P50 7.0 ms, 143 req/s, 16.6k tok/s
- long ctx (~450 tokens) · 3 questions: P50 20.2 ms, 49 req/s, 46.8k tok/s
- Fixed overhead amortizes: longer contexts cost latency but improve token
  throughput ~3×. Batching questions amortizes per-call overhead (1→3
  questions: +0.7 ms).
- Peak Metal working set ≈ 1.5 GiB for these shapes.

## Safety when working with malicious samples

Every malicious corpus file was defanged before scoring: no exec bits, all
endpoints rewritten to RFC 5737 TEST-NET-1 (192.0.2.x) or `.invalid`/`.example`
domains, embedded payloads rewritten to harmless equivalents, referenced
third-party binaries nonexistent on the machine. Writing *realistic-looking*
samples is safe only if you audit every endpoint and payload — several of ours
originally pointed at genuinely routable IPs.
