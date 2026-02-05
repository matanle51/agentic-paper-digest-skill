---
name: agentic-paper-digest-skill
description: Fetches and summarizes recent arXiv and Hugging Face papers with Agentic Paper Digest. Use when the user wants a paper digest, a JSON feed of recent papers, or to run the arXiv/HF pipeline.
homepage: https://github.com/matanle51/agentic_paper_digest
compatibility: Requires Python 3, network access, and either git or curl/wget for bootstrap. LLM access via OPENAI_API_KEY or LITELLM_API_KEY (OpenAI-compatible).
metadata: {"clawdbot":{"requires":{"anyBins":["python3","python"]}}}
---

# Agentic Paper Digest

## When to use
- Fetch a recent paper digest from arXiv and Hugging Face.
- Produce JSON output for downstream agents.
- Run a local API server when a polling workflow is needed.

## Prereqs
- Python 3 and network access.
- LLM access via `OPENAI_API_KEY` or an OpenAI-compatible provider via `LITELLM_API_BASE` + `LITELLM_API_KEY`.
- `git` is optional for bootstrap; otherwise `curl`/`wget` (or Python) is used to download the repo.

## Mandatory workflow (follow step-by-step)
1. **Clarify intent and scope**: Ask the user for:
   - Topics of interest (or confirm defaults).
   - Time window (hours) and timezone if relevant.
   - Output mode: CLI JSON vs API server.
   - Model/provider preference (OpenAI vs compatible proxy).
2. **Confirm workspace path**: Ask where to clone/run. Default to `PROJECT_DIR="$HOME/agentic_paper_digest"` if the user doesn’t care. Never hardcode `/Users/...` paths.
3. **Bootstrap the repo**: Run the bootstrap script (unless the repo already exists and the user says to skip).
4. **Create or verify `.env`**:
   - If `.env` is missing, create it from `.env.example` (in the repo), then ask the user to fill keys and preferences.
   - Ensure at least one of `OPENAI_API_KEY` or `LITELLM_API_KEY` is set before running.
5. **Collect or update topics**:
   - Ask for the topics of interest if not provided.
   - Update `config/topics.json` to match those topics (or use the API endpoints if running the server).
6. **Run the pipeline**:
   - Prefer `scripts/run_cli.sh` for one-off JSON output.
   - Use `scripts/run_api.sh` only if polling or UI/API access is needed.
7. **Report results**:
   - Summarize run stats (`seen`, `kept`, window).
   - If results are sparse, suggest increasing `WINDOW_HOURS`, `ARXIV_MAX_RESULTS`, or broadening topics.

## Get the code and install
- Preferred: run the bootstrap helper script. It uses git when available or falls back to a zip download.

```bash
bash "{baseDir}/scripts/bootstrap.sh"
```

- Override the clone location by setting `PROJECT_DIR`.

```bash
PROJECT_DIR="$HOME/agentic_paper_digest" bash "{baseDir}/scripts/bootstrap.sh"
```

## Run (CLI preferred)

```bash
bash "{baseDir}/scripts/run_cli.sh"
```

- Pass through CLI flags as needed.

```bash
bash "{baseDir}/scripts/run_cli.sh" --window-hours 24 --sources arxiv,hf
```

## Run (API optional)

```bash
bash "{baseDir}/scripts/run_api.sh"
```

- Trigger runs and read results.

```bash
curl -X POST http://127.0.0.1:8000/api/run
curl http://127.0.0.1:8000/api/status
curl http://127.0.0.1:8000/api/papers
```

- Stop the API server if needed.

```bash
bash "{baseDir}/scripts/stop_api.sh"
```

## Outputs
- CLI `--json` prints `run_id`, `seen`, `kept`, `window_start`, and `window_end`.
- Data store: `data/papers.sqlite3` (under `PROJECT_DIR`).
- API: `POST /api/run`, `GET /api/status`, `GET /api/papers`, `GET/POST /api/topics`, `GET/POST /api/settings`.

## Configuration
Config files live in `PROJECT_DIR/config`. Environment variables can be set in the shell or via a `.env` file. The wrappers here auto-load `.env` from `PROJECT_DIR` (override with `ENV_FILE=/path/to/.env`).

**Environment (.env or exported vars)**
- `OPENAI_API_KEY`: required for OpenAI models (litellm reads this).
- `LITELLM_API_BASE`, `LITELLM_API_KEY`: use an OpenAI-compatible proxy/provider.
- `LITELLM_MODEL_RELEVANCE`, `LITELLM_MODEL_SUMMARY`: models for relevance and summarization.
- `LITELLM_TEMPERATURE_RELEVANCE`, `LITELLM_TEMPERATURE_SUMMARY`: lower for more deterministic output.
- `WINDOW_HOURS`, `APP_TZ`: recency window and timezone.
- `ARXIV_CATEGORIES`: comma-separated categories (default includes `cs.CL,cs.AI,cs.LG,stat.ML,cs.CR`).
- `ARXIV_MAX_RESULTS`, `ARXIV_PAGE_SIZE`, `FETCH_TIMEOUT_S`, `MAX_CANDIDATES_PER_SOURCE`: fetch limits and timeouts.
- `ENABLE_PDF_TEXT=1`: include first-page PDF text in summaries; requires `PyMuPDF` (`pip install pymupdf`).
- Path overrides: `TOPICS_PATH`, `SETTINGS_PATH`, `AFFILIATION_BOOSTS_PATH`, `DATA_DIR`.

**Config files**
- `config/topics.json`: list of topics with `id`, `label`, `description`, `max_per_topic`, and `keywords`. The relevance classifier must output topic IDs exactly as defined here. `max_per_topic` is also used to cap results in `GET /api/papers` when `apply_topic_caps=1`.
- `config/settings.json`: overrides default fetch limits (`arxiv_max_results`, `arxiv_page_size`, `fetch_timeout_s`, `max_candidates_per_source`). Updated via `POST /api/settings`.
- `config/affiliations.json`: list of `{pattern, weight}` boosts applied by substring match over affiliations. Weights add up and are capped at 1.0. Invalid JSON disables boosts, so keep the file strict JSON (no trailing commas).

## Getting good results
- Keep topics focused and mutually exclusive so the classifier can choose the right IDs.
- Use a stronger model for summaries than for relevance if quality matters.
- Increase `WINDOW_HOURS` or `ARXIV_MAX_RESULTS` when results are sparse, or lower them if results are too noisy.
- Tune `ARXIV_CATEGORIES` to your research domains.
- Enable PDF text (`ENABLE_PDF_TEXT=1`) when abstracts are too thin.
- Use modest affiliation weights to bias ranking without swamping relevance.

## Troubleshooting
- Port 8000 busy: run `bash "{baseDir}/scripts/stop_api.sh"` or pass `--port` to the API command.
- Empty results: increase `WINDOW_HOURS` or verify the API key in `.env`.
- Missing API key errors: export `OPENAI_API_KEY` or `LITELLM_API_KEY` in the shell before running.
