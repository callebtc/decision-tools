# decision-tools

Central collection of the tools built on [laya-mlx](../laya-mlx) and
[Semif](../Semif). They used to live in each project's `tools/` directory;
this repo is now their home. The originals' field notes travel with them:
each directory keeps its own `README.md`.

```
laya-mlx/tools/   is-safe, laya-bench, laya-agent (+ demo html)
Semif/tools/      is-safe, semif-bench, semif-agent (+ demo html)
```

`laya-mlx/tools/browser-use/` (the community `laya-ultrafast` clone) is not
copied here — it has its own git history and stays in the laya-mlx repo.

## Prerequisites

- [uv](https://docs.astral.sh/uv/) — each sub dir is a self-contained uv
  project: `uv sync` inside it installs the environment (`.venv/`, pinned by
  the committed `uv.lock`). uv also picks a suitable Python (<3.14) per
  project automatically.
- Sibling checkouts of the source projects, in the same parent directory as
  this repo: each sub dir's `pyproject.toml` installs the local packages as
  editable sources, resolved **relative to the sub dir** (`../../laya-mlx`
  and `../../Semif`). If the checkouts live elsewhere, edit the
  `[tool.uv.sources]` path in the affected sub dir's `pyproject.toml`.
- Google Chrome for the web agents (both drive an installed Chrome via
  Playwright's `channel="chrome"` — no `playwright install` needed).
- Network on first run: each sub dir's env is built once by `uv sync`, and
  the model checkpoints download to the standard Hugging Face cache
  (`~/.cache/huggingface/`, outside any repo) — 421 MB for Laya, ~9 GB
  (Qwen3.5-4B) for SemIf.

## Running

Each sub dir is a uv project: `cd` into it, `uv sync` once, then run any tool
with `uv run tools/<name> …` or directly `./tools/<name> …` (the shebangs do
the same):

```bash
cd Semif                             # or: cd laya-mlx
uv sync                              # once: builds .venv/ from uv.lock
uv run tools/is-safe file.py         # scanner, in both sub dirs
```

```bash
uv run tools/laya-bench                    # from laya-mlx/
uv run tools/semif-bench --iterations 20  # from Semif/
uv run tools/laya-agent --demo --headed    # from laya-mlx/; --headed watches Chrome
uv run tools/semif-agent --demo           # from Semif/; --url/--goal for any page
```

Scanner exit codes: 0 safe / 1 caution / 2 unsafe / 3 error. Agent exit codes:
0 done+verified / 1 done+unverified / 2 blocked or stalled / 3 error; pass
`--verify-text` for evidence-based success — without it DONE is never more
than the gate's opinion. `laya-agent`'s text brain is one replaceable LLM
command (`--llm`, default `opencode run -m openrouter/z-ai/glm-5.3`;
`ollama run <model>` for fully local).

Note: `laya-bench --help` currently crashes on a literal `%` in one help
string (`--compile`, needs `%%`); running the benchmark itself is unaffected.

## Dependencies, per sub dir

Each sub dir's `pyproject.toml` pins the union of what its tools need; both
source packages install editable from the sibling checkout, so the tools
always run against the source sitting next to this repo.

| Sub dir      | Package source                                | Also pulls              |
| ------------ | --------------------------------------------- | ----------------------- |
| `laya-mlx/`  | `laya-mlx` (editable `../../laya-mlx`)        | `rich`, `playwright`    |
| `Semif/`     | `semif-phase1[mlx]` (editable `../../Semif`) | `rich`, `playwright`    |

## Further reading

Each directory's `README.md` is the dense field-notes summary (programming
model, measured numbers, what was learned the hard way) — read it before
changing prompts or thresholds:

- [`laya-mlx/tools/README.md`](laya-mlx/tools/README.md)
- [`Semif/tools/README.md`](Semif/tools/README.md)
