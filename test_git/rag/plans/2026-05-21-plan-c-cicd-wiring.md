# Plan C — CI/CD Wiring (GitHub Actions)

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans` to implement task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** Wire the four GitHub Actions workflows the spec calls for — `ci.yml` (lint + offline tests on every push/PR), `deploy.yml` (lint → test → RAGAS gate → build/push → SSM-triggered deploy on push to `main`), `live-tests.yml` (manual, hits real APIs), and `ragas-ondemand.yml` (manual, full RAGAS run + PR with snapshot). Plus an operator runbook for the GitHub Secrets that all this depends on.

**Architecture:** Workflows live in `.github/workflows/`. AWS access is via OIDC role-assumption (Plan B's `github-actions-deploy` role) — no long-lived AWS access keys in GitHub. Runtime API keys (Gemini, Jina, Qdrant, LangSmith) live in GitHub Encrypted Secrets and are exported as job-level env vars only where needed. EC2 instance lookup at deploy time is by tag (`Name=research-rag`) so no instance ID hardcoded in the workflow.

**Tech Stack:** GitHub Actions, `aws-actions/configure-aws-credentials@v4` (OIDC), `aws-actions/amazon-ecr-login@v2`, `docker/setup-buildx-action@v3`, `docker/build-push-action@v6`, `actions/checkout@v4`, `actions/setup-python@v5`, `astral-sh/setup-uv@v3`, the existing `scripts/ragas_gate.py` from Plan A, the existing `src/evaluation/ragas_runner.py`.

**Reference spec:** [`docs/superpowers/specs/2026-05-21-ec2-free-tier-deployment-design.md`](../specs/2026-05-21-ec2-free-tier-deployment-design.md) Section 2 (CI/CD pipeline).

**Out of scope:**
- Actually executing the workflows end-to-end against AWS — that depends on Plan B being run in the AWS Console first.
- Modifying `src/` code beyond what Plan A already added.

**Pre-requisites for *executing this plan*:**
- Plan A merged (commits `a22638a` through `bea732f`).
- `actionlint` optional but recommended: `brew install actionlint` or `go install github.com/rhysd/actionlint/cmd/actionlint@latest`. If absent, fall back to Python YAML validation.

**Pre-requisites for the workflows actually running end-to-end:**
- Plan B's AWS resources provisioned (OIDC provider, `github-actions-deploy` role, EC2 instance with `Name=research-rag` tag, SSM Parameter Store populated).
- GitHub repo settings → Secrets and Variables → Actions populated per [Task 5](#task-5-github-secrets-runbook).

---

## File map

| Path | Status | Responsibility |
|---|---|---|
| `.github/workflows/ci.yml` | modify | Lint + offline tests on every push and PR. Currently runs all tests; switch to `-m 'not live'` and rename `check-success` job. |
| `.github/workflows/live-tests.yml` | create | `workflow_dispatch` job that runs `pytest -m live` with real API keys. |
| `.github/workflows/ragas-ondemand.yml` | create | `workflow_dispatch` job that runs the full RAGAS evaluation, snapshots the result, opens a PR with the snapshot + gate comparison. |
| `.github/workflows/deploy.yml` | create | `push: branches: [main]` workflow. Five jobs in sequence: `lint-and-test`, `ragas-gate`, `build-and-push`, `deploy`, `verify`. |
| `deploy/aws-setup/github-secrets-setup.md` | create | Operator runbook: which GitHub Secrets to set, where, and how to derive their values from Plan B's output. |

**Out of file map:** anything under `src/` (Plan A already added the gate script and live marker), anything under `evaluation/results/` (CI produces these; gitignored snapshot dirs are fine to leave as workflow artifacts).

---

## Conventions used in this plan

- **No commits during execution.** Same rule as Plans A and B. User reviews diffs and slices commits manually after the plan completes.
- **Workflow syntax: actions pinned to major versions.** `@v4`, `@v3`, etc. Major-version pins keep us on the latest patched release while preventing breaking changes.
- **Job-level env, not workflow-level.** Secrets are exported per job that needs them; jobs that don't need API keys don't see them. This keeps the blast radius small if a workflow file is ever exploited.
- **Permissions least-privilege.** Each workflow declares the minimal `permissions:` block. Default is read-only; OIDC requires `id-token: write`; commenting on commits/PRs requires `pull-requests: write` and `contents: write`.
- **No actionlint installed in CI itself** — that's overkill. Local lint during Task 6 is enough.

---

## Task 1: Switch existing `ci.yml` to offline tests + bring it in line with the spec

**Files:**
- Modify: `.github/workflows/ci.yml`

**Context:** The existing `ci.yml` runs `uv run pytest tests/ -v` — that includes live tests once anyone marks any test as `@pytest.mark.live`. The spec says CI must skip live tests by default. Also moving to `astral-sh/setup-uv@v3` for cleaner uv install, and triggering on every branch (not just `main`) so feature branches also get linted.

- [ ] **Step 1: Replace the file**

Overwrite `.github/workflows/ci.yml` with:

```yaml
name: CI

on:
  push:
    branches: [ "**" ]
  pull_request:
    branches: [ main ]

permissions:
  contents: read

jobs:
  lint-and-test:
    name: Lint + offline tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --all-groups --frozen

      - name: Ruff lint
        run: uv run ruff check .

      - name: Ruff format check
        run: uv run ruff format --check .

      - name: Pytest (offline subset)
        run: uv run pytest -m 'not live' -v
```

- [ ] **Step 2: Validate YAML parses**

Run from repo root:

```bash
uv run python -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml')); print('ok')"
```

Expected: `ok`.

- [ ] **Step 3: Validate with actionlint if available**

Run: `actionlint .github/workflows/ci.yml 2>&1 || echo "actionlint not installed — skip"`

Expected: no errors, or skip notice.

- [ ] **Step 4: Sanity-check the existing test suite still maps**

The workflow runs `pytest -m 'not live'`. Plan A confirmed this returns the same count as the full collection (no test currently uses `live`). Re-confirm:

Run: `uv run pytest -m 'not live' --collect-only -q | tail -1`

Expected: the count matches the workflow's expectation (~183 tests as of Plan A).

---

## Task 2: `live-tests.yml` — manually-triggered live-API smoke

**Files:**
- Create: `.github/workflows/live-tests.yml`

**Context:** Triggered with `workflow_dispatch` from the GitHub UI. Runs `pytest -m live` so when any test is later annotated with `@pytest.mark.live` (currently zero are), this is the way to run them in CI. Exports all the runtime API-key secrets only inside this job.

- [ ] **Step 1: Create the file**

```yaml
name: Live tests

on:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  live:
    name: Pytest -m live (hits real APIs)
    runs-on: ubuntu-latest
    env:
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
      JINA_API_KEY: ${{ secrets.JINA_API_KEY }}
      QDRANT_URL: ${{ secrets.QDRANT_URL }}
      QDRANT_API_KEY: ${{ secrets.QDRANT_API_KEY }}
      LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
      LANGSMITH_PROJECT: ${{ secrets.LANGSMITH_PROJECT }}
      LANGSMITH_TRACING: "true"
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --all-groups --frozen

      - name: Pytest (live)
        run: uv run pytest -m live -v
```

- [ ] **Step 2: Validate YAML**

Run:

```bash
uv run python -c "import yaml; yaml.safe_load(open('.github/workflows/live-tests.yml')); print('ok')"
```

Expected: `ok`.

---

## Task 3: `ragas-ondemand.yml` — manual RAGAS run + PR with snapshot

**Files:**
- Create: `.github/workflows/ragas-ondemand.yml`

**Context:** Triggered with `workflow_dispatch`. Two inputs: `experiment` (the human-friendly name like `v3.2-mmr-enabled`) and `pipeline_version` (lets you override `settings.pipeline_version` so the snapshot dir matches the iteration label). After the run completes, opens a PR adding the snapshot + a gate comparison comment against the locked baseline.

Snapshot path produced by `src/evaluation/ragas_runner.py` is `evaluation/results/{pipeline_version}/{experiment}.json`.

- [ ] **Step 1: Create the file**

```yaml
name: RAGAS — on-demand evaluation

on:
  workflow_dispatch:
    inputs:
      experiment:
        description: "Experiment name (used as snapshot filename)"
        required: true
        type: string
      pipeline_version:
        description: "Pipeline version (snapshot dir name); overrides settings.pipeline_version"
        required: true
        type: string
      prompt:
        description: "Prompt variant (default uses RAGChain default; e.g. 'v1' or 'v2')"
        required: false
        type: string
        default: ""

permissions:
  contents: write
  pull-requests: write

jobs:
  ragas:
    name: Run RAGAS + open snapshot PR
    runs-on: ubuntu-latest
    env:
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
      JINA_API_KEY: ${{ secrets.JINA_API_KEY }}
      QDRANT_URL: ${{ secrets.QDRANT_URL }}
      QDRANT_API_KEY: ${{ secrets.QDRANT_API_KEY }}
      LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
      LANGSMITH_PROJECT: ${{ secrets.LANGSMITH_PROJECT }}
      LANGSMITH_TRACING: "true"
      PIPELINE_VERSION: ${{ inputs.pipeline_version }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Set up uv
        uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true

      - name: Set up Python
        run: uv python install 3.12

      - name: Install dependencies
        run: uv sync --all-groups --frozen

      - name: Run RAGAS
        run: |
          if [ -n "${{ inputs.prompt }}" ]; then
            uv run python src/evaluation/ragas_runner.py \
              --experiment "${{ inputs.experiment }}" \
              --prompt "${{ inputs.prompt }}"
          else
            uv run python src/evaluation/ragas_runner.py \
              --experiment "${{ inputs.experiment }}"
          fi

      - name: Compute gate comparison against baseline
        id: gate
        run: |
          SNAPSHOT="evaluation/results/${{ inputs.pipeline_version }}/${{ inputs.experiment }}.json"
          ls -la "$SNAPSHOT"
          uv run python scripts/ragas_gate.py \
            --baseline evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json \
            --new "$SNAPSHOT" \
            --output /tmp/gate-summary.md \
          || true
          {
            echo 'GATE_SUMMARY<<EOF'
            cat /tmp/gate-summary.md
            echo 'EOF'
          } >> "$GITHUB_OUTPUT"

      - name: Create branch + commit snapshot
        id: branch
        run: |
          BRANCH="ragas/${{ inputs.experiment }}-$(date +%Y%m%d-%H%M%S)"
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git checkout -b "$BRANCH"
          git add evaluation/results/
          git commit -m "ragas: snapshot ${{ inputs.experiment }} on ${{ inputs.pipeline_version }}"
          git push --set-upstream origin "$BRANCH"
          echo "branch=$BRANCH" >> "$GITHUB_OUTPUT"

      - name: Open pull request
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr create \
            --base main \
            --head "${{ steps.branch.outputs.branch }}" \
            --title "RAGAS: ${{ inputs.experiment }}" \
            --body "$(cat <<'EOF'
          Automated RAGAS run from \`ragas-ondemand.yml\`.

          - Experiment: \`${{ inputs.experiment }}\`
          - Pipeline version: \`${{ inputs.pipeline_version }}\`
          - Prompt variant: \`${{ inputs.prompt }}\`

          ## Gate comparison vs baseline (\`v3.1.1-prefetch-scale-30\`)

          ${{ steps.gate.outputs.GATE_SUMMARY }}

          ---

          Workflow run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          EOF
          )"
```

- [ ] **Step 2: Validate YAML**

Run:

```bash
uv run python -c "import yaml; yaml.safe_load(open('.github/workflows/ragas-ondemand.yml')); print('ok')"
```

Expected: `ok`.

- [ ] **Step 3: actionlint**

Run: `actionlint .github/workflows/ragas-ondemand.yml 2>&1 || echo "actionlint not installed — skip"`

Expected: no errors, or skip notice. If actionlint complains about `gh pr create` with a heredoc, ignore — it doesn't understand bash heredoc context.

---

## Task 4: `deploy.yml` — the main-branch deploy pipeline

**Files:**
- Create: `.github/workflows/deploy.yml`

**Context:** Five sequential jobs on `push: main`:

1. `lint-and-test` — re-uses the offline lint+pytest combo (fast, ~1 min).
2. `ragas-gate` — runs `src/evaluation/ragas_runner.py` against the real APIs with a CI-scoped pipeline version (`ci-<sha>`), then `scripts/ragas_gate.py` compares to baseline. Blocks deploy if delta < −0.02. Posts a markdown comment on the commit with the metric table.
3. `build-and-push` — assumes OIDC role, logs into ECR, builds + pushes both api and ui images for `linux/amd64` tagged `:${{ github.sha }}` and `:latest`.
4. `deploy` — sends `aws ssm send-command` to the EC2 instance (looked up by tag `Name=research-rag`): refreshes ECR login, `docker compose pull`, `docker compose up -d`.
5. `verify` — sends a second SSM command that `curl`s `/api/health` from inside the box; fails the workflow if the health endpoint doesn't return 200.

The reverse-order `needs:` chain means a failure in any earlier job halts the rest.

- [ ] **Step 1: Create the file**

```yaml
name: Deploy

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  id-token: write       # required for OIDC role-assumption
  pull-requests: write  # used by the gate job to comment on commits/PRs

env:
  AWS_REGION: us-east-1
  ECR_REPO_API: research-rag-api
  ECR_REPO_UI: research-rag-ui
  EC2_NAME_TAG: research-rag

jobs:
  lint-and-test:
    name: Lint + offline tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
      - run: uv python install 3.12
      - run: uv sync --all-groups --frozen
      - run: uv run ruff check .
      - run: uv run ruff format --check .
      - run: uv run pytest -m 'not live' -v

  ragas-gate:
    name: RAGAS regression gate
    needs: lint-and-test
    runs-on: ubuntu-latest
    env:
      GOOGLE_API_KEY: ${{ secrets.GOOGLE_API_KEY }}
      JINA_API_KEY: ${{ secrets.JINA_API_KEY }}
      QDRANT_URL: ${{ secrets.QDRANT_URL }}
      QDRANT_API_KEY: ${{ secrets.QDRANT_API_KEY }}
      LANGSMITH_API_KEY: ${{ secrets.LANGSMITH_API_KEY }}
      LANGSMITH_PROJECT: ${{ secrets.LANGSMITH_PROJECT }}
      LANGSMITH_TRACING: "true"
      PIPELINE_VERSION: ci-${{ github.sha }}
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
        with:
          enable-cache: true
      - run: uv python install 3.12
      - run: uv sync --all-groups --frozen

      - name: Run RAGAS against live APIs
        run: |
          uv run python src/evaluation/ragas_runner.py \
            --experiment "ci-${{ github.sha }}"

      - name: Compare to baseline
        id: gate
        run: |
          SNAPSHOT="evaluation/results/ci-${{ github.sha }}/ci-${{ github.sha }}.json"
          uv run python scripts/ragas_gate.py \
            --baseline evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json \
            --new "$SNAPSHOT" \
            --output /tmp/gate-summary.md

      - name: Post gate summary as commit comment
        if: always()
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          if [ -f /tmp/gate-summary.md ]; then
            gh api \
              --method POST \
              "/repos/${{ github.repository }}/commits/${{ github.sha }}/comments" \
              -f body="$(cat /tmp/gate-summary.md)" \
              >/dev/null
          fi

  build-and-push:
    name: Build + push images to ECR
    needs: ragas-gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Set up Docker buildx
        uses: docker/setup-buildx-action@v3

      - name: Build + push api image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: deploy/Dockerfile.api
          platforms: linux/amd64
          push: true
          tags: |
            ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO_API }}:${{ github.sha }}
            ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO_API }}:latest
          cache-from: type=gha,scope=api
          cache-to: type=gha,scope=api,mode=max

      - name: Build + push ui image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: deploy/Dockerfile.ui
          platforms: linux/amd64
          push: true
          tags: |
            ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO_UI }}:${{ github.sha }}
            ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPO_UI }}:latest
          cache-from: type=gha,scope=ui
          cache-to: type=gha,scope=ui,mode=max

  deploy:
    name: Trigger deploy via SSM
    needs: build-and-push
    runs-on: ubuntu-latest
    outputs:
      instance_id: ${{ steps.find.outputs.instance_id }}
    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Find EC2 instance by tag
        id: find
        run: |
          ID=$(aws ec2 describe-instances \
            --filters "Name=tag:Name,Values=${{ env.EC2_NAME_TAG }}" "Name=instance-state-name,Values=running" \
            --query 'Reservations[0].Instances[0].InstanceId' --output text)
          if [ -z "$ID" ] || [ "$ID" = "None" ]; then
            echo "No running instance with tag ${{ env.EC2_NAME_TAG }} found." >&2
            exit 1
          fi
          echo "instance_id=$ID" >> "$GITHUB_OUTPUT"

      - name: Send deploy command via SSM
        id: ssm
        run: |
          set -euo pipefail
          ECR_REGISTRY=$(aws ssm get-parameter --name /research-rag/ecr-registry --query 'Parameter.Value' --output text)
          # Build the parameters JSON with jq so quoting/escaping is bulletproof.
          jq -n \
            --arg region "${{ env.AWS_REGION }}" \
            --arg registry "$ECR_REGISTRY" \
            '{
               commands: [
                 "set -euo pipefail",
                 "cd /opt/research-rag/deploy",
                 ("aws ecr get-login-password --region " + $region + " | docker login --username AWS --password-stdin " + $registry),
                 "docker compose pull",
                 "docker compose up -d",
                 "sleep 15",
                 "docker compose ps"
               ]
             }' > /tmp/deploy-cmd.json
          CMD_ID=$(aws ssm send-command \
            --instance-ids "${{ steps.find.outputs.instance_id }}" \
            --document-name "AWS-RunShellScript" \
            --comment "deploy ${{ github.sha }}" \
            --parameters file:///tmp/deploy-cmd.json \
            --query 'Command.CommandId' --output text)
          echo "cmd_id=$CMD_ID" >> "$GITHUB_OUTPUT"
          aws ssm wait command-executed \
            --command-id "$CMD_ID" \
            --instance-id "${{ steps.find.outputs.instance_id }}"
          STATUS=$(aws ssm get-command-invocation \
            --command-id "$CMD_ID" \
            --instance-id "${{ steps.find.outputs.instance_id }}" \
            --query 'Status' --output text)
          aws ssm get-command-invocation \
            --command-id "$CMD_ID" \
            --instance-id "${{ steps.find.outputs.instance_id }}" \
            --query 'StandardOutputContent' --output text
          [ "$STATUS" = "Success" ] || { echo "Deploy command status: $STATUS" >&2; exit 1; }

  verify:
    name: Verify /health after deploy
    needs: deploy
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Curl /api/health via SSM
        run: |
          set -euo pipefail
          jq -n '{
            commands: [
              "set -euo pipefail",
              "sleep 5",
              "curl -fsS --max-time 10 http://localhost/api/health"
            ]
          }' > /tmp/health-cmd.json
          CMD_ID=$(aws ssm send-command \
            --instance-ids "${{ needs.deploy.outputs.instance_id }}" \
            --document-name "AWS-RunShellScript" \
            --comment "healthcheck ${{ github.sha }}" \
            --parameters file:///tmp/health-cmd.json \
            --query 'Command.CommandId' --output text)
          aws ssm wait command-executed \
            --command-id "$CMD_ID" \
            --instance-id "${{ needs.deploy.outputs.instance_id }}"
          STATUS=$(aws ssm get-command-invocation \
            --command-id "$CMD_ID" \
            --instance-id "${{ needs.deploy.outputs.instance_id }}" \
            --query 'Status' --output text)
          OUT=$(aws ssm get-command-invocation \
            --command-id "$CMD_ID" \
            --instance-id "${{ needs.deploy.outputs.instance_id }}" \
            --query 'StandardOutputContent' --output text)
          echo "Health output: $OUT"
          [ "$STATUS" = "Success" ] || { echo "Health command status: $STATUS" >&2; exit 1; }
          echo "$OUT" | grep -q '"status":"ok"' || { echo "Health did not report ok" >&2; exit 1; }
```

- [ ] **Step 2: Validate YAML**

Run:

```bash
uv run python -c "import yaml; yaml.safe_load(open('.github/workflows/deploy.yml')); print('ok')"
```

Expected: `ok`.

- [ ] **Step 3: actionlint**

Run: `actionlint .github/workflows/deploy.yml 2>&1 || echo "actionlint not installed — skip"`

Expected: no errors. If actionlint warns about `secrets.AWS_ROLE_TO_ASSUME` being undefined, that's fine — secrets aren't visible to actionlint statically.

---

## Task 5: GitHub Secrets runbook

**Files:**
- Create: `deploy/aws-setup/github-secrets-setup.md`

**Context:** Documents every Encrypted Secret the workflows need, where to find the right value, and how to enter it. Lives alongside the AWS setup docs because the AWS Role ARN comes from Plan B's `04-create-github-deploy-role.sh` output.

- [ ] **Step 1: Create the file**

```markdown
# GitHub Secrets setup

The four workflows in `.github/workflows/` depend on encrypted secrets configured at **Repo → Settings → Secrets and Variables → Actions → New repository secret**. This page lists every secret and where its value comes from.

## Required secrets

| Secret name | Used by | Where the value comes from |
|---|---|---|
| `AWS_ROLE_TO_ASSUME` | `deploy.yml` (build-push, deploy, verify jobs) | Plan B Task 6 output: `IAM_ROLE_DEPLOY_ARN` printed by `04-create-github-deploy-role.sh`. Format: `arn:aws:iam::<account>:role/github-actions-deploy` |
| `GOOGLE_API_KEY` | `deploy.yml` (ragas-gate), `live-tests.yml`, `ragas-ondemand.yml` | Same value as in your local `.env` |
| `JINA_API_KEY` | same | Same value as in your local `.env` |
| `QDRANT_URL` | same | Same value as in your local `.env` |
| `QDRANT_API_KEY` | same | Same value as in your local `.env` |
| `LANGSMITH_API_KEY` | same | Same value as in your local `.env` |
| `LANGSMITH_PROJECT` | same | Same value as in your local `.env` (typically a project name like `research-rag-prod`) |

Note: `GITHUB_TOKEN` is provided automatically by Actions — you don't set it.

## Adding the secrets

1. Open the repo on GitHub.
2. **Settings** (top tab) → **Secrets and variables** (left sidebar) → **Actions**.
3. **Repository secrets** → **New repository secret**.
4. Paste name + value. Save.
5. Repeat for each secret above. (7 in total.)

## Verifying

After all secrets are set, manually trigger the lightest workflow that needs them — `live-tests.yml` — even though no live tests exist yet. The workflow should complete with `0 tests collected` rather than failing on a missing-secret error.

1. **Actions** tab → **Live tests** workflow → **Run workflow** → Run.
2. Watch the run. The "Pytest (live)" step should output `collected 0 items` and exit 5 (pytest's "no tests collected" exit code). That counts as success-equivalent for verification purposes.

If the run fails with "Secrets context cannot be referenced", a secret is missing — re-check the table above.

## Rotation

If you ever rotate an API key (e.g. compromised LangSmith key), update **both**:

1. The GitHub Secret (Settings → Secrets and variables → Actions → edit).
2. The corresponding SSM Parameter Store entry: `aws ssm put-parameter --name /research-rag/<key-name> --value '<new-value>' --type SecureString --overwrite`.

The local `.env` should also be updated. Three places, three updates — pick a moment when you can do all three.

## Out of scope

- **Environment-scoped secrets.** GitHub supports separating `dev` vs `prod` secrets by Environment. The portfolio scope is one environment (`main` → EC2), so we don't use Environments.
- **OIDC for the API-key secrets.** Gemini / Jina / Qdrant don't speak OIDC; the long-lived key is the only credential they accept. AWS is the only secret that uses OIDC.
```

---

## Task 6: Pre-flight validation

**Files:**
- (no new files — verification only)

**Context:** Run final checks before the user pushes. Confirms YAML is valid for all four workflows, no obvious actionlint regressions, and the existing CI didn't break.

- [ ] **Step 1: YAML validate all four workflows**

Run:

```bash
for f in .github/workflows/ci.yml .github/workflows/live-tests.yml .github/workflows/ragas-ondemand.yml .github/workflows/deploy.yml; do
  uv run python -c "import yaml; yaml.safe_load(open('$f'))" && echo "OK $f" || echo "FAIL $f"
done
```

Expected: four `OK` lines.

- [ ] **Step 2: actionlint sweep (if installed)**

Run:

```bash
if command -v actionlint >/dev/null 2>&1; then
  actionlint .github/workflows/*.yml
else
  echo "actionlint not installed — skip"
fi
```

Expected: no errors. If actionlint flags `secrets.AWS_ROLE_TO_ASSUME` or any other secret as undefined, ignore — actionlint can't see GitHub Secrets at parse time.

- [ ] **Step 3: Confirm offline tests + lint still pass locally**

Run:

```bash
uv run ruff check . && uv run ruff format --check . && uv run pytest -m 'not live' -q
```

Expected: ruff clean, all `not live` tests pass (~183 as of Plan A).

- [ ] **Step 4: Confirm the gate script can read the baseline that `deploy.yml` references**

Run:

```bash
test -f evaluation/results/v3.1.1-prefetch-scale-30/v3.1.1-prefetch-scale-30.json && echo "baseline present"
```

Expected: `baseline present`. (`deploy.yml`'s ragas-gate job hardcodes this path; if it goes missing, the workflow fails.)

---

## Final verification (after Plan B is run and GitHub Secrets are set)

These checks live in the same plan but require Plan B's AWS resources to be live and the secrets from Task 5 to be configured. They cannot be run from this plan's execution session.

- [ ] **A push to a feature branch triggers `ci.yml`, all checks green.**

```bash
git checkout -b ci-smoke
git commit --allow-empty -m "chore: ci smoke"
git push --set-upstream origin ci-smoke
```

Open Actions tab on GitHub. The CI run should complete green in ~2 minutes.

- [ ] **Manual `live-tests.yml` dispatch returns "0 tests collected" until a `@pytest.mark.live` test is added.**

Actions tab → Live tests → Run workflow → main → Run workflow. Wait for completion.

- [ ] **Manual `ragas-ondemand.yml` dispatch opens a PR with a snapshot.**

Actions tab → RAGAS — on-demand evaluation → Run workflow. Inputs: experiment=`smoke-test`, pipeline_version=`v3.1.1-prefetch-scale-30`. Wait ~5-10 minutes. A PR should appear with the snapshot file added under `evaluation/results/v3.1.1-prefetch-scale-30/smoke-test.json` and a gate comparison comment in the body.

Close the PR without merging (it's a verification run, not real evaluation).

- [ ] **A push to `main` triggers `deploy.yml` end-to-end.**

Merge a tiny harmless PR (e.g. README typo fix) to `main`. Watch the Actions tab. All five jobs should complete; the last `verify` job's logs should show `Health output: {"status":"ok",...}`. Then hit `https://<your-duckdns-host>/api/health` from your browser — same response.

If the `ragas-gate` job fails on this run, the deploy is correctly blocked. Investigate the gate output before retrying. (No bypass mechanism is provided on purpose — adding one is a Plan D decision, not a YOLO commit.)

---

## What this plan deliberately does NOT include

- **No matrix builds.** One Python version, one runner OS. The project is locked to Python 3.12.
- **No PR-only gate run on push to `dev`.** Only `main` triggers the gate. Feature branches just run lint+test.
- **No rollback automation.** If a deploy puts a bad image into ECR's `:latest`, the recovery is to SSM into the box and `docker compose pull --policy never && docker compose up -d` after retagging an older sha in ECR. Documented operationally; not automated.
- **No artifact uploads (logs, RAGAS JSON, image manifests).** GitHub's per-step logs and ECR's image history are sufficient for portfolio scope.
- **No release tagging.** Every push to `main` is the implicit release. No semver, no GitHub Releases.
- **No status badge in README.md.** Add later by hand if desired (`![CI](https://github.com/<owner>/research-rag/actions/workflows/ci.yml/badge.svg)`).
- **No environment protection rules on the `main` workflow.** Adding "require manual approval" via a GitHub Environment is a one-click change later if the portfolio narrative wants it; current pipeline is straight-through.
