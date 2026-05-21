# Plan B — AWS Infrastructure Provisioning

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` or `superpowers:executing-plans` to implement task-by-task. Steps use checkbox (`- [ ]`) syntax. **Many tasks require AWS Console clicks the user must do** — subagents call those out and pause.

**Goal:** Provision all AWS resources needed to host the research-rag stack on free-tier EC2, with secure secret storage, GitHub OIDC for keyless CI deploys, and SSM Session Manager for break-glass access. Outcome: an `i-…` instance reachable at `<your-host>.duckdns.org` over HTTPS, populated SSM Parameter Store, ECR repos with at least one pushed image pair, and the docker compose stack running end-to-end against the real Qdrant Cloud cluster.

**Architecture:** Idempotent bash scripts (`deploy/aws-setup/*.sh`) drive `aws` CLI calls. IAM policy documents committed as JSON under `deploy/aws-setup/policies/`. Manual Console steps (account creation, MFA setup, billing alarm, DuckDNS sign-up) are documented as their own tasks with clear verification. EC2 is bootstrapped via `aws ssm send-command` once the instance comes up — no SSH ever.

**Tech Stack:** AWS (Account, IAM, OIDC, ECR, SSM Parameter Store, EC2, VPC, EBS, CloudWatch), `aws` CLI v2, bash, Amazon Linux 2023, Docker engine, Colima on the workstation, DuckDNS.

**Reference spec:** [`docs/superpowers/specs/2026-05-21-ec2-free-tier-deployment-design.md`](../specs/2026-05-21-ec2-free-tier-deployment-design.md) — implements Section 3 (AWS Resources & IAM) plus the first-run image build called out in the Open Items list.

**Out of scope (Plan C):** GitHub Actions workflows themselves, the RAGAS gate wired into CI, automated deploys triggered by push to `main`.

**Pre-requisites:**
- Plan A merged or working tree clean of Plan A artifacts in `deploy/`.
- `aws` CLI v2 installed locally: `aws --version` reports `aws-cli/2.x.y` or higher.
- `jq` installed: `jq --version` reports a version.
- Colima running on the workstation: `colima status` shows `Running` and `docker info` works (needed for the one-time image push).
- A `.env` file at repo root populated with: `GOOGLE_API_KEY`, `JINA_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`. These will be uploaded to SSM in Task 8 (the local `.env` becomes the source of truth for the SSM bootstrap).
- GitHub repo for this project — its `<owner>/<repo>` will be hardcoded into the OIDC trust policy.

---

## File map

| Path | Status | Responsibility |
|---|---|---|
| `deploy/aws-setup/README.md` | create | Operator runbook: prereqs, order of execution, common errors |
| `deploy/aws-setup/env.sh` | create | Sourced by all scripts; exports `AWS_REGION`, `AWS_ACCOUNT_ID`, `PROJECT_PREFIX`, etc. Idempotent re-runs read the existing values. |
| `deploy/aws-setup/01-bootstrap-account.md` | create | **Manual** Console steps: account creation, MFA on root + dev IAM user, billing alarm, AWS CLI profile setup |
| `deploy/aws-setup/02-create-ecr-repos.sh` | create | Creates `research-rag-api` and `research-rag-ui` ECR repos. Idempotent. |
| `deploy/aws-setup/03-create-oidc-provider.sh` | create | Creates the GitHub Actions OIDC identity provider in IAM. Idempotent. |
| `deploy/aws-setup/04-create-github-deploy-role.sh` | create | Creates the `github-actions-deploy` IAM role with trust + permissions policies. Idempotent. |
| `deploy/aws-setup/05-create-ec2-instance-role.sh` | create | Creates the `ec2-rag-instance-role` IAM role and matching instance profile. Idempotent. |
| `deploy/aws-setup/06-put-ssm-params.sh` | create | Reads local `.env` + a few CLI args, puts each value as a SecureString or String under `/research-rag/`. Idempotent (overwrites). |
| `deploy/aws-setup/07-create-security-group.sh` | create | Creates `rag-sg` allowing inbound 80, 443 from anywhere; no SSH. Idempotent. |
| `deploy/aws-setup/08-launch-ec2.sh` | create | Launches the t2.micro instance with Amazon Linux 2023, gp3 30 GiB, attached instance profile, attached SG. Outputs the instance ID. |
| `deploy/aws-setup/09-bootstrap-ec2.sh` | create | Runs (via SSM Send-Command) on the new EC2 box: installs docker, clones the repo, runs fetch-secrets + duckdns-update, installs systemd units. |
| `deploy/aws-setup/10-first-image-push.sh` | create | Workstation-side: `docker buildx build --platform linux/amd64` for both api and ui, push to ECR. |
| `deploy/aws-setup/11-start-stack.sh` | create | SSM-sends `systemctl start research-rag.service` to the instance and tails the status. |
| `deploy/aws-setup/policies/github-deploy-trust.json` | create | OIDC trust policy for `github-actions-deploy` |
| `deploy/aws-setup/policies/github-deploy-permissions.json` | create | ECR push + SSM SendCommand permissions for `github-actions-deploy` |
| `deploy/aws-setup/policies/ec2-instance-trust.json` | create | EC2 trust policy for `ec2-rag-instance-role` |
| `deploy/aws-setup/policies/ec2-instance-permissions.json` | create | SSM read + ECR pull permissions for `ec2-rag-instance-role` |
| `.gitignore` | modify | Add `deploy/aws-setup/.cache/` (the local cache where scripts persist account ID + instance ID between runs) |

**Out of file map (covered elsewhere):** `.github/workflows/*` (Plan C), anything under `src/`, anything under `tests/`.

---

## Conventions used in this plan

- **Do NOT commit during this plan.** Same as Plan A — leave the working tree dirty, the user reviews and commits later in their preferred slices.
- **Idempotency.** Every script must be safe to re-run. Pattern: `aws … describe-X` first; if the resource exists, log and skip; else create. Never `aws … delete-X` from a script — destructive cleanup is manual.
- **State cache.** Discovered IDs (account, OIDC provider ARN, role ARNs, instance ID, public IP) are written to `deploy/aws-setup/.cache/state.env` (gitignored) so later scripts can source them.
- **Region.** `us-east-1` everywhere unless overridden by `AWS_REGION`.
- **Profile.** Scripts assume an AWS CLI profile named `research-rag` (configured in Task 1 manual steps). Override with `AWS_PROFILE` env if you prefer.
- **No SSH.** Connect to EC2 only via `aws ssm start-session --target i-…`. No key pair is ever created, no port 22 is ever opened.
- **Verification first.** Every task ends with a `aws …` describe/list call that proves the desired state exists, not just that the create call returned 0.

---

## Task 1: Manual AWS account bootstrap (Console)

**Files:**
- Create: `deploy/aws-setup/01-bootstrap-account.md`

**Context:** AWS account creation, root-account hardening, billing alarm setup, and IAM admin user creation all require Console clicks — no CLI bootstraps these on a fresh account. Capture the exact click-path so the user does the right thing.

- [ ] **Step 1: Write the runbook**

Create `deploy/aws-setup/01-bootstrap-account.md` with this content:

```markdown
# 01 — Account bootstrap (manual Console steps)

These steps require AWS Console clicks; the AWS CLI cannot bootstrap a fresh account. Total time: ~30 minutes.

## 1. Create the AWS account (skip if you already have one)

1. Go to https://signup.aws.amazon.com/signup
2. Use a real email you control. Pick an account name (e.g. "research-rag-portfolio").
3. Add a payment method. AWS will not charge if you stay in free tier, but a card is required.
4. Complete identity verification (phone call or SMS).
5. Select the **Basic Support — Free** plan.

## 2. Secure the root user

1. Sign in at https://console.aws.amazon.com/ as the root user.
2. Top-right → username → Security credentials.
3. Enable MFA (Virtual MFA app — Authy, 1Password, Aegis, etc.). Save the secret somewhere safe.
4. Delete any pre-existing root access keys (there should be none, but check).

## 3. Set a billing alarm

The spec budgets $0 for the EC2 + free-tier services. A $1 alarm is the safety net.

1. Console → top-right → Billing and Cost Management.
2. **Budgets** → Create a budget → "Use a template" → "Monthly cost budget".
3. Budget amount: `$1.00`. Email: your email. Save.
4. Also enable **Free Tier usage alerts**: Billing preferences → "Receive Free Tier usage alerts" → save your email.

## 4. Create a dev IAM user (do NOT use root for daily work)

1. Console → IAM → Users → Create user.
2. Name: `dev-admin`. Tick "Provide user access to the AWS Management Console" → "I want to create an IAM user" → set a password.
3. Permissions → Attach policies directly → `AdministratorAccess`. (We'll tighten this later if you want; for portfolio scope this is fine.)
4. After creation, click the user → **Security credentials** → **Multi-factor authentication** → Assign → Virtual MFA. Set it up with your authenticator app.
5. Same user → **Security credentials** → **Access keys** → Create access key → choose "Command Line Interface (CLI)" → tick the acknowledgement → Create.
6. **Copy the Access Key ID and Secret Access Key now** — the secret is shown only once. Save both somewhere safe.

## 5. Configure the local AWS CLI

In your terminal:

\`\`\`bash
aws configure --profile research-rag
\`\`\`

Paste the Access Key ID, Secret Access Key, set region to `us-east-1`, output to `json`.

Verify:

\`\`\`bash
aws sts get-caller-identity --profile research-rag
\`\`\`

Expected output: a JSON blob with `"Account": "<12-digit-number>"`, `"Arn": "arn:aws:iam::…:user/dev-admin"`. **Capture the 12-digit Account ID** — every later script needs it.

## 6. Set the default profile (optional)

If you only have one AWS account, you can skip `--profile research-rag` everywhere by exporting:

\`\`\`bash
export AWS_PROFILE=research-rag
export AWS_REGION=us-east-1
\`\`\`

Add those to your shell rc file (`~/.zshrc` / `~/.bashrc`) so they persist.

## Verification before moving on

\`\`\`bash
aws sts get-caller-identity
\`\`\`

The output should show `dev-admin`, not the root user. If it shows root, your CLI is still using root credentials — re-do step 5.
```

- [ ] **Step 2: Confirm completion**

Before continuing to Task 2, the user must finish all six steps above and have a working `aws sts get-caller-identity` returning the `dev-admin` ARN. Pause and ask the user to confirm. Capture the 12-digit account ID for use in Task 2.

---

## Task 2: Shared env script + .cache scaffold + .gitignore

**Files:**
- Create: `deploy/aws-setup/env.sh`
- Create: `deploy/aws-setup/.cache/.keep`
- Modify: `.gitignore`

**Context:** All later scripts source `env.sh` to pick up region, account ID, project prefix, and a couple of name constants. `.cache/` is gitignored and holds runtime-discovered IDs (instance ID, public IP, etc.) so re-runs of later scripts can find them.

- [ ] **Step 1: Create `deploy/aws-setup/env.sh`**

```bash
#!/usr/bin/env bash
# Sourced by every other script in this directory.
# Re-runs safe: cached values are read, missing ones are discovered.

set -euo pipefail

export AWS_REGION="${AWS_REGION:-us-east-1}"
export AWS_PROFILE="${AWS_PROFILE:-research-rag}"

# Project naming
export PROJECT_PREFIX="research-rag"
export ECR_REPO_API="${PROJECT_PREFIX}-api"
export ECR_REPO_UI="${PROJECT_PREFIX}-ui"
export IAM_ROLE_DEPLOY="github-actions-deploy"
export IAM_ROLE_INSTANCE="ec2-rag-instance-role"
export IAM_INSTANCE_PROFILE="ec2-rag-instance-profile"
export SG_NAME="rag-sg"
export SSM_PREFIX="/research-rag"

# Cache file
THIS_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
export AWS_SETUP_DIR="$THIS_DIR"
export AWS_SETUP_CACHE="$THIS_DIR/.cache/state.env"
mkdir -p "$(dirname "$AWS_SETUP_CACHE")"
touch "$AWS_SETUP_CACHE"

# Read previously-discovered values, if any.
# shellcheck disable=SC1090
source "$AWS_SETUP_CACHE"

# Account ID — discover once, then persist.
if [[ -z "${AWS_ACCOUNT_ID:-}" ]]; then
    AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    echo "AWS_ACCOUNT_ID=$AWS_ACCOUNT_ID" >> "$AWS_SETUP_CACHE"
fi
export AWS_ACCOUNT_ID

# Helper: persist a key=value into the cache (overwrites existing key).
cache_set() {
    local key="$1" value="$2"
    # Strip existing line, append new.
    grep -v "^${key}=" "$AWS_SETUP_CACHE" > "${AWS_SETUP_CACHE}.tmp" || true
    echo "${key}=${value}" >> "${AWS_SETUP_CACHE}.tmp"
    mv "${AWS_SETUP_CACHE}.tmp" "$AWS_SETUP_CACHE"
    export "${key}=${value}"
}

# Pretty logging
log() { printf "\033[1;34m[aws-setup]\033[0m %s\n" "$*"; }
warn() { printf "\033[1;33m[aws-setup]\033[0m %s\n" "$*" >&2; }
die() { printf "\033[1;31m[aws-setup]\033[0m %s\n" "$*" >&2; exit 1; }
```

- [ ] **Step 2: Create the cache placeholder**

Run: `mkdir -p deploy/aws-setup/.cache && touch deploy/aws-setup/.cache/.keep`

- [ ] **Step 3: Update `.gitignore`**

Edit `.gitignore`. Add at the bottom:

```
# AWS bootstrap cache (instance IDs, public IPs)
deploy/aws-setup/.cache/
!deploy/aws-setup/.cache/.keep
```

- [ ] **Step 4: Verify sourcing works**

Run from the repo root:

```bash
bash -c 'source deploy/aws-setup/env.sh && echo "account=$AWS_ACCOUNT_ID region=$AWS_REGION"'
```

Expected: `account=<12-digit-id> region=us-east-1`. If account discovery fails, the `aws sts get-caller-identity` call is failing — go back to Task 1 Step 5.

---

## Task 3: IAM policy JSON documents

**Files:**
- Create: `deploy/aws-setup/policies/github-deploy-trust.json`
- Create: `deploy/aws-setup/policies/github-deploy-permissions.json`
- Create: `deploy/aws-setup/policies/ec2-instance-trust.json`
- Create: `deploy/aws-setup/policies/ec2-instance-permissions.json`

**Context:** IAM policies are committed as static JSON files so the scripts that create the roles stay tiny. The trust policies use placeholder tokens (`__ACCOUNT_ID__`, `__GITHUB_REPO__`) that scripts substitute at apply time with `sed`.

- [ ] **Step 1: Create `policies/github-deploy-trust.json`**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::__ACCOUNT_ID__:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:__GITHUB_REPO__:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

- [ ] **Step 2: Create `policies/github-deploy-permissions.json`**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRAuthAndPush",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeRepositories",
        "ecr:DescribeImages"
      ],
      "Resource": [
        "arn:aws:ecr:__AWS_REGION__:__ACCOUNT_ID__:repository/research-rag-api",
        "arn:aws:ecr:__AWS_REGION__:__ACCOUNT_ID__:repository/research-rag-ui",
        "*"
      ]
    },
    {
      "Sid": "SSMSendCommandToInstance",
      "Effect": "Allow",
      "Action": [
        "ssm:SendCommand",
        "ssm:GetCommandInvocation",
        "ssm:ListCommandInvocations"
      ],
      "Resource": "*"
    }
  ]
}
```

Note: `ecr:GetAuthorizationToken` requires `Resource: "*"`, which is why both an arn-scoped resource and `*` appear — AWS won't accept the auth-token call against a scoped repo ARN.

- [ ] **Step 3: Create `policies/ec2-instance-trust.json`**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- [ ] **Step 4: Create `policies/ec2-instance-permissions.json`**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadResearchRagParams",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:__AWS_REGION__:__ACCOUNT_ID__:parameter/research-rag/*"
    },
    {
      "Sid": "PullECRImages",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchGetImage",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchCheckLayerAvailability"
      ],
      "Resource": "*"
    }
  ]
}
```

- [ ] **Step 5: Verify JSON parses**

Run: `for f in deploy/aws-setup/policies/*.json; do jq empty "$f" && echo "OK $f"; done`

Expected: 4 lines, all `OK …json`. If any fails, fix the JSON before continuing.

---

## Task 4: ECR repositories

**Files:**
- Create: `deploy/aws-setup/02-create-ecr-repos.sh`

**Context:** Two private ECR repos, one per Docker image. Free tier covers 500 MB of storage. Image tagging is `MUTABLE` so `:latest` works for compose pulls.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Idempotent ECR repo creation.

set -euo pipefail
source "$(dirname "$0")/env.sh"

create_repo() {
    local name="$1"
    if aws ecr describe-repositories --repository-names "$name" >/dev/null 2>&1; then
        log "ECR repo $name already exists — skipping"
    else
        log "Creating ECR repo $name"
        aws ecr create-repository \
            --repository-name "$name" \
            --image-tag-mutability MUTABLE \
            --image-scanning-configuration scanOnPush=true \
            >/dev/null
    fi
}

create_repo "$ECR_REPO_API"
create_repo "$ECR_REPO_UI"

# Persist the registry URL for later scripts.
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
cache_set ECR_REGISTRY "$ECR_REGISTRY"
log "ECR registry: $ECR_REGISTRY"
```

- [ ] **Step 2: Make executable**

Run: `chmod +x deploy/aws-setup/02-create-ecr-repos.sh`

- [ ] **Step 3: Run it**

Run: `deploy/aws-setup/02-create-ecr-repos.sh`

Expected output: `[aws-setup] Creating ECR repo research-rag-api`, same for `-ui`, then `ECR registry: <acct>.dkr.ecr.us-east-1.amazonaws.com`.

- [ ] **Step 4: Verify**

Run: `aws ecr describe-repositories --query 'repositories[].repositoryName' --output text`

Expected output (in some order): `research-rag-api research-rag-ui`.

---

## Task 5: GitHub Actions OIDC identity provider

**Files:**
- Create: `deploy/aws-setup/03-create-oidc-provider.sh`

**Context:** This is the one-time setup that lets any IAM role in this account be assumed by GitHub Actions workflows (with the right `sub` condition). Per-account, not per-repo — re-running is safe.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Idempotent OIDC provider creation for GitHub Actions.

set -euo pipefail
source "$(dirname "$0")/env.sh"

OIDC_URL="https://token.actions.githubusercontent.com"
OIDC_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"

if aws iam get-open-id-connect-provider --open-id-connect-provider-arn "$OIDC_ARN" >/dev/null 2>&1; then
    log "OIDC provider already exists — skipping"
else
    log "Creating OIDC provider for GitHub Actions"
    # The thumbprint below is GitHub's well-known one. AWS now also accepts JWKS-derived
    # thumbprints automatically; passing one is required by the API but no longer authoritative.
    aws iam create-open-id-connect-provider \
        --url "$OIDC_URL" \
        --client-id-list "sts.amazonaws.com" \
        --thumbprint-list "6938fd4d98bab03faadb97b34396831e3780aea1" \
        >/dev/null
fi

cache_set OIDC_PROVIDER_ARN "$OIDC_ARN"
log "OIDC provider ARN: $OIDC_ARN"
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/03-create-oidc-provider.sh
deploy/aws-setup/03-create-oidc-provider.sh
```

Expected: `Creating OIDC provider for GitHub Actions` then the ARN line. Re-running prints the "already exists — skipping" line.

- [ ] **Step 3: Verify**

Run: `aws iam list-open-id-connect-providers --query 'OpenIDConnectProviderList[].Arn' --output text`

Expected: includes `arn:aws:iam::<acct>:oidc-provider/token.actions.githubusercontent.com`.

---

## Task 6: `github-actions-deploy` IAM role

**Files:**
- Create: `deploy/aws-setup/04-create-github-deploy-role.sh`

**Context:** The role GitHub Actions assumes via OIDC. Allowed only from pushes to `main` of the project repo. Permissions: push to the two ECR repos + SSM SendCommand to the EC2 instance (resource scope `*` because the instance doesn't exist yet — we'll tighten in a follow-up if desired).

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Idempotent IAM role + policy for the github-actions-deploy role.
# Requires GITHUB_REPO env var (e.g. "yourname/research-rag") on first run.

set -euo pipefail
source "$(dirname "$0")/env.sh"

if [[ -z "${GITHUB_REPO:-}" && -z "${GITHUB_REPO_CACHED:-}" ]]; then
    die "Set GITHUB_REPO=owner/repo (e.g. GITHUB_REPO=yourname/research-rag) before running this script."
fi
GITHUB_REPO="${GITHUB_REPO:-$GITHUB_REPO_CACHED}"
cache_set GITHUB_REPO_CACHED "$GITHUB_REPO"

POLICY_DIR="$AWS_SETUP_DIR/policies"
TRUST=$(sed -e "s|__ACCOUNT_ID__|$AWS_ACCOUNT_ID|g" \
            -e "s|__GITHUB_REPO__|$GITHUB_REPO|g" \
            "$POLICY_DIR/github-deploy-trust.json")
PERMS=$(sed -e "s|__ACCOUNT_ID__|$AWS_ACCOUNT_ID|g" \
            -e "s|__AWS_REGION__|$AWS_REGION|g" \
            "$POLICY_DIR/github-deploy-permissions.json")

# Create or update the role.
if aws iam get-role --role-name "$IAM_ROLE_DEPLOY" >/dev/null 2>&1; then
    log "Role $IAM_ROLE_DEPLOY already exists — updating trust policy"
    aws iam update-assume-role-policy \
        --role-name "$IAM_ROLE_DEPLOY" \
        --policy-document "$TRUST"
else
    log "Creating role $IAM_ROLE_DEPLOY"
    aws iam create-role \
        --role-name "$IAM_ROLE_DEPLOY" \
        --assume-role-policy-document "$TRUST" \
        --description "Assumed by GitHub Actions via OIDC; pushes to ECR + SSM Send-Command." \
        >/dev/null
fi

# Inline policy (re-puts overwrite cleanly, no idempotency dance needed).
log "Putting inline permissions policy on $IAM_ROLE_DEPLOY"
aws iam put-role-policy \
    --role-name "$IAM_ROLE_DEPLOY" \
    --policy-name "github-actions-deploy-permissions" \
    --policy-document "$PERMS"

ROLE_ARN=$(aws iam get-role --role-name "$IAM_ROLE_DEPLOY" --query Role.Arn --output text)
cache_set IAM_ROLE_DEPLOY_ARN "$ROLE_ARN"
log "Role ARN: $ROLE_ARN"
```

- [ ] **Step 2: Make executable, set GITHUB_REPO, run**

```bash
chmod +x deploy/aws-setup/04-create-github-deploy-role.sh
GITHUB_REPO=yourname/research-rag deploy/aws-setup/04-create-github-deploy-role.sh
```

(Replace `yourname/research-rag` with the real `<owner>/<repo>`. Used only on first run — subsequent runs read it from the cache.)

Expected: `Creating role github-actions-deploy`, `Putting inline permissions policy`, then the role ARN. Save this — Plan C needs it.

- [ ] **Step 3: Verify**

Run:

```bash
aws iam get-role --role-name github-actions-deploy --query 'Role.AssumeRolePolicyDocument' --output json | jq .
```

Expected: trust policy with the substituted `repo:<owner>/research-rag:ref:refs/heads/main` value (no `__GITHUB_REPO__` placeholder).

---

## Task 7: `ec2-rag-instance-role` and instance profile

**Files:**
- Create: `deploy/aws-setup/05-create-ec2-instance-role.sh`

**Context:** Attached to the EC2 instance. Allows it to (a) be managed by SSM (Session Manager + Send-Command), (b) read `/research-rag/*` SSM params, (c) pull from ECR.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Idempotent EC2 instance role + instance profile.

set -euo pipefail
source "$(dirname "$0")/env.sh"

POLICY_DIR="$AWS_SETUP_DIR/policies"
TRUST=$(cat "$POLICY_DIR/ec2-instance-trust.json")
PERMS=$(sed -e "s|__ACCOUNT_ID__|$AWS_ACCOUNT_ID|g" \
            -e "s|__AWS_REGION__|$AWS_REGION|g" \
            "$POLICY_DIR/ec2-instance-permissions.json")

# Create role.
if aws iam get-role --role-name "$IAM_ROLE_INSTANCE" >/dev/null 2>&1; then
    log "Role $IAM_ROLE_INSTANCE already exists — refreshing trust policy"
    aws iam update-assume-role-policy \
        --role-name "$IAM_ROLE_INSTANCE" \
        --policy-document "$TRUST"
else
    log "Creating role $IAM_ROLE_INSTANCE"
    aws iam create-role \
        --role-name "$IAM_ROLE_INSTANCE" \
        --assume-role-policy-document "$TRUST" \
        --description "Attached to EC2 instance; reads SSM params and pulls ECR." \
        >/dev/null
fi

# Attach the AWS-managed SSM core policy (enables SSM agent + Session Manager).
log "Attaching AmazonSSMManagedInstanceCore"
aws iam attach-role-policy \
    --role-name "$IAM_ROLE_INSTANCE" \
    --policy-arn "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"

# Custom inline policy.
log "Putting inline policy ec2-rag-instance-permissions"
aws iam put-role-policy \
    --role-name "$IAM_ROLE_INSTANCE" \
    --policy-name "ec2-rag-instance-permissions" \
    --policy-document "$PERMS"

# Instance profile.
if aws iam get-instance-profile --instance-profile-name "$IAM_INSTANCE_PROFILE" >/dev/null 2>&1; then
    log "Instance profile $IAM_INSTANCE_PROFILE already exists"
else
    log "Creating instance profile $IAM_INSTANCE_PROFILE"
    aws iam create-instance-profile --instance-profile-name "$IAM_INSTANCE_PROFILE" >/dev/null
fi

# Add role to profile (idempotent — error if already added is silently swallowed by --no-cli-pager + retry).
if aws iam get-instance-profile --instance-profile-name "$IAM_INSTANCE_PROFILE" \
    --query 'InstanceProfile.Roles[].RoleName' --output text \
    | grep -qw "$IAM_ROLE_INSTANCE"; then
    log "Role already attached to instance profile"
else
    log "Adding role to instance profile"
    aws iam add-role-to-instance-profile \
        --instance-profile-name "$IAM_INSTANCE_PROFILE" \
        --role-name "$IAM_ROLE_INSTANCE"
fi

ROLE_ARN=$(aws iam get-role --role-name "$IAM_ROLE_INSTANCE" --query Role.Arn --output text)
cache_set IAM_ROLE_INSTANCE_ARN "$ROLE_ARN"
log "Instance role ARN: $ROLE_ARN"
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/05-create-ec2-instance-role.sh
deploy/aws-setup/05-create-ec2-instance-role.sh
```

- [ ] **Step 3: Verify**

Run:

```bash
aws iam list-attached-role-policies --role-name ec2-rag-instance-role --query 'AttachedPolicies[].PolicyArn' --output text
aws iam list-role-policies --role-name ec2-rag-instance-role --query 'PolicyNames' --output text
```

Expected first command: includes `arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore`.
Expected second command: includes `ec2-rag-instance-permissions`.

---

## Task 8: SSM Parameter Store secrets

**Files:**
- Create: `deploy/aws-setup/06-put-ssm-params.sh`

**Context:** Reads the local repo-root `.env` file (which already has the project secrets) plus a couple of CLI args for things not in `.env` (DuckDNS host/token, ECR registry), and uploads each value under `/research-rag/`. Sensitive ones become `SecureString` (KMS-encrypted with the AWS-managed `alias/aws/ssm` key, which is free). Hosts/URLs become plain `String`.

Note: ECR_REGISTRY is automatically populated from the cache (set by Task 4). DuckDNS values are taken from CLI args because the user creates those externally (Task 10) — re-run this script after Task 10 with DuckDNS flags to fill in those two keys.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Uploads runtime secrets to SSM Parameter Store under /research-rag/.
# Reads sensitive values from the repo-root .env; takes DuckDNS values as flags.

set -euo pipefail
source "$(dirname "$0")/env.sh"

DUCKDNS_HOST_ARG=""
DUCKDNS_TOKEN_ARG=""
SKIP_DUCKDNS=0

while [[ $# -gt 0 ]]; do
    case "$1" in
        --duckdns-host) DUCKDNS_HOST_ARG="$2"; shift 2;;
        --duckdns-token) DUCKDNS_TOKEN_ARG="$2"; shift 2;;
        --skip-duckdns) SKIP_DUCKDNS=1; shift;;
        *) die "Unknown arg: $1";;
    esac
done

ENV_FILE="$(cd "$AWS_SETUP_DIR/../.." && pwd)/.env"
[[ -f "$ENV_FILE" ]] || die ".env not found at $ENV_FILE"

# shellcheck disable=SC1090
source "$ENV_FILE"

# Required-from-.env keys
for key in GOOGLE_API_KEY JINA_API_KEY QDRANT_URL QDRANT_API_KEY LANGSMITH_API_KEY LANGSMITH_PROJECT; do
    if [[ -z "${!key:-}" ]]; then
        die "$key is empty in $ENV_FILE"
    fi
done

[[ -n "${ECR_REGISTRY:-}" ]] || die "ECR_REGISTRY not in cache. Run 02-create-ecr-repos.sh first."

put() {
    local name="$1" type="$2" value="$3"
    log "Putting $name ($type)"
    aws ssm put-parameter \
        --name "$name" \
        --type "$type" \
        --value "$value" \
        --overwrite \
        >/dev/null
}

put "$SSM_PREFIX/google-api-key"    SecureString "$GOOGLE_API_KEY"
put "$SSM_PREFIX/jina-api-key"      SecureString "$JINA_API_KEY"
put "$SSM_PREFIX/qdrant-url"        String       "$QDRANT_URL"
put "$SSM_PREFIX/qdrant-api-key"    SecureString "$QDRANT_API_KEY"
put "$SSM_PREFIX/langsmith-api-key" SecureString "$LANGSMITH_API_KEY"
put "$SSM_PREFIX/langsmith-project" String       "$LANGSMITH_PROJECT"
put "$SSM_PREFIX/ecr-registry"      String       "$ECR_REGISTRY"

if [[ "$SKIP_DUCKDNS" -eq 0 ]]; then
    [[ -n "$DUCKDNS_HOST_ARG" ]] || die "Pass --duckdns-host <subdomain>.duckdns.org or --skip-duckdns"
    [[ -n "$DUCKDNS_TOKEN_ARG" ]] || die "Pass --duckdns-token <token> or --skip-duckdns"
    put "$SSM_PREFIX/duckdns-host"  String       "$DUCKDNS_HOST_ARG"
    put "$SSM_PREFIX/duckdns-token" SecureString "$DUCKDNS_TOKEN_ARG"
fi

log "All parameters uploaded to $SSM_PREFIX/"
```

- [ ] **Step 2: Make executable and do the first run (without DuckDNS)**

```bash
chmod +x deploy/aws-setup/06-put-ssm-params.sh
deploy/aws-setup/06-put-ssm-params.sh --skip-duckdns
```

Expected: 7 "Putting …" lines then "All parameters uploaded". DuckDNS params will be filled in by Task 10's re-run.

- [ ] **Step 3: Verify**

Run:

```bash
aws ssm get-parameters-by-path --path /research-rag --query 'Parameters[].Name' --output text | tr '\t' '\n' | sort
```

Expected (sorted): 7 lines, paths under `/research-rag/`: `ecr-registry`, `google-api-key`, `jina-api-key`, `langsmith-api-key`, `langsmith-project`, `qdrant-api-key`, `qdrant-url`. (DuckDNS shows up after Task 10.)

---

## Task 9: Security group

**Files:**
- Create: `deploy/aws-setup/07-create-security-group.sh`

**Context:** Allows inbound 80, 443 from the world. **No port 22** — break-glass shell is via SSM Session Manager. Outbound is unrestricted.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Idempotent security group creation.

set -euo pipefail
source "$(dirname "$0")/env.sh"

# Find the default VPC for this region.
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)
[[ "$VPC_ID" != "None" && -n "$VPC_ID" ]] || die "No default VPC found in $AWS_REGION. Create one or specify a custom VPC."

# Create or find SG.
SG_ID=$(aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=$SG_NAME" "Name=vpc-id,Values=$VPC_ID" \
    --query 'SecurityGroups[0].GroupId' --output text 2>/dev/null || echo "None")

if [[ "$SG_ID" == "None" || -z "$SG_ID" ]]; then
    log "Creating security group $SG_NAME in $VPC_ID"
    SG_ID=$(aws ec2 create-security-group \
        --group-name "$SG_NAME" \
        --description "research-rag: ports 80/443 inbound, no SSH" \
        --vpc-id "$VPC_ID" \
        --query GroupId --output text)
else
    log "Security group $SG_NAME exists ($SG_ID) — ensuring rules"
fi

# Authorize ingress rules. Errors swallowed if rule already exists.
authorize() {
    local port="$1"
    aws ec2 authorize-security-group-ingress \
        --group-id "$SG_ID" \
        --protocol tcp \
        --port "$port" \
        --cidr 0.0.0.0/0 \
        2>&1 | grep -v "InvalidPermission.Duplicate" || true
}
authorize 80
authorize 443

cache_set SG_ID "$SG_ID"
cache_set VPC_ID "$VPC_ID"
log "Security group: $SG_ID (VPC $VPC_ID)"
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/07-create-security-group.sh
deploy/aws-setup/07-create-security-group.sh
```

- [ ] **Step 3: Verify**

Run:

```bash
aws ec2 describe-security-groups --group-ids "$(grep '^SG_ID=' deploy/aws-setup/.cache/state.env | cut -d= -f2)" \
    --query 'SecurityGroups[0].IpPermissions[].{Port:FromPort,Cidr:IpRanges[0].CidrIp}' --output table
```

Expected: rows for port 80 and 443, both with CIDR `0.0.0.0/0`. **No row for port 22.**

---

## Task 10: Register DuckDNS subdomain (manual)

**Files:**
- (no repo files — external action only)

**Context:** DuckDNS gives free `*.duckdns.org` subdomains pointing at any IP. Account is free, no email verification, signs in via GitHub/Google/Twitter/Reddit.

- [ ] **Step 1: Sign up + create subdomain**

1. Open https://www.duckdns.org/ in a browser.
2. Sign in with any of GitHub/Google/Twitter/Reddit.
3. In the "domains" field, pick a subdomain — e.g. `research-rag-portfolio`. Click "add domain".
4. **Copy the token** shown at the top of the page (looks like a UUID).

- [ ] **Step 2: Re-run the SSM put script with DuckDNS values**

```bash
deploy/aws-setup/06-put-ssm-params.sh \
  --duckdns-host research-rag-portfolio.duckdns.org \
  --duckdns-token <your-token-uuid>
```

(Replace the host and token with the real values.)

Expected: 9 "Putting …" lines (the 7 from before + 2 DuckDNS). Re-runs are safe because SSM `put-parameter --overwrite` is idempotent.

- [ ] **Step 3: Verify all 9 params**

Run:

```bash
aws ssm get-parameters-by-path --path /research-rag --query 'Parameters[].Name' --output text | tr '\t' '\n' | sort
```

Expected: 9 entries including `duckdns-host` and `duckdns-token`.

---

## Task 11: Launch the EC2 instance

**Files:**
- Create: `deploy/aws-setup/08-launch-ec2.sh`

**Context:** t2.micro (1 vCPU, 1 GiB) in `us-east-1` with the latest Amazon Linux 2023 AMI, 30 GiB gp3 root volume, attached `rag-sg` + `ec2-rag-instance-profile`. No key pair — only SSM access.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Launches the t2.micro EC2 instance. Idempotent — re-runs detect an existing tagged instance.

set -euo pipefail
source "$(dirname "$0")/env.sh"

[[ -n "${SG_ID:-}" ]] || die "SG_ID not in cache. Run 07-create-security-group.sh first."

# Latest Amazon Linux 2023 AMI for x86_64.
AMI_ID=$(aws ssm get-parameter \
    --name "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64" \
    --query 'Parameter.Value' --output text)
log "Using AMI: $AMI_ID"

# Look up an existing instance by tag.
INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=$PROJECT_PREFIX" "Name=instance-state-name,Values=running,pending,stopped" \
    --query 'Reservations[0].Instances[0].InstanceId' --output text 2>/dev/null || echo "None")

if [[ "$INSTANCE_ID" != "None" && -n "$INSTANCE_ID" ]]; then
    log "Instance $INSTANCE_ID already exists — skipping launch"
else
    log "Launching new t2.micro instance"
    INSTANCE_ID=$(aws ec2 run-instances \
        --image-id "$AMI_ID" \
        --instance-type t2.micro \
        --security-group-ids "$SG_ID" \
        --iam-instance-profile "Name=$IAM_INSTANCE_PROFILE" \
        --block-device-mappings 'DeviceName=/dev/xvda,Ebs={VolumeSize=30,VolumeType=gp3,DeleteOnTermination=true}' \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=$PROJECT_PREFIX}]" \
        --metadata-options 'HttpEndpoint=enabled,HttpTokens=required' \
        --count 1 \
        --query 'Instances[0].InstanceId' --output text)
fi

log "Instance ID: $INSTANCE_ID"
log "Waiting for instance to enter 'running' state (60-120s)..."
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"
log "Waiting for SSM agent to register (60-180s)..."
for i in {1..60}; do
    if aws ssm describe-instance-information \
        --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
        --query 'InstanceInformationList[0].PingStatus' --output text 2>/dev/null \
        | grep -q "Online"; then
        log "SSM agent online"
        break
    fi
    sleep 5
done

PUBLIC_IP=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

cache_set INSTANCE_ID "$INSTANCE_ID"
cache_set PUBLIC_IP "$PUBLIC_IP"
log "Instance ready: $INSTANCE_ID at $PUBLIC_IP"
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/08-launch-ec2.sh
deploy/aws-setup/08-launch-ec2.sh
```

Expected: takes 2-4 minutes. Final line: `Instance ready: i-… at <ip>`.

- [ ] **Step 3: Verify SSM connection works**

Run:

```bash
INSTANCE_ID=$(grep '^INSTANCE_ID=' deploy/aws-setup/.cache/state.env | cut -d= -f2)
aws ssm start-session --target "$INSTANCE_ID"
```

A shell should open. Inside, type `whoami` (expect `ssm-user`), then `exit`.

If the session fails with "TargetNotConnected", wait another minute for the SSM agent and retry.

---

## Task 12: Bootstrap the EC2 box (install Docker + deploy artifacts)

**Files:**
- Create: `deploy/aws-setup/09-bootstrap-ec2.sh`

**Context:** Runs Plan A's `deploy/` artifacts onto the new instance: installs Docker, drops the repo's `deploy/` tree into `/opt/research-rag/deploy/`, installs systemd units, runs `fetch-secrets.sh` once to verify SSM access. Uses `aws ssm send-command` so we never SSH.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Bootstraps the running EC2 instance: installs Docker, ships deploy/ artifacts,
# installs systemd units, runs fetch-secrets once.

set -euo pipefail
source "$(dirname "$0")/env.sh"

[[ -n "${INSTANCE_ID:-}" ]] || die "INSTANCE_ID not in cache. Run 08-launch-ec2.sh first."

REPO_ROOT="$(cd "$AWS_SETUP_DIR/../.." && pwd)"
DEPLOY_DIR="$REPO_ROOT/deploy"

# Step A: install docker + create /opt/research-rag/.
log "Installing Docker on $INSTANCE_ID"
CMD_ID=$(aws ssm send-command \
    --instance-ids "$INSTANCE_ID" \
    --document-name "AWS-RunShellScript" \
    --comment "research-rag bootstrap: install docker" \
    --parameters 'commands=[
        "set -euo pipefail",
        "dnf update -y",
        "dnf install -y docker",
        "systemctl enable --now docker",
        "usermod -aG docker ssm-user",
        "mkdir -p /opt/research-rag/deploy/systemd",
        "chown -R ssm-user:ssm-user /opt/research-rag",
        "docker --version"
    ]' \
    --query 'Command.CommandId' --output text)

log "Waiting for docker install (CommandId $CMD_ID)..."
aws ssm wait command-executed --command-id "$CMD_ID" --instance-id "$INSTANCE_ID"
aws ssm get-command-invocation --command-id "$CMD_ID" --instance-id "$INSTANCE_ID" --query 'Status' --output text

# Step B: upload deploy/ artifacts via base64-encoded heredoc through SSM.
# (S3 would be cleaner; sticking with SSM-only to avoid an extra resource.)
log "Uploading deploy/ artifacts via SSM"

upload_file() {
    local local_path="$1" remote_path="$2" mode="${3:-644}"
    local b64
    b64=$(base64 -w0 "$local_path")
    local cmd_id
    cmd_id=$(aws ssm send-command \
        --instance-ids "$INSTANCE_ID" \
        --document-name "AWS-RunShellScript" \
        --comment "upload $remote_path" \
        --parameters "commands=[\"mkdir -p \$(dirname $remote_path)\",\"echo '$b64' | base64 -d > $remote_path\",\"chmod $mode $remote_path\"]" \
        --query 'Command.CommandId' --output text)
    aws ssm wait command-executed --command-id "$cmd_id" --instance-id "$INSTANCE_ID"
    local status
    status=$(aws ssm get-command-invocation --command-id "$cmd_id" --instance-id "$INSTANCE_ID" --query 'Status' --output text)
    [[ "$status" == "Success" ]] || die "Upload of $remote_path failed: $status"
}

upload_file "$DEPLOY_DIR/Dockerfile.api"                /opt/research-rag/deploy/Dockerfile.api               644
upload_file "$DEPLOY_DIR/Dockerfile.ui"                 /opt/research-rag/deploy/Dockerfile.ui                644
upload_file "$DEPLOY_DIR/docker-compose.yml"            /opt/research-rag/deploy/docker-compose.yml           644
upload_file "$DEPLOY_DIR/Caddyfile"                     /opt/research-rag/deploy/Caddyfile                    644
upload_file "$DEPLOY_DIR/fetch-secrets.sh"              /opt/research-rag/deploy/fetch-secrets.sh             755
upload_file "$DEPLOY_DIR/duckdns-update.sh"             /opt/research-rag/deploy/duckdns-update.sh            755
upload_file "$DEPLOY_DIR/systemd/research-rag.service"          /etc/systemd/system/research-rag.service          644
upload_file "$DEPLOY_DIR/systemd/research-rag-secrets.service"  /etc/systemd/system/research-rag-secrets.service  644
upload_file "$DEPLOY_DIR/systemd/duckdns-update.service"        /etc/systemd/system/duckdns-update.service        644
upload_file "$DEPLOY_DIR/systemd/duckdns-update.timer"          /etc/systemd/system/duckdns-update.timer          644

# Step C: enable systemd units + run the secrets fetcher + first DuckDNS update.
log "Enabling systemd units and running first secrets fetch"
CMD_ID=$(aws ssm send-command \
    --instance-ids "$INSTANCE_ID" \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=[
        "systemctl daemon-reload",
        "systemctl enable research-rag-secrets.service",
        "systemctl enable duckdns-update.timer",
        "systemctl enable research-rag.service",
        "AWS_REGION='"$AWS_REGION"' /opt/research-rag/deploy/fetch-secrets.sh",
        "ls -la /opt/research-rag/.env",
        "systemctl start duckdns-update.service",
        "systemctl status duckdns-update.service --no-pager"
    ]' \
    --query 'Command.CommandId' --output text)

aws ssm wait command-executed --command-id "$CMD_ID" --instance-id "$INSTANCE_ID"
log "Bootstrap command output:"
aws ssm get-command-invocation --command-id "$CMD_ID" --instance-id "$INSTANCE_ID" \
    --query '{Status:Status,StdOut:StandardOutputContent,StdErr:StandardErrorContent}' --output table

log "EC2 box bootstrapped"
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/09-bootstrap-ec2.sh
deploy/aws-setup/09-bootstrap-ec2.sh
```

Expected: ~5 minutes. The final output table should show `Status: Success` and stdout containing the `.env` file listing (mode `-rw-------`) plus `duckdns-update: <host>.duckdns.org updated`.

- [ ] **Step 3: Verify the box is set up correctly**

Open an SSM session and check:

```bash
INSTANCE_ID=$(grep '^INSTANCE_ID=' deploy/aws-setup/.cache/state.env | cut -d= -f2)
aws ssm start-session --target "$INSTANCE_ID"
```

In the session:

```bash
sudo ls -la /opt/research-rag/deploy/
sudo ls -la /opt/research-rag/.env
sudo systemctl list-unit-files | grep research-rag
sudo systemctl list-timers --all | grep duckdns
docker --version
exit
```

Expected:
- `deploy/` contains all 6 files (Dockerfile.api, Dockerfile.ui, docker-compose.yml, Caddyfile, fetch-secrets.sh, duckdns-update.sh)
- `.env` exists at `/opt/research-rag/.env`, mode `-rw-------`, ~9 lines (one per SSM param)
- systemd units listed: `research-rag.service`, `research-rag-secrets.service`, `duckdns-update.service`, `duckdns-update.timer` — all `enabled`
- `duckdns-update.timer` in the timers list with a next fire time
- Docker installed (version reports)

---

## Task 13: First-time image build and push (workstation)

**Files:**
- Create: `deploy/aws-setup/10-first-image-push.sh`

**Context:** Workstation-side: log into ECR using the dev-admin credentials, build both images for `linux/amd64`, push. This bypasses GitHub Actions for the very first push so we can verify the EC2 stack end-to-end before Plan C automates it.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Workstation-side: build api+ui images and push to ECR.
# Requires Colima running + .env populated.

set -euo pipefail
source "$(dirname "$0")/env.sh"

REPO_ROOT="$(cd "$AWS_SETUP_DIR/../.." && pwd)"

if ! docker info >/dev/null 2>&1; then
    die "Docker daemon not reachable. Run 'colima start' first."
fi

log "Logging into ECR: $ECR_REGISTRY"
aws ecr get-login-password --region "$AWS_REGION" \
    | docker login --username AWS --password-stdin "$ECR_REGISTRY"

build_and_push() {
    local name="$1" dockerfile="$2"
    log "Building $name from $dockerfile (linux/amd64) and pushing to ECR"
    # buildx does not support --load and --push together; --push alone is enough for ECR.
    docker buildx build \
        --platform linux/amd64 \
        --file "$dockerfile" \
        --tag "$ECR_REGISTRY/$name:latest" \
        --tag "$ECR_REGISTRY/$name:$(git rev-parse --short HEAD)" \
        --push \
        "$REPO_ROOT"
    log "Pushed $ECR_REGISTRY/$name:latest"
}

build_and_push "$ECR_REPO_API" "$REPO_ROOT/deploy/Dockerfile.api"
build_and_push "$ECR_REPO_UI"  "$REPO_ROOT/deploy/Dockerfile.ui"

log "Images pushed:"
aws ecr describe-images --repository-name "$ECR_REPO_API" \
    --query 'imageDetails[].imageTags' --output text
aws ecr describe-images --repository-name "$ECR_REPO_UI" \
    --query 'imageDetails[].imageTags' --output text
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/10-first-image-push.sh
colima start                                          # if not already running
deploy/aws-setup/10-first-image-push.sh
```

Expected: ~5 minutes the first time (image downloads + builds). Final lines show the tags pushed for each repo (e.g. `latest <sha>` for both).

- [ ] **Step 3: Verify images are in ECR**

Run:

```bash
aws ecr describe-images --repository-name research-rag-api --query 'imageDetails[].{Tags:imageTags,Size:imageSizeInBytes}' --output table
aws ecr describe-images --repository-name research-rag-ui  --query 'imageDetails[].{Tags:imageTags,Size:imageSizeInBytes}' --output table
```

Expected: at least one image in each repo, each ~400-800 MB.

---

## Task 14: Start the stack on EC2

**Files:**
- Create: `deploy/aws-setup/11-start-stack.sh`

**Context:** Now that images exist in ECR and the box is bootstrapped, start `research-rag.service` and confirm everything comes up. This is the first end-to-end smoke.

- [ ] **Step 1: Create the script**

```bash
#!/usr/bin/env bash
# Starts the research-rag systemd stack via SSM and reports status.

set -euo pipefail
source "$(dirname "$0")/env.sh"

[[ -n "${INSTANCE_ID:-}" ]] || die "INSTANCE_ID not in cache."

CMD_ID=$(aws ssm send-command \
    --instance-ids "$INSTANCE_ID" \
    --document-name "AWS-RunShellScript" \
    --comment "start research-rag stack" \
    --parameters 'commands=[
        "set -euo pipefail",
        "cd /opt/research-rag/deploy",
        "aws ecr get-login-password --region '"$AWS_REGION"' | docker login --username AWS --password-stdin '"$ECR_REGISTRY"'",
        "systemctl start research-rag.service",
        "sleep 20",
        "docker compose ps",
        "curl -fsS --max-time 10 http://localhost/api/health || true",
        "systemctl status research-rag.service --no-pager | head -20"
    ]' \
    --query 'Command.CommandId' --output text)

aws ssm wait command-executed --command-id "$CMD_ID" --instance-id "$INSTANCE_ID"

log "Stack startup output:"
aws ssm get-command-invocation --command-id "$CMD_ID" --instance-id "$INSTANCE_ID" \
    --query '{Status:Status,StdOut:StandardOutputContent}' --output text
```

- [ ] **Step 2: Make executable and run**

```bash
chmod +x deploy/aws-setup/11-start-stack.sh
deploy/aws-setup/11-start-stack.sh
```

Expected output contains:
- `docker compose ps`: three rows — `caddy` (running), `api` (running, healthy), `ui` (running)
- `curl /api/health`: `{"status":"ok","pipeline_version":"v3.1.1-prefetch-scale-30"}` (or whatever current pipeline_version is)
- `systemctl status`: `Active: active`

If the `docker compose ps` rows show errors, open SSM session and run `docker compose -f /opt/research-rag/deploy/docker-compose.yml logs` to diagnose.

- [ ] **Step 3: Verify from the public internet**

Get the DuckDNS host:

```bash
DUCKDNS_HOST=$(aws ssm get-parameter --name /research-rag/duckdns-host --query 'Parameter.Value' --output text)
echo "Trying https://$DUCKDNS_HOST/api/health"
sleep 30  # let Caddy provision the cert if first boot
curl -fsS "https://$DUCKDNS_HOST/api/health"
```

Expected: `{"status":"ok","pipeline_version":"…"}` over HTTPS with a valid Let's Encrypt cert.

Open `https://<duckdns-host>/` in a browser — Streamlit page renders with green sidebar.

- [ ] **Step 4: Smoke a real query**

In the browser, ask "What is LoRA?" and verify the answer + source chunks render. (This hits the real Qdrant Cloud cluster + Gemini + Jina — uses live API quota.)

---

## Task 15: Operator README for `deploy/aws-setup/`

**Files:**
- Create: `deploy/aws-setup/README.md`

**Context:** Single file at the top of `aws-setup/` explaining order, prereqs, common errors. Someone coming back in three months should be able to read this and reconstruct the runbook.

- [ ] **Step 1: Create the README**

```markdown
# deploy/aws-setup/

One-time AWS bootstrap for the research-rag EC2 deployment. Designed to be run from a workstation; idempotent so re-runs are safe.

## What this directory does

Provisions, in order:

1. **AWS account + IAM admin user** (manual, Console) — see [01-bootstrap-account.md](01-bootstrap-account.md).
2. **ECR repos** for `research-rag-api` and `research-rag-ui` images.
3. **GitHub Actions OIDC provider** + IAM role assumable from `repo:<owner>/research-rag:ref:refs/heads/main`.
4. **EC2 instance role + instance profile** for SSM read + ECR pull.
5. **SSM Parameter Store** entries holding all runtime secrets and config.
6. **DuckDNS subdomain** (manual, https://duckdns.org) — gives the box a free HTTPS hostname.
7. **Security group** allowing only ports 80 + 443 (no SSH).
8. **EC2 instance** — t2.micro, Amazon Linux 2023, 30 GiB gp3.
9. **Bootstrap** of the instance: installs Docker, copies `deploy/` artifacts, installs systemd units.
10. **First-time image build + push** to ECR (workstation-side, via Colima).
11. **First stack start** via `systemctl start research-rag.service` over SSM.

## Prerequisites

- AWS CLI v2: `aws --version`
- jq: `jq --version`
- Colima running (for image builds): `colima status`
- `.env` file at repo root with: `GOOGLE_API_KEY`, `JINA_API_KEY`, `QDRANT_URL`, `QDRANT_API_KEY`, `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT`
- GitHub repo `<owner>/<repo>` known (used in the OIDC trust policy)

## Order of execution

```bash
# 0. Read 01-bootstrap-account.md and complete every step manually.
#    Confirm `aws sts get-caller-identity` returns the dev-admin ARN.

# 1. From the repo root, in order:
deploy/aws-setup/02-create-ecr-repos.sh
deploy/aws-setup/03-create-oidc-provider.sh
GITHUB_REPO=<owner>/research-rag deploy/aws-setup/04-create-github-deploy-role.sh
deploy/aws-setup/05-create-ec2-instance-role.sh
deploy/aws-setup/06-put-ssm-params.sh --skip-duckdns

# 2. Register a DuckDNS subdomain at https://duckdns.org; then:
deploy/aws-setup/06-put-ssm-params.sh \
  --duckdns-host <your-subdomain>.duckdns.org \
  --duckdns-token <your-duckdns-token>

# 3. Network + instance:
deploy/aws-setup/07-create-security-group.sh
deploy/aws-setup/08-launch-ec2.sh
deploy/aws-setup/09-bootstrap-ec2.sh

# 4. First image push and stack start:
deploy/aws-setup/10-first-image-push.sh
deploy/aws-setup/11-start-stack.sh

# 5. DuckDNS → EC2 IP — wait a minute, then verify HTTPS:
DUCKDNS_HOST=$(aws ssm get-parameter --name /research-rag/duckdns-host --query Parameter.Value --output text)
curl -fsS "https://$DUCKDNS_HOST/api/health"
```

Total wall-clock: 30-45 minutes from a fresh AWS account; ~10 minutes on re-runs (idempotent).

## State cache

Scripts persist discovered IDs (account, ECR registry, instance ID, public IP, etc.) to `.cache/state.env`. This is gitignored. Delete it to force re-discovery; never edit it by hand.

## Tearing down

There's no teardown script — tearing down is the user's call, since data loss matters. Manual order:

```bash
# Stop & terminate the instance (frees the t2.micro hour budget):
aws ec2 terminate-instances --instance-ids $(grep '^INSTANCE_ID=' deploy/aws-setup/.cache/state.env | cut -d= -f2)

# Optionally remove IAM roles, SG, OIDC provider, ECR repos (only if you're done with the project):
# Each requires detaching dependents first. Look up the AWS docs.
```

## Common errors

| Error | Likely cause | Fix |
|---|---|---|
| `An error occurred (InvalidClientTokenId)` | AWS CLI is using stale or wrong creds | `aws sts get-caller-identity` to confirm; rerun `aws configure` |
| `OperationNotPermitted: t2.micro is no longer available` | Free tier expired (after 12 months) | Switch to `t3.micro` (~$8/mo) or migrate off AWS |
| `TargetNotConnected` when starting SSM session | SSM agent not yet online | Wait 60s and retry; check `aws ssm describe-instance-information` |
| Caddy logs `unable to obtain certificate` | DuckDNS host hasn't propagated to the instance's public IP | Verify `dig +short <host>.duckdns.org` returns the right IP; restart `duckdns-update.service` |
| `docker compose pull` fails on EC2 | ECR login expired (12 h) | Re-run step 5 in `11-start-stack.sh`'s commands, or call `aws ecr get-login-password` again |

## Out of scope

- GitHub Actions workflows (Plan C — uses the OIDC role created here).
- Image rebuilds on every commit (Plan C — automates `10-first-image-push.sh`).
- Backups / disaster recovery. Spec accepts loss of the EC2 instance; recovery means re-running this directory.
```

---

## Final verification

After all 15 tasks are complete:

- [ ] **Stack reachable over public HTTPS**

```bash
DUCKDNS_HOST=$(aws ssm get-parameter --name /research-rag/duckdns-host --query Parameter.Value --output text)
curl -fsSI "https://$DUCKDNS_HOST/" | head -5
```

Expected: `HTTP/2 200`, `server: Caddy`, a valid Let's Encrypt cert (no `--insecure` needed).

- [ ] **Real query returns a real answer**

Open `https://<duckdns-host>/` in a browser and submit "What is LoRA?". Verify an answer renders with at least 1 source chunk.

- [ ] **Working tree contains expected aws-setup artifacts**

Run: `git status --short deploy/aws-setup/`

Expected: untracked entries for the 4 policy JSONs, 10 shell scripts, the README, the manual bootstrap markdown, and `.cache/.keep`. User reviews and decides commit boundaries.

- [ ] **AWS resources confirmed**

```bash
aws ecr describe-repositories --query 'repositories[].repositoryName' --output text
aws iam list-roles --query 'Roles[?starts_with(RoleName, `github-actions`) || starts_with(RoleName, `ec2-rag`)].RoleName' --output text
aws ssm get-parameters-by-path --path /research-rag --query 'length(Parameters)'
aws ec2 describe-instances --filters "Name=tag:Name,Values=research-rag" --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name}' --output table
```

Expected:
- 2 ECR repos
- 2 IAM roles (`github-actions-deploy`, `ec2-rag-instance-role`)
- 9 SSM parameters under `/research-rag/`
- 1 EC2 instance, state `running`

After all four checks pass, you're ready to start **Plan C (GitHub Actions CI/CD wiring + first automated deploy)**.

---

## What this plan deliberately does NOT include

- **No GitHub Actions workflows** — Plan C territory. The OIDC role exists but no workflow uses it yet.
- **No automated image rebuilds** — `10-first-image-push.sh` is a one-time manual push. Plan C automates this in `deploy.yml`.
- **No Terraform / CDK / CloudFormation** — bash + aws CLI is enough for a 1-instance free-tier deploy and matches the spec's stay-simple constraint. If you ever rebuild this in a production context, port to Terraform.
- **No CloudWatch Logs shipping** — `docker logs` + LangSmith are sufficient.
- **No backups** — nothing on the box is worth preserving; everything is reproducible from git + ECR + Qdrant Cloud.
- **No HTTPS certificate management** — Caddy handles ACME automatically; nothing for you to renew.
- **No KMS customer-managed keys** — SSM SecureString uses the AWS-managed `alias/aws/ssm` key, which is free. CMK is ~$1/month per key and unnecessary for portfolio scope.
- **No EC2 key pair.** The design spec listed one for "emergency SSH" but break-glass shell access is handled by `aws ssm start-session`, which works without any key on the instance and without port 22 open. Skipping key-pair creation removes a credential to manage. If you ever lose SSM access entirely (the agent dies, the role gets corrupted, the network goes weird), the recovery path is to terminate the instance and re-run Tasks 11+12.
