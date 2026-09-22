# decision-tools

A collection of tools for building on [laya-mlx](laya-mlx/vendor) and
[SemIf](Semif/vendor). Both source projects are vendored as git submodules
(`laya-mlx/vendor`, `Semif/vendor`), pinned to the revisions the tools were
tuned against and installed as editable sources by each sub dir's
`pyproject.toml`.

## Prerequisites

- [uv](https://docs.astral.sh/uv/) — picks a suitable Python (<3.14) itself.
- Google Chrome — the agents drive an installed Chrome via Playwright's
  `channel="chrome"` (no `playwright install` needed).
- Network on first run — model checkpoints download to the Hugging Face cache
  (`~/.cache/huggingface/`, outside the repo): 421 MB Laya, ~9 GB SemIf.

## Setup from a fresh clone

A plain `git clone` leaves the `vendor/` dirs empty — `uv sync` fails until
the submodules are checked out:

```bash
git clone https://github.com/callebtc/decision-tools
cd decision-tools
git submodule update --init          # checks out laya-mlx/vendor and Semif/vendor
cd laya-mlx                          # or: cd Semif
uv sync                              # once per sub dir: builds .venv/ from uv.lock
```

`git clone --recurse-submodules …` does both clone steps in one.

## Usage

```bash
uv run tools/is-safe file.py              # scanner, in both sub dirs
uv run tools/laya-bench                   # from laya-mlx/
uv run tools/semif-bench --iterations 20  # from Semif/
uv run tools/laya-agent --demo --headed    # from laya-mlx/; --headed watches Chrome
uv run tools/semif-agent --demo           # from Semif/; --url/--goal for any page
```

The tools also run directly — `./tools/<name> …` (the shebangs do the same).

Details (exit codes, flags, field notes) are in each sub dir's `tools/README.md`.