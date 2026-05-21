# Deployment Design — research-rag v3.1.1 on AWS EC2 free tier

**Status:** Approved — ready for implementation plan.
**Date:** 2026-05-21
**Branch at time of design:** `dev`
**Pipeline version being deployed:** `v3.1.1-prefetch-scale-30` (current best, composite 0.9062)
**Goal:** Public, HTTPS-accessible portfolio demo of the current RAG system, hosted entirely on free-tier services for ~12 months.

---

## Locked decisions

| Topic | Decision | Why |
|---|---|---|
| UI | Streamlit | CLAUDE.md spec; fastest portfolio polish |
| Public access | DuckDNS subdomain + Caddy auto-TLS (Let's Encrypt) | Free, stable URL, no domain purchase, browser-trusted HTTPS |
| CI/CD gate | Full CLAUDE.md pipeline (ruff → pytest → RAGAS regression → build → deploy) | Matches the portfolio narrative; user accepts API quota cost |
| AWS account | New, free-tier eligible | t2.micro / ECR / EBS within free tier for 12 months |
| Secrets | AWS SSM Parameter Store (Standard tier) + IAM instance role | Free; production-grade story for portfolio |
| Deploy trigger | `aws ssm send-command` from GitHub Actions runner | No inbound SSH port; OIDC-based AWS auth from GH |
| Region | `us-east-1` | Cheapest, widest free-tier coverage |
| Host OS | Amazon Linux 2023 | SSM agent + Docker engine preinstalled, free-tier AMI |

---

## Section 1 — Architecture & topology

```
                                    ┌─────────────────────┐
   GitHub                            │     EC2 t2.micro    │
   ┌─────────────────────┐           │  (Amazon Linux 2023)│
   │ Actions:            │  SSM      │                     │
   │  ruff → pytest ──┐  │  Send-    │  ┌───────────────┐  │
   │  → ragas-gate  ──┤  │ Command   │  │ caddy         │  │
   │  → build+push  ──┼──┼──────────►│  │  :80, :443    │  │
   │  → ssm trigger ──┘  │           │  │  auto-TLS via │  │
   └──────────────────────┘          │  │  DuckDNS      │  │
                │                    │  └───────┬───────┘  │
                ▼                    │     ┌────┴────┐     │
        ┌───────────────┐            │     ▼         ▼     │
        │   AWS ECR     │ docker     │  ┌──────┐  ┌─────┐  │
        │  api / ui     │◄───────────│  │ ui   │  │ api │  │
        │   images      │   pull     │  │:8501 │  │:8000│  │
        └───────────────┘            │  │Strea │  │Fast │  │
                                     │  │mlit  │  │API  │  │
                                     │  └──────┘  └──┬──┘  │
                                     │               │     │
                                     │  ┌────────────┴───┐ │
                                     │  │ IAM instance   │ │
                                     │  │ role → SSM     │ │
                                     │  │ Parameter Store│ │
                                     │  └────────────────┘ │
                                     └──────────┬──────────┘
                                                │  HTTPS
                          ┌─────────────────────┼──────────────────┐
                          ▼                     ▼                  ▼
                    Qdrant Cloud           Gemini API         Jina rerank
                    (free 1 GB)            LangSmith
```

### Routing on the box (Caddy)

- `https://<your-subdomain>.duckdns.org/`        → `ui:8501` (Streamlit)
- `https://<your-subdomain>.duckdns.org/api/*`   → `api:8000` (FastAPI)
- HTTP on `:80` → 301 redirect to HTTPS

Streamlit calls FastAPI internally over the docker compose network at `http://api:8000` (no extra public hop).

### Containers on the box

| Container | Image | Purpose | Ports |
|---|---|---|---|
| `caddy` | `caddy:2-alpine` | Reverse proxy + automatic Let's Encrypt | 80, 443 (host) |
| `api` | ECR: `research-rag-api` | FastAPI (`src/api/main.py`) | 8000 (internal only) |
| `ui` | ECR: `research-rag-ui` | Streamlit page | 8501 (internal only) |

### Resource budget (t2.micro: 1 vCPU, 1 GiB RAM, 30 GiB EBS)

| Component | RAM est. | Notes |
|---|---|---|
| OS + dockerd | ~250 MB | |
| caddy | ~30 MB | |
| api (FastAPI + retriever client) | ~150 MB | No model weights on box |
| ui (Streamlit) | ~250 MB | Heaviest of the three |
| Headroom | ~320 MB | Safe |

All embedding, reranking, LLM, and vector storage happens off-box — the EC2 instance is essentially an orchestrator + UI shell.

### Why this shape

- Single failure domain is fine for a portfolio demo.
- Compose mirrors local dev (CLAUDE.md locks Docker + compose).
- No load balancer, no autoscaling — all extra cost.
- Caddy chosen over Nginx because it does ACME (Let's Encrypt) with two lines of config.

---

## Section 2 — CI/CD pipeline

```
Push to dev               Push to main                   Manual (workflow_dispatch)
─────────────             ─────────────                  ─────────────
   │                         │                              │
   ▼                         ▼                              ▼
 ┌───────┐               ┌───────┐                       ┌────────┐
 │ ruff  │               │ ruff  │                       │ ragas  │
 │ pytest│               │ pytest│                       │ on full│
 └───────┘               └───┬───┘                       │ testset│
   (stop)                    │                           │ +log to│
                             ▼                           │ Lang-  │
                       ┌──────────────┐                  │ Smith  │
                       │ ragas-gate   │ ← block if       └────────┘
                       │ (composite   │   composite drops
                       │  ≥ baseline  │   below v3.1.1
                       │  − 0.02)     │   tolerance
                       └──────┬───────┘
                              │ green
                              ▼
                       ┌──────────────┐
                       │ build images │ docker buildx for api + ui
                       │ push to ECR  │ tagged with git sha + 'latest'
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │ ssm send-cmd │ remote: docker compose pull
                       │ to EC2       │         && docker compose up -d
                       │ wait+verify  │ then curl healthcheck
                       └──────────────┘
```

### Workflow files (`.github/workflows/`)

| File | Trigger | Jobs |
|---|---|---|
| `ci.yml` | push to any branch, PRs | `lint`, `test` (pytest `-m 'not live'`) |
| `deploy.yml` | push to `main` | `lint`, `test`, `ragas-gate`, `build-push`, `deploy` |
| `ragas-ondemand.yml` | `workflow_dispatch` | `ragas-full` (logs new experiment to LangSmith, snapshots `evaluation/results/<name>.json`, opens a PR with the snapshot) |
| `live-tests.yml` | `workflow_dispatch` | runs `pytest -m live` against live Gemini / Jina / Qdrant |

### Gate semantics

- **Baseline reference:** `evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json` (the locked best, composite 0.9062).
- **Pass condition:** `composite_new ≥ composite_baseline − 0.02`. Tolerance accounts for RAGAS LLM-as-judge variance (~±0.01–0.02).
- **No per-metric guard.** A single-metric dip is acceptable as long as composite holds. (Rationale: v3.1.1 already has CP=0.83 with strong composite — narrow per-metric guards on a noisy judge would block more good-faith deploys than they save.)
- **Output:** Posts a markdown comment on the commit with the metric table + delta vs baseline.

### Tests in CI

- Existing `tests/` directory mirrors `src/`.
- **Live test marker:** introduce `@pytest.mark.live` for tests that hit Gemini / Jina / Qdrant. Configure in `pyproject.toml`:
  ```toml
  [tool.pytest.ini_options]
  markers = ["live: hits live external APIs; excluded by default"]
  ```
  - `ci.yml` runs `pytest -m 'not live'` (fast, free)
  - A separate `live-tests.yml` (`workflow_dispatch`) runs `pytest -m live` when wanted
- RAGAS gate itself is the integration signal end-to-end.

### Build + push

- `docker buildx build` for each of `api` and `ui` (separate `deploy/Dockerfile.api` and `deploy/Dockerfile.ui`).
- Push to ECR with two tags: `:${{ github.sha }}` (immutable, for rollback) and `:latest` (compose pulls this).
- ECR auth via GitHub OIDC → IAM role (no long-lived AWS access keys in GitHub Secrets).

### Deploy step (no SSH port open)

GitHub Actions assumes an OIDC-trusted IAM role and calls:

```
aws ssm send-command \
  --instance-ids i-xxxx \
  --document-name AWS-RunShellScript \
  --parameters 'commands=["cd /opt/research-rag && \
                          aws ecr get-login-password ... | docker login ... && \
                          docker compose pull && \
                          docker compose up -d && \
                          curl -fsS http://localhost:8000/health"]'
```

Action waits for command completion via `aws ssm wait command-executed`, parses status, fails the deploy if the healthcheck didn't return 200.

### Secrets in GitHub

Only two secrets needed in GitHub:
- `AWS_ROLE_TO_ASSUME` — ARN of the OIDC-trusted IAM role
- `AWS_REGION` — `us-east-1`

All runtime secrets (Gemini, Jina, LangSmith, Qdrant) live in SSM Parameter Store, never in GitHub.

---

## Section 3 — AWS resources & IAM

### One-time AWS setup (manual, ~30 min)

| # | Resource | Purpose | Free-tier note |
|---|---|---|---|
| 1 | Account + billing alarm | Safety net | Set alarm at $1 |
| 2 | IAM user `dev-admin` with MFA | Stop using root | Free |
| 3 | OIDC identity provider for `token.actions.githubusercontent.com` | Lets GH Actions assume an AWS role without static keys | Free |
| 4 | IAM role `github-actions-deploy` | Assumed by GH Actions | Free |
| 5 | IAM role `ec2-rag-instance-role` | Attached to the EC2 instance | Free |
| 6 | ECR repos: `research-rag-api`, `research-rag-ui` | Image registry | Free 500 MB private storage |
| 7 | EC2 key pair (for emergency SSH only) | Break-glass access | Free |
| 8 | Security group `rag-sg` | Allows :80, :443 inbound from `0.0.0.0/0`. No SSH from the internet — SSH only via SSM Session Manager | Free |
| 9 | EC2 instance: t2.micro, Amazon Linux 2023, 30 GiB gp3, instance role from #5, SG from #8 | The box | Free for 12 mo |
| 10 | SSM Parameter Store entries (Standard tier) | Runtime secrets | Free for Standard params |
| 11 | DuckDNS record pointing at the EC2 public IP | Public hostname | Free |

### IAM role: `github-actions-deploy`

**Trust policy** — only assumable by the GitHub repo's `main` branch via OIDC:

```jsonc
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<acct>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub":
          "repo:<owner>/research-rag:ref:refs/heads/main"
      }
    }
  }]
}
```

**Permissions** — least privilege:
- `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload` on the two ECR repos
- `ssm:SendCommand` on the EC2 instance ARN only
- `ssm:GetCommandInvocation` for status polling

### IAM role: `ec2-rag-instance-role` (attached to the instance)

- `AmazonSSMManagedInstanceCore` (AWS-managed) — enables SSM agent, Session Manager, Send-Command
- Inline policy: `ssm:GetParameter` / `ssm:GetParameters` on `arn:aws:ssm:us-east-1:<acct>:parameter/research-rag/*`
- Inline policy: `ecr:GetAuthorizationToken` + ECR pull actions on the two repos

### SSM Parameter Store layout

```
/research-rag/google-api-key       SecureString   # Gemini (embedding + LLM)
/research-rag/jina-api-key         SecureString
/research-rag/qdrant-url           String
/research-rag/qdrant-api-key       SecureString
/research-rag/langsmith-api-key    SecureString
/research-rag/langsmith-project    String
/research-rag/duckdns-host         String         # consumed by Caddyfile
/research-rag/duckdns-token        SecureString   # consumed by duckdns-update.sh
/research-rag/ecr-registry         String         # `<acct>.dkr.ecr.us-east-1.amazonaws.com`, consumed by compose
```

`deploy/fetch-secrets.sh` on the EC2 box (run by systemd one-shot before docker compose) reads these and writes `/opt/research-rag/.env` consumed by compose. Refresh on boot only — no live polling.

### Security posture

- **No inbound SSH** — security group blocks port 22 from the internet entirely.
- **Break-glass SSH** via AWS Session Manager (`aws ssm start-session --target i-xxxx`) — no public key on the instance, no port open.
- **All inter-service traffic on the box** uses the docker compose internal network. Only Caddy binds to host ports.
- **Outbound** unrestricted (needed for Gemini/Jina/Qdrant calls).
- **No public S3, no Secrets Manager** — staying inside free tier and avoiding the $0.40/mo/secret cost.

---

## Section 4 — Deployment artifacts (files to add)

New top-level `deploy/` directory holds everything infra-related; nothing about app behavior changes.

```
deploy/
├── docker-compose.yml        # 3 services: caddy, api, ui
├── Caddyfile                 # reverse proxy + auto-TLS config
├── Dockerfile.api            # multi-stage build for FastAPI
├── Dockerfile.ui             # multi-stage build for Streamlit
├── fetch-secrets.sh          # pulls SSM params, writes /opt/research-rag/.env
├── duckdns-update.sh         # updates DuckDNS record with current EC2 public IP
└── systemd/
    ├── research-rag.service          # runs `docker compose up -d` on boot
    ├── research-rag-secrets.service  # one-shot, runs fetch-secrets.sh before compose
    └── duckdns-update.timer          # runs duckdns-update.sh on boot + every 30 min
```

New top-level apps:

```
src/ui/                       # Streamlit page (new module)
├── __init__.py
└── app.py                    # single-page chat-with-papers UI
```

### `deploy/docker-compose.yml` (sketch)

```yaml
services:
  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on: [api, ui]

  api:
    image: ${ECR_REGISTRY}/research-rag-api:latest
    restart: unless-stopped
    env_file: /opt/research-rag/.env
    expose: ["8000"]
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  ui:
    image: ${ECR_REGISTRY}/research-rag-ui:latest
    restart: unless-stopped
    environment:
      - API_BASE_URL=http://api:8000
    expose: ["8501"]
    depends_on:
      api: { condition: service_healthy }

volumes:
  caddy_data:
  caddy_config:
```

### `deploy/Caddyfile`

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

Caddy reads `DUCKDNS_HOST` from the .env on the box. ACME challenge happens on `:80` automatically; cert renews itself.

### `Dockerfile.api` (multi-stage, uv-based)

```dockerfile
FROM ghcr.io/astral-sh/uv:python3.12-bookworm-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY src ./src
RUN uv sync --frozen --no-dev

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY --from=builder /app/src /app/src
ENV PATH="/app/.venv/bin:$PATH" PYTHONPATH=/app
EXPOSE 8000
CMD ["uvicorn", "src.api.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`Dockerfile.ui` is structurally identical, with `CMD ["streamlit", "run", "src/ui/app.py", "--server.port=8501", "--server.address=0.0.0.0"]`.

### `deploy/fetch-secrets.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail
REGION="${AWS_REGION:-us-east-1}"
OUT="/opt/research-rag/.env"
PREFIX="/research-rag"
PARAMS=$(aws ssm get-parameters-by-path --path "$PREFIX" --with-decryption --region "$REGION" --query "Parameters[].[Name,Value]" --output text)
{
  while IFS=$'\t' read -r name value; do
    key=$(basename "$name" | tr '[:lower:]-' '[:upper:]_')
    printf '%s=%s\n' "$key" "$value"
  done <<< "$PARAMS"
} > "$OUT"
chmod 600 "$OUT"
```

Runs as `root` before docker compose starts. EC2 instance role provides the `ssm:GetParameters` permission.

### `deploy/systemd/research-rag.service`

```ini
[Unit]
Description=research-rag stack
Requires=docker.service research-rag-secrets.service
After=docker.service research-rag-secrets.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/research-rag/deploy
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down

[Install]
WantedBy=multi-user.target
```

### Streamlit UI scope (deliberately small)

- Text input → calls `POST /api/query` → renders answer + cited chunks below
- Sidebar shows `pipeline_version` from `/api/health` and a link to the GitHub repo
- No auth, no history, no settings — portfolio demo, not a product

### What does NOT belong here

- No nginx, no certbot, no Let's Encrypt cron — Caddy handles all of it.
- No `docker-compose.override.yml` for local dev — local dev keeps using `uv run` directly. Compose is deploy-only.
- No CloudWatch agent, no log shipping — `docker logs` + LangSmith are enough for portfolio scope.

---

## Section 5 — Ingestion, Qdrant lifecycle, runtime cost

### Ingestion is out of scope for the deploy

- The Qdrant Cloud free cluster should already be populated from the v3.1.1 evaluation runs.
- Ingestion stays a **local CLI workflow** (`main.py` → `SimpleIngestionPipeline`). It is never run on EC2.
- The EC2 stack reads from Qdrant; it does not write.

### Qdrant Cloud free tier — the suspension problem

Qdrant Cloud auto-suspends free clusters after **~7 days of zero traffic**. Resuming is manual via the dashboard and takes a few minutes. For a portfolio link people might hit sporadically, this would break the demo.

**Mitigation:** a small keep-alive cron on the EC2 box that pings Qdrant once a day so the cluster stays warm. Free, runs anyway.

```cron
# /etc/cron.d/qdrant-keepalive
0 6 * * * ec2-user curl -fsS -H "api-key: $QDRANT_API_KEY" "$QDRANT_URL/collections" >/dev/null 2>&1
```

A daily ping is plenty — Qdrant counts any authenticated request as activity.

### Runtime API cost (per real user query)

| Service | Calls per query | Approx free quota |
|---|---|---|
| Gemini embedding (`embed_content`) | 1 (cached after first hit) | Generous — embeddings are cheap |
| Gemini LLM (sub-query extraction) | 1 | Free tier sufficient for demo traffic |
| Gemini LLM (final answer) | 1 | Same |
| Jina rerank | 1 | 1M tokens/month free |
| Qdrant query | 1 hybrid query (dense + sparse internally) | Unmetered on free cluster |
| LangSmith trace | 1 | 5k traces/month free |

The embedding cache (`data/cache/query_embeddings.jsonl`) lives on the EC2 disk via a docker volume — repeat queries don't re-embed.

### Cost guardrails

- **AWS billing alarm at $1** — primary backstop. Email alert if anything goes wrong.
- **CloudWatch free-tier basic metrics** on the EC2 instance (CPU, network, disk) — free, no agent install needed.
- **No NAT Gateway, no Elastic IP** — both leak money. Use the EC2 auto-assigned public IP and update DuckDNS via a tiny startup script if the IP changes after a stop/start. (For a always-on demo this rarely matters, but worth handling.)
- **No EBS snapshots** — nothing on disk is worth backing up; everything reproducible from git + ECR + Qdrant Cloud.
- **Stop, don't terminate** if you ever pause the project — `stop` keeps the volume free; terminate destroys it.

### Failure modes & manual recovery

| Failure | Symptom | Recovery |
|---|---|---|
| Qdrant suspended | API returns errors, UI shows "service unavailable" | Log into Qdrant Cloud, click resume. Keep-alive cron prevents recurrence. |
| EC2 stop/start changed public IP | DuckDNS points to wrong IP | Re-run the DuckDNS update script (or set it on boot via systemd) |
| Gemini quota exhausted | Errors on `/query` | Wait for quota reset (per-day on free tier) or rotate to a new key |
| Free tier expired (after 12 months) | AWS bill notification | Either migrate to HF Spaces / Render or accept ~$8/mo |
| Image pull fails (ECR) | Compose can't start | SSM in, `docker compose pull` manually with verbose logs |
| RAGAS gate blocks deploy | Deploy workflow red | Re-run on-demand RAGAS, inspect snapshot, decide if a baseline update is warranted |

### What this design does NOT promise

- High availability — one box, one region. Acceptable for a portfolio link.
- Scale beyond ~1 concurrent user — t2.micro has 1 vCPU; concurrent Gemini calls will block on each other.
- Disaster recovery — losing the EC2 instance means re-running terraform-or-clicks to recreate. Acceptable, no real data on the box.
- 100% uptime past the 12-month free window. After that, this stack either migrates or starts costing money.

---

## Open items not blocking the design

- **`/health` endpoint** does not exist on the FastAPI app yet. To be added during implementation (Step in the plan). Returns `{ "status": "ok", "pipeline_version": "v3.1.1-prefetch-scale-30" }`.
- **DuckDNS dynamic IP updater** — `deploy/duckdns-update.sh` + `duckdns-update.timer` (boot + 30 min). Token stored in SSM as `/research-rag/duckdns-token`.
- **First-run image build** — initial ECR push will be a manual `docker buildx ... && docker push` from a workstation; once GitHub Actions has the OIDC role wired, all subsequent pushes are automated.
- **Baseline composite reference value** — `0.9062` is treated as the gate baseline; if a future RAGAS run intentionally beats it the doc/baseline value gets updated by PR.

