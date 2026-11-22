# Agent Instructions

This repository is a **personal fork** of NautilusTrader. It is not contributed upstream, so ignore
`CONTRIBUTING.md`, `AI_POLICY.md`, pull request templates, and any guidance about becoming a
contributor or opening a pull request. Follow the
[coding standards](docs/developer_guide/coding_standards.md) for the area you change.

## Working rules

**NautilusTrader can execute live trades involving real capital. Hold every change to a very high
standard for correctness, reliability, testing, and clarity.**

### Development environment

Build and test inside the dev container; the host only needs Docker. The source tree is bind-mounted
at `/workspace` and a non-root `dev` user matches the host UID/GID, so files written by the
container are owned by you on the host.

```bash
cd /mnt/storage/nautilus_trader

# Only needed on first setup, or when rust-toolchain.toml / Cargo.toml / Cargo.lock change.
docker build -f .docker/DockerfileUbuntu -t nautilus-dev-base .

docker compose -f docker-compose.dev.yml up -d
docker compose -f docker-compose.dev.yml exec dev bash
```

Inside the container:

```bash
make sync         # first run, or after python/pyproject.toml or python/uv.lock changes
make build-debug  # after Rust or Python changes (incremental, preferred)
make build        # release build
make py-stubs     # after PyO3 binding or stub changes
make pytest       # Python tests
```

Rules that keep this setup working:

- Use the uv project environment at `python/.venv`. Do not create a root `.venv`.
- Never run `uv run --project python ...` without `--no-sync`. uv treats the editable install as
  stale and starts a full maturin rebuild; all `make` targets already pass `--no-sync`. To run the
  interpreter directly, use `python/.venv/bin/python`.
- `python/.venv` is container-specific: its interpreter symlink points at `/opt/uv-python/...` in
  the image, so it does not work from the host. Recreate it (`rm -rf python/.venv && make sync`) if
  the base image's Python version changes.
- `make cargo-test` needs `PYTHONHOME` exported (the linker path is already baked into the image):
  `export PYTHONHOME="$(python/.venv/bin/python -c 'import sys; print(sys.base_prefix)')"`
- `.docker/dev.dockerfile` and `docker-compose.dev.yml` are deliberately untracked (listed in
  `.git/info/exclude`). Do not delete them, `git add` them, or move that exclusion into
  `.gitignore`.
- The image installs `jq` for `scripts/check-cargo-cooldown.sh`, which every `make build*` target
  runs. If a target reports a missing tool, add it to `.docker/dev.dockerfile` and rebuild the
  image rather than installing it on the host.
- `CARGO_BUILD_JOBS` defaults to 6 in the compose file (8 cores / 13 GB). Lower it if `rustc` gets
  OOM-killed.
- `target/` is roughly 12 GB and the Docker build cache around 16 GB; `docker builder prune`
  reclaims the cache.

### Scope and style

- Read the affected code and search for existing patterns before proposing or making changes.
- Keep each change focused on the requested outcome. Note unrelated issues instead of fixing them,
  and avoid drive-by refactors, renames, and abstractions.
- Match the existing style and use established functions, types, names, and dependencies.

### Correctness

- Preserve exact arithmetic for prices, quantities, money, fees, and other discrete values. Use the
  project domain types or `Decimal`.
- Normalize path separators with the shared helpers (for example `normalize_path_separators` in
  `nautilus_persistence::common::paths`) before splitting or joining; never split a raw path on
  `/` alone.

### Tests

- Do not add test-only behavior, branches, attributes, or interfaces to production code.
- Do not weaken, remove, bypass, or rewrite tests merely to obtain a passing result. Fix the
  underlying problem and preserve the behavior the tests protect. Change a test only when the task
  intentionally changes the required behavior or when you can independently verify it is wrong.
- Do not let tests depend on adapter environment variables; `make pre-flight` runs with all of them
  unset via `scripts/strip-adapter-env.bash`. Register new adapter environment variables there.
- Before calling a change done, run `make format` and the tests relevant to the change.
  `make pre-commit` runs the full lint suite.

### Change mechanics

- Change generated artifacts through their source and generator. Never edit them by hand.
- Do not modify `RELEASES.md` unless asked.

### Git

- Do not commit, amend, push, or change remote state unless the user explicitly asks.
- Do not interact with GitHub issues or pull requests, on this fork or upstream, unless the user
  explicitly asks.
