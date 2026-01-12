# Repository Guidelines

## Project Structure & Module Organization
- `src/` contains the application code (`chat_engine`, `service`, `engine_utils`) and entrypoint `src/demo.py`.
- `src/handlers/` hosts modular components (ASR/LLM/TTS/avatar/VAD) and related runtime assets.
- `config/` holds runnable YAML presets such as `config/chat_with_lam.yaml`.
- `tests/` is split into `tests/unittest/` and `tests/inttest/`.
- `scripts/` includes install and model download helpers (for example `scripts/post_config_install.sh`).
- `assets/`, `docs/`, `ssl_certs/`, and `coturn-data/` store static media, documentation, and RTC support files.

## Build, Test, and Development Commands
- `uv venv --python 3.11.11`: create a local Python 3.11 virtual environment.
- `uv sync --all-packages`: install all dependencies defined in `pyproject.toml`.
- `uv run install.py --uv --config config/<preset>.yaml`: install only the handler deps required by a config.
- `./scripts/pre_config_install.sh --config config/<preset>.yaml` and `./scripts/post_config_install.sh --config config/<preset>.yaml`: pre/post steps for CUDA libs.
- `uv run src/demo.py --config config/<preset>.yaml`: run the app locally.
- `./build_and_run.sh --config config/<preset>.yaml`: build and start the Docker image in one step.
- `./build_cuda128.sh` and `./run_docker_cuda128.sh --config config/<preset>.yaml`: CUDA 12.8 image build/run.
- `docker compose up` / `docker compose down`: start/stop the service set in `docker-compose.yml`.

## Coding Style & Naming Conventions
- Python 3.11; 4-space indentation; keep lines ≤120 chars (see `setup.cfg` flake8 config).
- Use `snake_case` for modules/functions, `PascalCase` for classes; follow existing handler naming under `src/handlers/`.
- Config files follow `config/chat_with_*.yaml` naming.

## Testing Guidelines
- Tests use the standard `unittest` framework.
- Run unit tests: `python -m unittest discover -s tests/unittest`.
- Run integration tests: `python -m unittest discover -s tests/inttest`.
- Test files are named `test_*.py` and test classes start with `Test`.

## Commit & Pull Request Guidelines
- Git history is minimal (single commit “1”), so no formal convention is established. Use short, imperative commit summaries (optionally with a scope).
- PRs should describe the config used (e.g., `config/chat_with_openai_compatible.yaml`), hardware/OS where relevant, and any tests run. Include screenshots or short clips for UI/UX changes.

## Configuration & Model Assets
- Configuration lives in `config/`; keep API keys and secrets out of version control.
- Large models are expected to be downloaded via `scripts/` (e.g., `download_*` scripts). Use Git LFS only where required by upstream assets.
