# AGENTS.md

## Cursor Cloud specific instructions

### Repository overview

This is the X "For You" Feed Recommendation Algorithm. It contains four components:

| Component | Language | Runnable locally |
|---|---|---|
| **Phoenix** (`phoenix/`) | Python (JAX/Haiku) | Yes |
| **Home Mixer** (`home-mixer/`) | Rust | No (depends on private `xai_*` crates) |
| **Thunder** (`thunder/`) | Rust | No (depends on private `xai_*` crates) |
| **Candidate Pipeline** (`candidate-pipeline/`) | Rust | No (shared library for home-mixer) |
| **Grox** (`grox/`) | Python | No (depends on internal packages) |

Only **Phoenix** is buildable and runnable locally. The Rust components have no `Cargo.toml` files and depend on private internal crates. Grox depends on internal Python packages not present in this repo.

### Phoenix development

- **Dependencies**: `cd phoenix && uv sync` (installs JAX, dm-haiku, numpy, pyright, pytest)
- **Lint**: `cd phoenix && ruff check .` (ruff must be installed via `uv tool install ruff` or globally)
- **Type check**: `cd phoenix && uv run pyright .` (pre-existing JAX/numpy type errors are expected; ~12 errors from array type mismatches)
- **Tests**: `cd phoenix && uv run pytest test_recsys_model.py test_recsys_retrieval_model.py` (34 tests)
- **Run pipeline**: `cd phoenix && uv run run_pipeline.py --artifacts_dir artifacts/oss-phoenix-artifacts`

### Model artifacts (Git LFS)

The pre-trained model checkpoint is stored as a ~2.8GB zip via Git LFS at `phoenix/artifacts/oss-phoenix-artifacts.zip`. Before running the pipeline:

1. `git lfs pull` (downloads the actual file; takes ~2 minutes)
2. `cd phoenix && unzip -o artifacts/oss-phoenix-artifacts.zip -d artifacts/`

The extracted directory contains retrieval/ranker model weights, embedding tables, a sports corpus (~84K posts), and an example user sequence.

### Gotchas

- The `run_pipeline.py` emits a `RuntimeWarning: overflow encountered in scalar multiply` during hash computation — this is expected and harmless.
- `pyright` reports ~12 type errors related to JAX/numpy array type annotations — these are pre-existing and do not affect runtime.
- `ruff` is configured in `pyproject.toml` but not listed as a project dependency; install it separately.
