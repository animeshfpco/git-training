# Plan A — App Readiness for EC2 Deployment

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prepare the research-rag application for deployment to AWS EC2 by adding the missing `/health` endpoint, building a minimal Streamlit UI, registering the `live` pytest marker, authoring all deploy-side container and systemd artifacts under `deploy/`, and adding a RAGAS regression-gate script. Outcome: `docker compose up -d` under Colima on the workstation brings up `caddy + api + ui` against the existing Qdrant Cloud cluster, and the gate script can compare any RAGAS run to the v3.1.1 baseline.

**Architecture:** Three Python surface changes (`/health`, `src/ui/app.py`, `scripts/ragas_gate.py`); one pytest config change (live marker); and a new top-level `deploy/` directory containing the Caddy + compose + Dockerfiles + shell helpers + systemd units defined in the design spec.

**Tech Stack:** Python 3.12, FastAPI, Streamlit, pytest, Docker + docker compose under Colima, Caddy v2, systemd, bash, uv.

**Reference spec:** [`docs/superpowers/specs/2026-05-21-ec2-free-tier-deployment-design.md`](../specs/2026-05-21-ec2-free-tier-deployment-design.md). This plan implements its app-side scope (Sections 4 + `/health` open item + live marker from Section 2).

**Out of scope (covered by later plans):** AWS account/IAM/OIDC/ECR/SSM/EC2 provisioning (Plan B), GitHub Actions workflows + first end-to-end deploy (Plan C).

**Pre-requisites for executing this plan:**
- Colima installed and reachable: `colima start` succeeds, `docker info` works.
- Existing `.env` at the repo root with valid `GOOGLE_API_KEY`, `JINA_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` (matches `.env.template`).
- `uv` installed and `uv sync` completed.

---

## File map

| Path | Status | Responsibility |
|---|---|---|
| `src/api/main.py` | modify | Add `GET /health` returning `{status, pipeline_version}`. No other changes. |
| `tests/integration/test_api.py` | modify | Add tests for `/health`. |
| `src/ui/__init__.py` | create | Marker for the new `ui` package. |
| `src/ui/app.py` | create | Streamlit single-page chat-with-papers UI. Calls FastAPI over the env-configured base URL. |
| `tests/unit/test_ui.py` | create | Pure-function tests for the UI's request/response helpers (no Streamlit runtime needed). |
| `pyproject.toml` | modify | Register `live` pytest marker; pin Streamlit + httpx (UI deps). |
| `scripts/ragas_gate.py` | create | Compare a new RAGAS result JSON against the v3.1.1 baseline; exit non-zero if composite drops more than 0.02. |
| `tests/unit/test_ragas_gate.py` | create | Unit tests for the gate's pass/fail logic against fixture JSONs. |
| `deploy/Dockerfile.api` | create | Multi-stage uv-based image for FastAPI. |
| `deploy/Dockerfile.ui` | create | Multi-stage uv-based image for Streamlit. |
| `deploy/.dockerignore` | create | Keep image lean (exclude `.venv`, `data/`, `evaluation/`, `notebooks/`, `tests/`, etc.). |
| `deploy/docker-compose.yml` | create | 3-service stack: caddy, api, ui. |
| `deploy/Caddyfile` | create | Reverse proxy + auto-TLS routing. |
| `deploy/fetch-secrets.sh` | create | Pull SSM params into `/opt/research-rag/.env`. (Executable; no smoke test in this plan — depends on Plan B.) |
| `deploy/duckdns-update.sh` | create | Update DuckDNS record with current public IP. (Executable; no smoke test in this plan — depends on Plan B.) |
| `deploy/systemd/research-rag.service` | create | systemd unit running `docker compose up -d`. |
| `deploy/systemd/research-rag-secrets.service` | create | Pre-compose one-shot running fetch-secrets.sh. |
| `deploy/systemd/duckdns-update.service` | create | One-shot calling duckdns-update.sh. |
| `deploy/systemd/duckdns-update.timer` | create | Boot + 30-min cadence for the above. |
| `deploy/README.md` | create | One-page operator README: how to bring up the stack locally, where each artifact is used, what each env var means. |

**Out of this plan's file map** (Plan B/C territory): `.github/workflows/*.yml`, anything under `evaluation/results/`, ingestion files.

---

## Conventions used in this plan

- **TDD where it fits.** `/health`, the Streamlit helper functions, and the RAGAS gate are all unit-testable — those tasks follow red→green. Infra files (Dockerfiles, Caddyfile, compose, systemd) get a write-then-verify pattern (lint/parse the file, then a smoke check where possible).
- **Do NOT commit during this plan.** Leave the working tree dirty after each task. The user reviews all changes manually and decides commit boundaries themselves. Each task ends after verification — no `git add`, no `git commit`.
- **Colima quirk.** Where a step needs Docker, prefix it with a `colima status` check. If it returns `Stopped`, run `colima start` first.
- **Cross-arch builds.** When building images locally for EC2 t2.micro (x86_64), use `--platform linux/amd64` on every `docker buildx build` invocation. Plan A only builds locally for smoke testing; Plan C pushes to ECR.
- **Python style.** Match the existing codebase: ruff defaults, no docstrings unless the behavior is non-obvious, no emojis.

---

## Task 1: Add `/health` endpoint with pipeline version

**Files:**
- Modify: [src/api/main.py](../../../src/api/main.py)
- Test: [tests/integration/test_api.py](../../../tests/integration/test_api.py)

**Context:** The Caddyfile + docker compose healthcheck both expect `GET /health` to return 200 with the current pipeline version. Configured `pipeline_version` already lives in `settings.pipeline_version` ([src/config/config.py:97](../../../src/config/config.py#L97)).

- [ ] **Step 1: Write the failing tests**

Append the following class to `tests/integration/test_api.py` (after `TestQueryEndpoint`):

```python
class TestHealthEndpoint:
    def test_returns_200(self, client: TestClient) -> None:
        response = client.get("/health")
        assert response.status_code == 200

    def test_payload_has_status_ok(self, client: TestClient) -> None:
        response = client.get("/health")
        assert response.json()["status"] == "ok"

    def test_payload_includes_pipeline_version(self, client: TestClient) -> None:
        from src.config.config import settings

        response = client.get("/health")
        assert response.json()["pipeline_version"] == settings.pipeline_version
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/integration/test_api.py::TestHealthEndpoint -v`

Expected: 3 failures, all returning 404 (endpoint does not exist yet).

- [ ] **Step 3: Implement the endpoint**

In `src/api/main.py`, add a response model and route. Place the model right above `QueryResponse`:

```python
class HealthResponse(BaseModel):
    status: str
    pipeline_version: str
```

Place the route at the bottom of the file (after the `/query` route):

```python
@app.get("/health", response_model=HealthResponse)
def health() -> HealthResponse:
    return HealthResponse(status="ok", pipeline_version=settings.pipeline_version)
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/integration/test_api.py -v`

Expected: all `TestHealthEndpoint` tests pass plus all pre-existing `TestQueryEndpoint` tests stay green.

- [ ] **Step 5: Lint**

Run: `uv run ruff check . && uv run ruff format --check .`

Expected: no errors.

---

## Task 2: Register the `live` pytest marker

**Files:**
- Modify: [pyproject.toml](../../../pyproject.toml)

**Context:** Plan C (CI) runs `pytest -m 'not live'` to skip live-API tests in CI. We need the marker registered so pytest doesn't warn about it, and so future tests that hit real Gemini/Jina/Qdrant can opt in. No existing tests are live (`tests/conftest.py` ships dummy env vars and all retriever/chain tests are mocked).

- [ ] **Step 1: Add the marker to `[tool.pytest.ini_options]`**

Edit `pyproject.toml`. Find the existing block:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
```

Replace it with:

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["."]
markers = [
    "live: hits live external APIs (Gemini / Jina / Qdrant); excluded by default in CI",
]
```

- [ ] **Step 2: Verify pytest recognizes the marker**

Run: `uv run pytest --markers | grep -F 'live:'`

Expected output contains: `@pytest.mark.live: hits live external APIs ...`

- [ ] **Step 3: Verify excluding live tests still runs everything (since none are live yet)**

Run: `uv run pytest -m 'not live' --collect-only -q | tail -1`

Expected: same test count as `uv run pytest --collect-only -q | tail -1`.

---

## Task 3: Streamlit UI helpers (pure functions, unit-tested)

**Files:**
- Create: `src/ui/__init__.py`
- Create: `src/ui/api_client.py`
- Test: `tests/unit/test_ui.py`

**Context:** Streamlit pages are awkward to unit-test, so we extract the HTTP-shaped logic into a small pure module and test that. The Streamlit page itself in Task 4 only wires inputs/outputs around these helpers.

- [ ] **Step 1: Write the failing tests**

Create `tests/unit/test_ui.py`:

```python
from unittest.mock import MagicMock, patch

import pytest

from src.ui.api_client import APIClient, QueryResult


def test_api_client_uses_configured_base_url() -> None:
    client = APIClient(base_url="http://api:8000")
    assert client.base_url == "http://api:8000"


def test_api_client_strips_trailing_slash_from_base_url() -> None:
    client = APIClient(base_url="http://api:8000/")
    assert client.base_url == "http://api:8000"


def test_query_returns_parsed_result() -> None:
    client = APIClient(base_url="http://api:8000")
    fake_response = MagicMock(status_code=200)
    fake_response.json.return_value = {
        "answer": "Transformers use self-attention.",
        "sources": [
            {
                "title": "Attention Is All You Need",
                "authors": ["Vaswani"],
                "paper_id": "1706.03762",
                "chunk_index": 0,
                "score": 0.92,
            }
        ],
    }
    with patch("src.ui.api_client.requests.post", return_value=fake_response) as post:
        result = client.query("What is attention?")
    post.assert_called_once_with(
        "http://api:8000/query",
        json={"question": "What is attention?"},
        timeout=60,
    )
    assert isinstance(result, QueryResult)
    assert result.answer == "Transformers use self-attention."
    assert len(result.sources) == 1
    assert result.sources[0]["paper_id"] == "1706.03762"


def test_query_raises_on_http_error() -> None:
    client = APIClient(base_url="http://api:8000")
    fake_response = MagicMock(status_code=503)
    fake_response.raise_for_status.side_effect = RuntimeError("Service Unavailable")
    fake_response.json.return_value = {"detail": "boom"}
    with patch("src.ui.api_client.requests.post", return_value=fake_response):
        with pytest.raises(RuntimeError):
            client.query("anything")


def test_health_returns_payload() -> None:
    client = APIClient(base_url="http://api:8000")
    fake_response = MagicMock(status_code=200)
    fake_response.json.return_value = {"status": "ok", "pipeline_version": "v3.1.1"}
    with patch("src.ui.api_client.requests.get", return_value=fake_response):
        payload = client.health()
    assert payload == {"status": "ok", "pipeline_version": "v3.1.1"}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/unit/test_ui.py -v`

Expected: all 5 tests fail with `ModuleNotFoundError: src.ui.api_client`.

- [ ] **Step 3: Create the package marker**

Create `src/ui/__init__.py` (empty file):

```python
```

- [ ] **Step 4: Implement the API client**

Create `src/ui/api_client.py`:

```python
from dataclasses import dataclass
from typing import Any

import requests


@dataclass
class QueryResult:
    answer: str
    sources: list[dict[str, Any]]


class APIClient:
    def __init__(self, base_url: str, timeout: int = 60) -> None:
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout

    def query(self, question: str) -> QueryResult:
        response = requests.post(
            f"{self.base_url}/query",
            json={"question": question},
            timeout=self.timeout,
        )
        response.raise_for_status()
        payload = response.json()
        return QueryResult(answer=payload["answer"], sources=payload["sources"])

    def health(self) -> dict[str, Any]:
        response = requests.get(f"{self.base_url}/health", timeout=self.timeout)
        response.raise_for_status()
        return response.json()
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `uv run pytest tests/unit/test_ui.py -v`

Expected: all 5 tests pass.

- [ ] **Step 6: Lint**

Run: `uv run ruff check . && uv run ruff format --check .`

Expected: no errors.

---

## Task 4: Streamlit page wiring

**Files:**
- Create: `src/ui/app.py`
- Modify: [pyproject.toml](../../../pyproject.toml) (add `streamlit` runtime dep)

**Context:** Single-page UI: text input → calls APIClient → renders answer + sources. Sidebar shows pipeline_version (from `/health`) + a link to the repo. Base URL comes from the `API_BASE_URL` env var; defaults to `http://localhost:8000` for local dev.

- [ ] **Step 1: Add Streamlit to runtime dependencies**

Edit `pyproject.toml`. Find the `dependencies = [...]` block and add `"streamlit>=1.40"` to the list:

```toml
dependencies = [
    "arxiv>=2.4.1",
    "fastapi>=0.135.1",
    "google-genai>=1.66.0",
    "langchain-text-splitters>=1.1.1",
    "langsmith>=0.7.14",
    "pydantic-settings>=2.13.1",
    "pymupdf>=1.27.1",
    "qdrant-client>=1.17.0",
    "uvicorn>=0.41.0",
    "requests>=2.0",
    "tenacity>=9.0.0",
    "tiktoken>=0.12.0",
    "jsonref>=1.1.0",
    "ragas>=0.4.3",
    "huggingface-hub>=1.6.0",
    "streamlit>=1.40",
]
```

- [ ] **Step 2: Sync deps**

Run: `uv sync`

Expected: streamlit downloads and installs.

- [ ] **Step 3: Create the Streamlit page**

Create `src/ui/app.py`:

```python
import os

import streamlit as st

from src.ui.api_client import APIClient

API_BASE_URL = os.environ.get("API_BASE_URL", "http://localhost:8000")
REPO_URL = os.environ.get("REPO_URL", "https://github.com/")


@st.cache_resource
def get_client() -> APIClient:
    return APIClient(base_url=API_BASE_URL)


def render_sources(sources: list[dict]) -> None:
    if not sources:
        st.info("No sources returned.")
        return
    for i, source in enumerate(sources, start=1):
        with st.expander(f"[{i}] {source['title']} — score {source['score']:.3f}"):
            authors = source.get("authors") or []
            st.write(f"**Authors:** {', '.join(authors) if authors else 'Unknown'}")
            st.write(f"**ArXiv ID:** `{source['paper_id']}`")
            st.write(f"**Chunk index:** {source['chunk_index']}")


def render_sidebar(client: APIClient) -> None:
    st.sidebar.title("research-rag")
    try:
        health = client.health()
        st.sidebar.success(f"API healthy — pipeline `{health['pipeline_version']}`")
    except Exception as e:
        st.sidebar.error(f"API unreachable: {e}")
    st.sidebar.markdown(f"[View source on GitHub]({REPO_URL})")


def main() -> None:
    st.set_page_config(page_title="research-rag", layout="wide")
    client = get_client()
    render_sidebar(client)

    st.title("Ask a question about ArXiv ML papers")
    question = st.text_area("Question", height=100, placeholder="What does LoRA optimize?")
    submit = st.button("Ask", type="primary", disabled=not question.strip())

    if submit:
        with st.spinner("Retrieving and generating..."):
            try:
                result = client.query(question.strip())
            except Exception as e:
                st.error(f"Query failed: {e}")
                return
        st.subheader("Answer")
        st.markdown(result.answer)
        st.subheader("Sources")
        render_sources(result.sources)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Smoke-test the Streamlit app boots**

Open a second terminal. Start FastAPI:

```bash
uv run uvicorn src.api.main:app --port 8000
```

In the first terminal, start Streamlit:

```bash
API_BASE_URL=http://localhost:8000 uv run streamlit run src/ui/app.py --server.headless true --server.port 8501
```

Expected: terminal shows `You can now view your Streamlit app in your browser. URL: http://0.0.0.0:8501`. Open `http://localhost:8501` in a browser — sidebar shows "API healthy — pipeline v3.1.1-prefetch-scale-30" (or whatever `settings.pipeline_version` currently is).

Stop both processes with Ctrl-C when verified.

- [ ] **Step 5: Lint**

Run: `uv run ruff check . && uv run ruff format --check .`

Expected: no errors.

---

## Task 5: RAGAS regression gate script

**Files:**
- Create: `scripts/ragas_gate.py`
- Test: `tests/unit/test_ragas_gate.py`

**Context:** Plan C's `deploy.yml` calls this script as the regression gate. It reads the locked baseline JSON and a freshly-produced result JSON, computes composite scores, and exits non-zero if `composite_new < composite_baseline - tolerance`. Baseline is `evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json`; tolerance is `0.02` (per design Section 2). No per-metric guard.

Sample baseline shape (already on disk):

```json
{
  "experiment": "v3.1.1-prefetch-scale-30",
  "pipeline_version": "v3.1.1-prefetch-scale-30",
  "aggregate": {
    "faithfulness": 0.9844,
    "answer_relevancy": 0.897,
    "context_precision": 0.8328,
    "context_recall": 0.9106
  }
}
```

Composite = arithmetic mean of the four metrics (consistent with how the iteration history in the memory file recorded composites — e.g. 0.9062 for v3.1.1).

- [ ] **Step 1: Write the failing tests**

Create `tests/unit/test_ragas_gate.py`:

```python
import json
from pathlib import Path

import pytest

from scripts.ragas_gate import GateResult, composite_score, evaluate_gate, load_result


BASELINE = {
    "experiment": "v3.1.1-prefetch-scale-30",
    "pipeline_version": "v3.1.1-prefetch-scale-30",
    "aggregate": {
        "faithfulness": 0.9844,
        "answer_relevancy": 0.897,
        "context_precision": 0.8328,
        "context_recall": 0.9106,
    },
}


def _write(tmp_path: Path, name: str, payload: dict) -> Path:
    p = tmp_path / name
    p.write_text(json.dumps(payload))
    return p


def test_composite_score_is_mean_of_four_metrics() -> None:
    composite = composite_score(BASELINE["aggregate"])
    assert composite == pytest.approx(0.9062, abs=0.0005)


def test_load_result_parses_aggregate(tmp_path: Path) -> None:
    path = _write(tmp_path, "baseline.json", BASELINE)
    loaded = load_result(path)
    assert loaded["aggregate"]["faithfulness"] == 0.9844


def test_gate_passes_when_new_composite_equals_baseline(tmp_path: Path) -> None:
    baseline_path = _write(tmp_path, "baseline.json", BASELINE)
    new_path = _write(tmp_path, "new.json", BASELINE)
    result = evaluate_gate(baseline_path, new_path, tolerance=0.02)
    assert result.passed is True
    assert result.delta == pytest.approx(0.0, abs=1e-6)


def test_gate_passes_when_new_composite_within_tolerance(tmp_path: Path) -> None:
    new = json.loads(json.dumps(BASELINE))
    new["aggregate"]["faithfulness"] -= 0.05  # composite drops by ~0.0125
    new_path = _write(tmp_path, "new.json", new)
    baseline_path = _write(tmp_path, "baseline.json", BASELINE)
    result = evaluate_gate(baseline_path, new_path, tolerance=0.02)
    assert result.passed is True


def test_gate_fails_when_new_composite_drops_beyond_tolerance(tmp_path: Path) -> None:
    new = json.loads(json.dumps(BASELINE))
    new["aggregate"]["faithfulness"] -= 0.5  # composite drops by ~0.125
    new_path = _write(tmp_path, "new.json", new)
    baseline_path = _write(tmp_path, "baseline.json", BASELINE)
    result = evaluate_gate(baseline_path, new_path, tolerance=0.02)
    assert result.passed is False
    assert result.delta < -0.02


def test_gate_result_renders_markdown_summary() -> None:
    result = GateResult(
        baseline_composite=0.9062,
        new_composite=0.8800,
        delta=-0.0262,
        tolerance=0.02,
        passed=False,
        baseline_metrics=BASELINE["aggregate"],
        new_metrics={
            "faithfulness": 0.95,
            "answer_relevancy": 0.85,
            "context_precision": 0.80,
            "context_recall": 0.90,
        },
    )
    md = result.to_markdown()
    assert "FAIL" in md
    assert "0.9062" in md
    assert "0.8800" in md
    assert "-0.0262" in md
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `uv run pytest tests/unit/test_ragas_gate.py -v`

Expected: all 6 tests fail with `ModuleNotFoundError: scripts.ragas_gate` (or similar).

- [ ] **Step 3: Implement the gate**

Create `scripts/ragas_gate.py`:

```python
"""Compare a RAGAS run to a baseline; non-zero exit if composite drops beyond tolerance."""

from __future__ import annotations

import argparse
import json
import sys
from dataclasses import dataclass
from pathlib import Path

METRIC_KEYS = (
    "faithfulness",
    "answer_relevancy",
    "context_precision",
    "context_recall",
)


def composite_score(aggregate: dict[str, float]) -> float:
    return sum(aggregate[k] for k in METRIC_KEYS) / len(METRIC_KEYS)


def load_result(path: Path) -> dict:
    return json.loads(Path(path).read_text())


@dataclass
class GateResult:
    baseline_composite: float
    new_composite: float
    delta: float
    tolerance: float
    passed: bool
    baseline_metrics: dict[str, float]
    new_metrics: dict[str, float]

    def to_markdown(self) -> str:
        status = "PASS" if self.passed else "FAIL"
        header = (
            f"### RAGAS Gate: {status}\n"
            f"- Baseline composite: **{self.baseline_composite:.4f}**\n"
            f"- New composite: **{self.new_composite:.4f}**\n"
            f"- Delta: **{self.delta:+.4f}** (tolerance −{self.tolerance:.2f})\n\n"
        )
        rows = ["| Metric | Baseline | New | Delta |", "|---|---:|---:|---:|"]
        for key in METRIC_KEYS:
            b = self.baseline_metrics[key]
            n = self.new_metrics[key]
            rows.append(f"| {key} | {b:.4f} | {n:.4f} | {n - b:+.4f} |")
        return header + "\n".join(rows)


def evaluate_gate(baseline_path: Path, new_path: Path, tolerance: float) -> GateResult:
    baseline = load_result(baseline_path)["aggregate"]
    new = load_result(new_path)["aggregate"]
    b_comp = composite_score(baseline)
    n_comp = composite_score(new)
    delta = n_comp - b_comp
    return GateResult(
        baseline_composite=b_comp,
        new_composite=n_comp,
        delta=delta,
        tolerance=tolerance,
        passed=delta >= -tolerance,
        baseline_metrics=baseline,
        new_metrics=new,
    )


def main() -> int:
    parser = argparse.ArgumentParser(description="RAGAS regression gate.")
    parser.add_argument("--baseline", required=True, type=Path)
    parser.add_argument("--new", required=True, type=Path)
    parser.add_argument("--tolerance", type=float, default=0.02)
    parser.add_argument(
        "--output",
        type=Path,
        default=None,
        help="If provided, write the markdown summary to this path.",
    )
    args = parser.parse_args()

    result = evaluate_gate(args.baseline, args.new, args.tolerance)
    md = result.to_markdown()
    print(md)
    if args.output:
        args.output.write_text(md)
    return 0 if result.passed else 1


if __name__ == "__main__":
    sys.exit(main())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `uv run pytest tests/unit/test_ragas_gate.py -v`

Expected: all 6 tests pass.

- [ ] **Step 5: Smoke-test the CLI against the real baseline**

Run:

```bash
uv run python scripts/ragas_gate.py \
  --baseline evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json \
  --new evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json
```

Expected: exit code 0, prints "RAGAS Gate: PASS", delta `+0.0000`.

Then test failure mode:

```bash
uv run python scripts/ragas_gate.py \
  --baseline evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json \
  --new evaluation/results/v4-pre-rrf-rerank/v4-pre-rrf-rerank.json
```

Expected: exit code 1, "FAIL" (v4 composite was lower).

- [ ] **Step 6: Lint**

Run: `uv run ruff check . && uv run ruff format --check .`

Expected: no errors.

---

## Task 6: Dockerfile for the API service

**Files:**
- Create: `deploy/Dockerfile.api`
- Create: `deploy/.dockerignore`

**Context:** Multi-stage build using the official `uv` builder image. Final image is `python:3.12-slim` plus the resolved `.venv`. Builds for `linux/amd64` (EC2 target). The `.dockerignore` keeps the build context small.

- [ ] **Step 1: Create `deploy/.dockerignore`**

```
**/.git
**/.gitignore
**/.venv
**/__pycache__
**/*.pyc
**/.pytest_cache
**/.ruff_cache
**/.mypy_cache
**/.env
**/.env.local
data/
evaluation/results/
logs/
notebooks/
images/
docs/
tests/
.vscode/
.claude/
*.md
!README.md
```

- [ ] **Step 2: Create `deploy/Dockerfile.api`**

```dockerfile
# syntax=docker/dockerfile:1.7

FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder
WORKDIR /app
ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project
COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH=/app \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:8000/health || exit 1
CMD ["uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

- [ ] **Step 3: Confirm Colima is running**

Run: `colima status`

Expected: status shows `Running`. If not, run `colima start` and re-check.

- [ ] **Step 4: Build the image locally**

Run from repo root:

```bash
docker buildx build \
  --platform linux/amd64 \
  --file deploy/Dockerfile.api \
  --tag research-rag-api:dev \
  --load \
  .
```

Expected: build succeeds, final image `research-rag-api:dev` listed by `docker images research-rag-api`.

- [ ] **Step 5: Smoke-run the container against the real `.env`**

Run:

```bash
docker run --rm -d --name rag-api-smoke \
  --env-file .env \
  -p 8000:8000 \
  research-rag-api:dev
```

Wait ~5 seconds, then:

```bash
curl -fsS http://localhost:8000/health
```

Expected: `{"status":"ok","pipeline_version":"..."}`. Tear down:

```bash
docker stop rag-api-smoke
```

---

## Task 7: Dockerfile for the Streamlit UI

**Files:**
- Create: `deploy/Dockerfile.ui`

**Context:** Structurally identical to `Dockerfile.api`; only the `CMD` differs. Streamlit listens on port 8501 inside the container.

- [ ] **Step 1: Create `deploy/Dockerfile.ui`**

```dockerfile
# syntax=docker/dockerfile:1.7

FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder
WORKDIR /app
ENV UV_LINK_MODE=copy \
    UV_COMPILE_BYTECODE=1
COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev --no-install-project
COPY src ./src
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src
ENV PATH="/app/.venv/bin:$PATH" \
    PYTHONPATH=/app \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
EXPOSE 8501
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -fsS http://localhost:8501/_stcore/health || exit 1
CMD ["streamlit", "run", "src/ui/app.py", \
     "--server.port=8501", \
     "--server.address=0.0.0.0", \
     "--server.headless=true", \
     "--browser.gatherUsageStats=false"]
```

- [ ] **Step 2: Build the image**

Run:

```bash
docker buildx build \
  --platform linux/amd64 \
  --file deploy/Dockerfile.ui \
  --tag research-rag-ui:dev \
  --load \
  .
```

Expected: build succeeds.

- [ ] **Step 3: Smoke-run against the local FastAPI started in Task 6 (or fresh)**

In one terminal:

```bash
docker run --rm -d --name rag-api-smoke --env-file .env -p 8000:8000 research-rag-api:dev
```

In another:

```bash
docker run --rm -d --name rag-ui-smoke \
  -e API_BASE_URL=http://host.docker.internal:8000 \
  -p 8501:8501 \
  research-rag-ui:dev
```

(On Linux without Docker Desktop's networking shim, replace `host.docker.internal` with the host IP, e.g. `172.17.0.1`, or use `--network host`.)

Open `http://localhost:8501` — sidebar should say "API healthy — pipeline `<version>`".

Tear down:

```bash
docker stop rag-ui-smoke rag-api-smoke
```

---

## Task 8: docker compose stack

**Files:**
- Create: `deploy/docker-compose.yml`

**Context:** Three services: `caddy`, `api`, `ui`. Caddy is the only one binding to host ports. `api` and `ui` are reachable only via the internal compose network at `api:8000` and `ui:8501`. The `caddy` Caddyfile is mounted read-only.

On EC2 the file lives at `/opt/research-rag/deploy/docker-compose.yml`; for local smoke we bring it up from the repo. The `env_file` path is the absolute path EC2 will use; for local dev we override with a docker compose `--env-file` flag (see step 3).

- [ ] **Step 1: Create `deploy/docker-compose.yml`**

```yaml
name: research-rag

services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    environment:
      DUCKDNS_HOST: ${DUCKDNS_HOST:?DUCKDNS_HOST is required}
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      api:
        condition: service_healthy
      ui:
        condition: service_started

  api:
    image: ${ECR_REGISTRY:-local}/research-rag-api:${IMAGE_TAG:-latest}
    restart: unless-stopped
    env_file:
      - ${ENV_FILE:-/opt/research-rag/.env}
    expose:
      - "8000"
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s

  ui:
    image: ${ECR_REGISTRY:-local}/research-rag-ui:${IMAGE_TAG:-latest}
    restart: unless-stopped
    environment:
      API_BASE_URL: http://api:8000
      REPO_URL: ${REPO_URL:-https://github.com/}
    expose:
      - "8501"
    depends_on:
      api:
        condition: service_healthy

volumes:
  caddy_data:
  caddy_config:
```

- [ ] **Step 2: Validate the compose file parses**

Run from repo root:

```bash
DUCKDNS_HOST=:80 ECR_REGISTRY=local IMAGE_TAG=dev \
  docker compose --env-file .env -f deploy/docker-compose.yml config >/dev/null
```

Expected: no output, exit 0. (`DUCKDNS_HOST=:80` tells Caddy to bind plain HTTP on port 80 only, skipping ACME — used for local smoke. Real EC2 value is set via SSM in Plan B.)

(End-to-end smoke deferred to Task 9 once the Caddyfile exists — the compose stack can't start without it.)

---

## Task 9: Caddyfile + local end-to-end smoke

**Files:**
- Create: `deploy/Caddyfile`

**Context:** Caddy normally auto-provisions HTTPS for any hostname. For local smoke we set the Caddyfile site address to `:80` so Caddy listens on plain HTTP only (no ACME, no internal CA cert) — done by overriding `DUCKDNS_HOST=:80` in the environment. The real EC2 value will be the DuckDNS subdomain, and Caddy will then provision Let's Encrypt automatically.

- [ ] **Step 1: Create `deploy/Caddyfile`**

```
{$DUCKDNS_HOST} {
    encode gzip

    handle /api/* {
        uri strip_prefix /api
        reverse_proxy api:8000
    }

    handle {
        reverse_proxy ui:8501
    }
}
```

- [ ] **Step 2: Bring up the full stack locally**

Make sure `research-rag-api:dev` and `research-rag-ui:dev` from Tasks 6 and 7 exist (`docker images | grep research-rag`). Override the image refs in compose by setting `ECR_REGISTRY=local` and `IMAGE_TAG=dev` plus `DUCKDNS_HOST=:80`:

```bash
DUCKDNS_HOST=:80 ECR_REGISTRY=local IMAGE_TAG=dev ENV_FILE=$(pwd)/.env \
  docker compose -f deploy/docker-compose.yml up -d
```

Watch logs:

```bash
docker compose -f deploy/docker-compose.yml logs -f --tail=50
```

When Caddy logs `serving initial configuration`, hit it:

```bash
curl -fsS http://localhost/api/health
```

Expected: `{"status":"ok","pipeline_version":"..."}`.

Open `http://localhost/` in a browser — Streamlit UI renders, sidebar shows the pipeline version.

- [ ] **Step 3: Tear down**

```bash
docker compose -f deploy/docker-compose.yml down -v
```

---

## Task 10: SSM secrets fetch script

**Files:**
- Create: `deploy/fetch-secrets.sh`

**Context:** Runs as `root` on EC2 before docker compose starts. Reads all `/research-rag/*` params from SSM (instance role provides `ssm:GetParametersByPath`) and writes them to `/opt/research-rag/.env`. SSM names like `/research-rag/google-api-key` become env vars `GOOGLE_API_KEY`.

Cannot be end-to-end smoke-tested in this plan (depends on Plan B's SSM params + IAM role). Smoke is limited to syntax and shellcheck.

- [ ] **Step 1: Create `deploy/fetch-secrets.sh`**

```bash
#!/usr/bin/env bash
# Pulls /research-rag/* params from AWS SSM Parameter Store into /opt/research-rag/.env.
# Runs as root on EC2 via the research-rag-secrets.service systemd unit.

set -euo pipefail

REGION="${AWS_REGION:-us-east-1}"
PREFIX="/research-rag"
OUT="/opt/research-rag/.env"
TMP="$(mktemp)"

trap 'rm -f "$TMP"' EXIT

# get-parameters-by-path returns Name<TAB>Value pairs.
aws ssm get-parameters-by-path \
    --path "$PREFIX" \
    --with-decryption \
    --region "$REGION" \
    --query "Parameters[].[Name,Value]" \
    --output text \
| while IFS=$'\t' read -r name value; do
    [[ -z "$name" ]] && continue
    key=$(basename "$name" | tr '[:lower:]-' '[:upper:]_')
    # Escape value for shell: wrap in single quotes, escape embedded single quotes.
    escaped=${value//\'/\'\\\'\'}
    printf "%s='%s'\n" "$key" "$escaped"
done > "$TMP"

if [[ ! -s "$TMP" ]]; then
    echo "fetch-secrets: no parameters found under $PREFIX" >&2
    exit 1
fi

install -m 600 -o root -g root "$TMP" "$OUT"
echo "fetch-secrets: wrote $(wc -l < "$OUT") variables to $OUT"
```

- [ ] **Step 2: Make it executable**

Run: `chmod +x deploy/fetch-secrets.sh`

- [ ] **Step 3: Lint with shellcheck if available**

Run: `shellcheck deploy/fetch-secrets.sh || echo "shellcheck not installed — skip"`

Expected: no errors (or skip notice). If shellcheck reports warnings, fix them.

- [ ] **Step 4: Verify bash syntax parses**

Run: `bash -n deploy/fetch-secrets.sh && echo OK`

Expected: `OK`.

---

## Task 11: DuckDNS dynamic IP updater

**Files:**
- Create: `deploy/duckdns-update.sh`

**Context:** Called by a systemd timer on the EC2 box (boot + every 30 minutes). Hits the DuckDNS update URL with the current public IP. Token + host come from `/opt/research-rag/.env` (populated by Task 10's fetch script).

- [ ] **Step 1: Create `deploy/duckdns-update.sh`**

```bash
#!/usr/bin/env bash
# Updates the DuckDNS A record with the EC2 instance's current public IPv4.
# Reads DUCKDNS_HOST and DUCKDNS_TOKEN from /opt/research-rag/.env.

set -euo pipefail

ENV_FILE="${ENV_FILE:-/opt/research-rag/.env}"

if [[ ! -r "$ENV_FILE" ]]; then
    echo "duckdns-update: missing $ENV_FILE" >&2
    exit 1
fi

# shellcheck disable=SC1090
source "$ENV_FILE"

: "${DUCKDNS_HOST:?DUCKDNS_HOST not set}"
: "${DUCKDNS_TOKEN:?DUCKDNS_TOKEN not set}"

# DuckDNS expects only the subdomain (the part before .duckdns.org).
domain="${DUCKDNS_HOST%%.duckdns.org}"

response=$(curl -fsS "https://www.duckdns.org/update?domains=${domain}&token=${DUCKDNS_TOKEN}&ip=" || true)

if [[ "$response" != "OK" ]]; then
    echo "duckdns-update: unexpected response: $response" >&2
    exit 1
fi

echo "duckdns-update: $domain.duckdns.org updated"
```

- [ ] **Step 2: Make it executable**

Run: `chmod +x deploy/duckdns-update.sh`

- [ ] **Step 3: Verify bash syntax**

Run: `bash -n deploy/duckdns-update.sh && echo OK`

Expected: `OK`.

- [ ] **Step 4: Optional shellcheck**

Run: `shellcheck deploy/duckdns-update.sh || echo "shellcheck not installed — skip"`

---

## Task 12: systemd units (boot orchestration)

**Files:**
- Create: `deploy/systemd/research-rag-secrets.service`
- Create: `deploy/systemd/research-rag.service`
- Create: `deploy/systemd/duckdns-update.service`
- Create: `deploy/systemd/duckdns-update.timer`

**Context:** Boot order on EC2:
1. `research-rag-secrets.service` — one-shot, fetches SSM → writes `.env`.
2. `duckdns-update.service` (one-shot) — first invocation by timer at boot.
3. `research-rag.service` — runs `docker compose up -d`.

All units assume the working tree is laid out at `/opt/research-rag/` (Plan B installs them). Compose runs from `/opt/research-rag/deploy/`.

- [ ] **Step 1: Create `deploy/systemd/research-rag-secrets.service`**

```ini
[Unit]
Description=research-rag: fetch secrets from AWS SSM into /opt/research-rag/.env
Wants=network-online.target
After=network-online.target
Before=research-rag.service

[Service]
Type=oneshot
ExecStart=/opt/research-rag/deploy/fetch-secrets.sh
RemainAfterExit=yes
TimeoutStartSec=120
User=root

[Install]
WantedBy=multi-user.target
```

- [ ] **Step 2: Create `deploy/systemd/research-rag.service`**

```ini
[Unit]
Description=research-rag: docker compose stack
Requires=docker.service research-rag-secrets.service
After=docker.service research-rag-secrets.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/research-rag/deploy
EnvironmentFile=/opt/research-rag/.env
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

- [ ] **Step 3: Create `deploy/systemd/duckdns-update.service`**

```ini
[Unit]
Description=research-rag: refresh DuckDNS A record with current public IP
Wants=network-online.target
After=network-online.target research-rag-secrets.service
Requires=research-rag-secrets.service

[Service]
Type=oneshot
ExecStart=/opt/research-rag/deploy/duckdns-update.sh
User=root
```

- [ ] **Step 4: Create `deploy/systemd/duckdns-update.timer`**

```ini
[Unit]
Description=research-rag: DuckDNS refresh timer

[Timer]
OnBootSec=30s
OnUnitActiveSec=30min
Persistent=true
Unit=duckdns-update.service

[Install]
WantedBy=timers.target
```

- [ ] **Step 5: Validate unit syntax**

If `systemd-analyze` is available locally (it often is on WSL2):

```bash
systemd-analyze verify deploy/systemd/*.service deploy/systemd/*.timer || \
  echo "systemd-analyze not usable here; verification deferred to Plan B"
```

Expected: no errors, or the deferral message.

---

## Task 13: Deploy README for operators

**Files:**
- Create: `deploy/README.md`

**Context:** One-page operator reference. The CI/CD workflows + AWS provisioning live in their own plans/docs; this file is just "what each file in `deploy/` is and how to bring the stack up locally."

- [ ] **Step 1: Create `deploy/README.md`**

```markdown
# deploy/

All deployment artifacts for the research-rag EC2 stack. App-side code is in `../src/`; runtime infrastructure (AWS resources, GitHub Actions) lives elsewhere — see the design spec at [`docs/superpowers/specs/2026-05-21-ec2-free-tier-deployment-design.md`](../docs/superpowers/specs/2026-05-21-ec2-free-tier-deployment-design.md).

## Files

| File | Where it runs | Purpose |
|---|---|---|
| `Dockerfile.api` | local build, ECR | Multi-stage image for FastAPI |
| `Dockerfile.ui` | local build, ECR | Multi-stage image for Streamlit |
| `.dockerignore` | docker build context | Keeps images lean |
| `docker-compose.yml` | EC2 (`/opt/research-rag/deploy/`) and local smoke | 3-service stack: caddy + api + ui |
| `Caddyfile` | inside the `caddy` container | Reverse proxy + auto-TLS |
| `fetch-secrets.sh` | EC2, as root, via systemd | Pulls `/research-rag/*` SSM params into `/opt/research-rag/.env` |
| `duckdns-update.sh` | EC2, as root, via systemd timer | Updates DuckDNS A record with current public IP |
| `systemd/research-rag-secrets.service` | EC2 | One-shot: runs `fetch-secrets.sh` before compose |
| `systemd/research-rag.service` | EC2 | Brings up the compose stack on boot |
| `systemd/duckdns-update.service` | EC2 | One-shot: runs `duckdns-update.sh` |
| `systemd/duckdns-update.timer` | EC2 | Calls the service at boot + every 30 minutes |

## Local smoke test

Pre-reqs: Colima running (`colima status` shows `Running`), `.env` populated at repo root (`GOOGLE_API_KEY`, `JINA_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`).

Build the images (from repo root):

```bash
docker buildx build --platform linux/amd64 -f deploy/Dockerfile.api -t research-rag-api:dev --load .
docker buildx build --platform linux/amd64 -f deploy/Dockerfile.ui  -t research-rag-ui:dev  --load .
```

Bring the stack up:

```bash
DUCKDNS_HOST=:80 \
ECR_REGISTRY=local \
IMAGE_TAG=dev \
ENV_FILE=$(pwd)/.env \
  docker compose -f deploy/docker-compose.yml up -d
```

Verify:

```bash
curl -fsS http://localhost/api/health
# → {"status":"ok","pipeline_version":"..."}
```

Open `http://localhost/` for the Streamlit UI.

Tear down:

```bash
docker compose -f deploy/docker-compose.yml down -v
```

## Environment variables consumed at runtime

| Variable | Required | Source on EC2 | Notes |
|---|---|---|---|
| `GOOGLE_API_KEY` | yes | SSM `/research-rag/google-api-key` | Gemini embedding + LLM |
| `JINA_API_KEY` | yes | SSM `/research-rag/jina-api-key` | Reranker |
| `QDRANT_URL` | yes | SSM `/research-rag/qdrant-url` | Qdrant Cloud cluster |
| `QDRANT_API_KEY` | yes | SSM `/research-rag/qdrant-api-key` | |
| `LANGSMITH_API_KEY` | yes | SSM `/research-rag/langsmith-api-key` | Tracing |
| `LANGSMITH_PROJECT` | yes | SSM `/research-rag/langsmith-project` | |
| `DUCKDNS_HOST` | yes | SSM `/research-rag/duckdns-host` | Caddy uses this as the cert hostname |
| `DUCKDNS_TOKEN` | yes | SSM `/research-rag/duckdns-token` | Updater token |
| `ECR_REGISTRY` | yes | SSM `/research-rag/ecr-registry` | `<acct>.dkr.ecr.us-east-1.amazonaws.com` |
| `IMAGE_TAG` | no | compose default `latest` | Pin to a sha for rollback |
| `API_BASE_URL` | no | compose default `http://api:8000` | Streamlit → FastAPI |
| `REPO_URL` | no | compose default `https://github.com/` | Sidebar link |

## What this directory does NOT do

- It does not provision AWS resources. See Plan B.
- It does not configure GitHub Actions. See Plan C.
- It does not ingest data into Qdrant. That stays a local-CLI workflow (`uv run python main.py`).
```

---

## Final verification

After all 13 tasks are complete, run the following to verify nothing regressed:

- [ ] **All tests still pass (offline subset)**

Run: `uv run pytest -m 'not live' -v`

Expected: every test passes.

- [ ] **Lint clean**

Run: `uv run ruff check . && uv run ruff format --check .`

Expected: no errors.

- [ ] **Stack boots locally end-to-end under Colima**

Re-run the smoke test from `deploy/README.md`. Verify:
- `curl http://localhost/api/health` returns 200 with `pipeline_version`.
- Browser at `http://localhost/` shows the Streamlit page with a green "API healthy" sidebar.
- A real query (e.g. "What is LoRA?") returns an answer and source chunks.

- [ ] **Image sizes are reasonable**

Run: `docker images research-rag-api research-rag-ui --format '{{.Repository}}:{{.Tag}}\t{{.Size}}'`

Expected: each image well under 1 GB (the `python:3.12-slim` + venv shape typically lands around 500–700 MB).

- [ ] **Working tree contains all expected changes (uncommitted)**

Run: `git status --short`

Expected: a set of `M` and `??` entries covering exactly the files in the File Map above — no surprises, no extras. The user reviews this state and decides how to slice it into commits.

After this you're ready to start **Plan B (AWS infrastructure provisioning)**.

---

## What this plan deliberately does NOT include

- **No `.env.template` updates** — additions like `DUCKDNS_HOST` are EC2-only and would be misleading in the local template.
- **No CI workflow files** — those live in Plan C, where they can be authored alongside the OIDC role they need.
- **No real DuckDNS or ACME testing** — both require a public IP, deferred to the first real deploy in Plan C.
- **No CLAUDE.md update** — the deploy work doesn't change the project's core philosophy; CLAUDE.md gets a brief deploy section in Plan C once the system is live.
- **No refactor of `src/api/main.py`** beyond adding `/health`. The module-level instantiation of `Retriever` and `RAGChain` is an existing pattern; changing it is out of scope.
