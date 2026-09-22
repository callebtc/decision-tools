# decision-tools

Scanners, benchmarks, and web agents built on [laya-mlx](../laya-mlx) and
[Semif](../Semif). One self-contained uv project per sub dir:

```
laya-mlx/tools/   is-safe (scanner), laya-bench, laya-agent (+ demo html)
Semif/tools/      is-safe (scanner), semif-bench, semif-agent (+ demo html)
```

Each sub dir's own `README.md` has the detailed field notes.

## Prerequisites

- [uv](https://docs.astral.sh/uv/) — builds each sub dir's environment
  (`.venv/`, pinned by the committed `uv.lock`). A suitable Python (<3.14)
  is picked automatically.
- Sibling checkouts of the source projects, in the same parent directory
  as this repo: each sub dir's `pyproject.toml` installs them as editable
  sources, resolved relative to the sub dir (`../../laya-mlx` and
  `../../Semif`). If they live elsewhere, edit the `[tool.uv.sources]`
  path in the affected sub dir's `pyproject.toml`.
- Google Chrome — the web agents drive an installed Chrome via Playwright's
  `channel="chrome"` (no `playwright install` needed).
- Network on first run — model checkpoints download to the standard
  Hugging Face cache (`~/.cache/huggingface/`, outside any repo):
  421 MB for Laya, ~9 GB (Qwen3.5-4B) for SemIf.

## Usage

```bash
cd Semif                              # or: cd laya-mlx
uv sync                              # once: builds .venv/ from uv.lock

uv run tools/is-safe file.py         # scanner, in both sub dirs
uv run tools/laya-bench              # from laya-mlx/
uv run tools/semif-bench --iterations 20  # from Semif/
uv run tools/laya-agent --demo --headed   # from laya-mlx/; --headed watches Chrome
uv run tools/semif-agent --demo          # from Semif/; --url/--goal for any page
```

The tools also run directly — `./tools/<name> …` (the shebangs do the same).

Exit codes — `is-safe`: 0 safe / 1 caution / 2 unsafe / 3 error. Agents:
0 done+verified / 1 done+unverified / 2 blocked or stalled / 3 error. Pass
`--verify-text` for evidence-based success — without it DONE is never more
than the gate's opinion.

Notable flags:

- `is-safe`: `--model`, `--revision`, `--max-chars`, `--dump-signals`
- `laya-bench`: `--model`, `--dtype`, `--batch-size`
- `semif-bench`: `--model`, `--revision`, `--iterations`, `--warmup`, `--max-tokens`
- `laya-agent`: `--trace`, `--verify-text`, `--verify-report`, `--max-steps`,
  `--headed`, `--snapshot-only`, `--model`, `--no-refine`, and `--llm` — the
  text brain is one replaceable LLM command (default
  `opencode run -m openrouter/z-ai/glm-5.3`; `ollama run <model>` for fully local)
- `semif-agent`: `--trace`, `--verify-text`, `--max-steps`, `--headed`

Note: `laya-bench --help` crashes on a literal `%` in one help string
(`--compile`, needs `%%`); running the benchmark itself is unaffected.