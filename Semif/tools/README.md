# tools/ — building with SemIf

Field notes from porting laya-mlx's tools to SemIf's typed readout and from
building a web agent on the same pattern: `is-safe` (a security scanner),
`semif-bench` (a scoring benchmark), and `semif-agent` (a jev-ultrafast-style
browser agent). This is a dense summary of the design, measured numbers, and
what differs from the originals. Numbers refer to our machine (M5 Max,
128 GiB); yours will differ.

## Running the tools

`Semif/` is a uv project: `uv sync` installs the local `semif-phase1[mlx]`
package (editable, from the sibling checkout) plus `rich` and `playwright`.
Run the tools from inside it — `uv run tools/is-safe …` or
`./tools/is-safe …`, the shebangs do the same. The pinned 9 GB Qwen3.5-4B
checkpoint lands in the standard HF cache, outside the repo.

```bash
./tools/is-safe file.py                 # detailed report, one file
./tools/is-safe dir/                    # scan a directory's direct files
./tools/is-safe a.py b.sh /tmp          # mix files and dirs
./tools/semif-bench                     # direct/shared latency/throughput table
./tools/semif-agent --demo              # local web-agent task, headless
./tools/semif-agent --demo --headed     # watch it drive Chrome
./tools/semif-agent --url URL --goal '...' --verify-text '...'
```

`is-safe` exit codes: 0 safe / 1 caution / 2 unsafe / 3 error. `--model`,
`--revision`, `--max-chars` flags exist on `is-safe`, plus `--dump-signals`
(create-only JSONL of every axis/category probability, for offline rule
fitting); `--model`, `--revision`,
`--iterations`, `--warmup`, `--max-tokens` on `semif-bench`; `--trace`,
`--verify-text`, `--max-steps`, `--headed` on `semif-agent`.

## The SemIf programming model

- A decision row is `{id, state, question, options: [{id, description}...]}`.
  The option **descriptions are rubric anchors** rendered into the prompt — they
  are prompt engineering, not just labels.
- Probabilities come from answer-token logits (the option letters A/B/C/...):
  conditional scores, uncalibrated as confidence, **zero output tokens by
  design** — so "tokens/s" always means input tokens evaluated.
- Three MLX modes: `direct` (one decision per forward pass), `serial` (reuse an
  identical consecutive state's prefix), `shared` (all decisions over one
  *exact* state: one prefill, batched suffixes).
- `encode_prompt` enforces the prompt limit **without truncation** — tools must
  truncate input content themselves (is-safe caps at `--max-chars`, default 4000).

## How the ports differ from laya-mlx

- **No cross-file GPU grouping.** SemIf's shared mode requires one exact shared
  state, so is-safe batches a file's questions (5 axes in one shared pass) and
  walks files sequentially. laya could group 16 files into shared batches.
- **Two passes per file.** Five score axes (malware, disguise, payload,
  theft, deception) run in one shared batch over the raw state; the category
  decision runs over a benign-framed preamble state (its own state, so its
  own forward pass).
- **Verdict rule is locally calibrated** (2026-09-22):
  `unsafe = mal ≥ 0.75 ∧ cat_harm ≥ 0.85`;
  `caution = (mal ≥ 0.50 ∧ cat_harm ≥ 0.50) ∨ deception ≥ 0.60`, where
  `cat_harm` = category p(attack) + p(social). The laya AND-gate (which
  required disguise) was replaced after per-axis anchors made disguise an
  honest, non-universal signal. Disguise/payload/theft are report-only
  explanatory axes.

## is-safe: measured accuracy (2026-09-22)

Corpus: 15 defanged malicious files (TEST-NET/`.invalid`, no exec bits) and
188 benign files (39 known-tricky cache/ops files + 150 random /tmp files).

- Malicious (15): **15/15 UNSAFE**, including both phishing files the first
  port missed (phishing routes to the `social` category, so the signal is
  attack+social, plus a dedicated deception axis at 0.99 vs 0.12 benign max).
- Benign (188): 0 unsafe, 3 caution (probe/ops scripts that talk about
  probing external systems — semantically ambiguous; CAUTION is the honest
  verdict). The tricky cache-* set improved from ~6 caution to 3.
- Wild true positive: a real credential stealer left in /tmp scored
  mal 0.99 / theft 1.00 → UNSAFE during benign-corpus sampling.
- Separation: evil min mal 0.90 vs benign max 0.63; the feasible rule region
  spans mal ∈ [0.65, 0.85] × harm ∈ [0.70, 0.95] with 15/15 detection and
  zero benign unsafe at every point. No holdouts yet — grow the corpus
  (Windows tooling, packed binaries, non-English lures) before trusting
  the exact thresholds.

## semif-bench: measured (M5 Max, MLX, BF16, 15 iterations)

| Workload | tokens/req | P50 | P95 | req/s | tok/s |
|---|---:|---:|---:|---:|---:|
| direct · 1 question · short ctx | 155 | 48.4 ms | 48.9 ms | 20.7 | 3,202 |
| shared · 3 questions · short ctx | 299 | 110.8 ms | 111.7 ms | 9.0 | 2,697 |
| shared · 3 questions · long ctx | 3,977 | 759.3 ms | 779.7 ms | 1.3 | 5,273 |

- Long contexts amortize fixed overhead: tok/s improves ~2× short→long.
- Unlike laya (an encoder, where batching questions was nearly free), shared
  mode pays cache-branch replication per question; on short states direct
  mode wins req/s. Shared mode pays off in tokens/s as state grows.
- Peak Metal working set ≈ 10.0 GiB (weights + caches + branches).
- The backend caps MLX's inactive allocation cache at 256 MiB, so repeated
  variable-length prompts don't balloon RSS between files.

## semif-agent: the jev-ultrafast loop on SemIf

A port of browser-use's jev-ultrafast pattern with SemIf replacing the Jev
call. Every observation produces an indexed element table (visible controls
tagged on the live DOM); **one shared-mode pass per cycle** scores the
operation (CLICK/TYPE_TEXT/SELECT/WAIT/DONE/BLOCKED) plus speculative target
heads, each over only its compatible elements — zero generated tokens in the
decision path. The same local Qwen3.5-4B writes field text only when the
operation is TYPE_TEXT (strict JSON, one retry, then BLOCKED). DONE is never
trusted: an independent verifier (`--verify-text` regex on the page body)
decides, and a failed verification feeds back into the next state so the
agent can recover. Model output never becomes selectors or coordinates;
targets resolve from observed nodes via `data-semif-idx`.

Measured on the local flights demo (3/3 identical runs, greedy scoring is
deterministic): SELECT One way → type Zurich/London/date → click Search →
DONE+verified. 6 cycles, 3.5 s task time excluding the 5 s model load,
decide passes 143–208 ms/cycle. Wait discipline matters: only CLICK waits
for the page to change (≤1.5 s); typing settles 120 ms — typed values are
read by the next snapshot, not the body text. Limits: one fixture page, no
frames/shadow DOM/uploads, 16 options max per head, no SCROLL, single-task
evidence — not a reliability benchmark.

Wild-site runs are site-generic — no benchmark has code in the agent. The
same loop completed a three-stage task on a real SaaS portal end-to-end:
register a fresh account (the generator invents credentials with a
per-run id), auto-login, navigate to API keys, create a key, and verify the
secret on the page (21 cycles, 31.7 s; premature DONEs rejected twice by
the verifier along the way). The mechanisms that carried it, each measured
from a failure trace: action history in state and in the value generator,
password/number/tel/url inputs captured (masked), verified DONE and
two-strike BLOCKED, no-op and cycle detection by action signature
transient actions never kill a run.

## Safety when working with malicious samples

Every malicious corpus file must be defanged before scoring: no exec bits, all
endpoints rewritten to RFC 5737 TEST-NET (192.0.2.x) or `.invalid`/`.example`
domains, embedded payloads rewritten to harmless equivalents. Writing
realistic-looking samples is safe only if you audit every endpoint and payload.
