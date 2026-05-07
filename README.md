# sonar-fix-onboarding

Issue-ops self-service onboarding and offboarding for [sonar-fix](https://github.com/jon-lipsky-sonarsource/sonar-autofix-on-pr) — automated SonarQube AI issue fixing on pull requests.

## How it works

### Onboarding

1. A developer opens an [Onboard Repo](../../issues/new?template=onboard.yml) issue
2. An admin approves it by adding the `approved` label
3. The onboarding workflow automatically:
   - Generates a **scoped SonarCloud project analysis token** (least-privilege — valid only for that project)
   - Sets `SONAR_TOKEN` as a secret in the target repo
   - Sets `SONAR_PROJECT_KEY` and `SONAR_HOST_URL` as repo variables
   - Opens a **PR** in the target repo with the sonar-fix caller workflow and config
4. The developer merges the PR — sonar-fix is live

### Offboarding

1. A developer opens an [Offboard Repo](../../issues/new?template=offboard.yml) issue
2. An admin approves it by adding the `approved` label
3. The offboarding workflow automatically:
   - **Revokes** the scoped SonarCloud token
   - Removes `SONAR_TOKEN`, `SONAR_PROJECT_KEY`, and `SONAR_HOST_URL` from the target repo
   - Opens a **PR** in the target repo removing the sonar-fix workflow and config
4. The developer merges the PR — sonar-fix is fully removed

---

## Admin setup guide

Follow these steps once when deploying this repo in your organization.

### Prerequisites

- GitHub CLI (`gh`) installed and authenticated as an org admin
- A SonarCloud account with org admin access
- The [sonar-fix](https://github.com/jon-lipsky-sonarsource/sonar-autofix-on-pr) central workflow repo already deployed in your org

### Step 1 — Fork or clone this repo into your org

```bash
gh repo create YOUR-ORG/sonar-fix-onboarding \
  --public \
  --description "Issue-ops self-service onboarding for sonar-fix" \
  --clone
```

Push the contents of this repo to it, or use GitHub's fork flow.

### Step 2 — Create issue labels

```bash
REPO="YOUR-ORG/sonar-fix-onboarding"

gh label create "onboarding"       --repo "$REPO" --color "0075ca" --description "Repo onboarding request"
gh label create "offboarding"      --repo "$REPO" --color "e4e669" --description "Repo offboarding request"
gh label create "pending-approval" --repo "$REPO" --color "d93f0b" --description "Awaiting admin approval"
gh label create "approved"         --repo "$REPO" --color "0e8a16" --description "Approved — triggers onboarding/offboarding workflow"
gh label create "onboarded"        --repo "$REPO" --color "006b75" --description "Repo successfully onboarded"
gh label create "offboarded"       --repo "$REPO" --color "5319e7" --description "Repo successfully offboarded"
```

### Step 3 — Create a SonarCloud admin token

In SonarCloud: **My Account → Security → Generate Token**

- Name: `sonar-fix-onboarding`
- Type: **User Token** (needs org-level permission to create and revoke project analysis tokens)

Copy the token value — you'll need it in the next step.

### Step 4 — Create a GitHub PAT for onboarding

Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens** and create a token with:

- **Resource owner**: your org
- **Repository access**: All repositories (or select the repos you want to allow onboarding for)
- **Permissions**:
  - `Actions variables` — Read and write
  - `Secrets` — Read and write
  - `Contents` — Read and write (to push the onboarding branch)
  - `Pull requests` — Read and write (to open the onboarding PR)

> For `AGENT_PUSH_TOKEN` (used by sonar-fix itself to push AI fix commits), create a separate fine-grained PAT with `Contents` and `Pull requests` write access across all onboardable repos.

### Step 5 — Set secrets and variables in this repo

```bash
REPO="YOUR-ORG/sonar-fix-onboarding"

# Required secrets
gh secret set SONAR_ADMIN_TOKEN --repo "$REPO"   # paste the SonarCloud token from Step 3
gh secret set ONBOARDING_PAT    --repo "$REPO"   # paste the GitHub PAT from Step 4

# Required variables
gh variable set SONAR_HOST_URL     --repo "$REPO" --body "https://sonarcloud.io"
# For SonarQube Server:            --body "https://your-sonarqube.example.com"

gh variable set SONAR_ORGANIZATION --repo "$REPO" --body "your-sonarcloud-org-slug"
# The SonarCloud organization key (visible in SonarCloud → Organization → Settings)

gh variable set GITHUB_ALM_SETTING --repo "$REPO" --body "GitHub"
# The name of the GitHub ALM integration configured in SonarCloud
# (SonarCloud → Organization → Settings → DevOps Platform Integrations → name field)

gh variable set SONAR_FIX_AGENT    --repo "$REPO" --body "claude"
# Which AI agent to use for all onboarded repos: claude | codex | copilot
# Optional — defaults to "copilot" if not set
```

### Step 6 — Set AI agent secrets at the org level

The agent API keys are shared across all onboarded repos. Set whichever agents your org supports:

```bash
ORG="YOUR-ORG"

gh secret set ANTHROPIC_API_KEY --org "$ORG" --visibility all   # Claude
gh secret set OPENAI_API_KEY    --org "$ORG" --visibility all   # Codex
gh secret set COPILOT_PAT       --org "$ORG" --visibility all   # GitHub Copilot
```

Also set the push token so that AI fix commits trigger downstream CI:

```bash
gh secret set AGENT_PUSH_TOKEN --org "$ORG" --visibility all
```

### Step 7 — Configure GitHub Actions permissions

In this repo: **Settings → Actions → General**

- Set **Actions permissions** to allow all actions (or at minimum allow actions from `anthropics`, `openai`, and `SonarSource`)
- Under **Workflow permissions**, enable **Read and write permissions**

In the **sonar-fix central workflow repo**: **Settings → Actions → General**

- Enable **"Allow other repositories in the org to call this workflow"** so onboarded repos can use the reusable workflow

### Step 8 — Update the sonar-fix repo reference in the onboarding script

Open `scripts/open-onboarding-pr.sh` and confirm the `SONAR_FIX_RAW` URL points to your org's copy of the sonar-fix repo:

```bash
SONAR_FIX_RAW="https://raw.githubusercontent.com/YOUR-ORG/sonar-fix/main/examples/${WORKFLOW_TEMPLATE}"
```

### Step 9 — Verify end-to-end

1. Open a test [Onboard Repo](../../issues/new?template=onboard.yml) issue for a repo that already has SonarCloud configured
2. Add the `approved` label
3. Watch the **Actions** tab — the onboarding workflow should run and post back to the issue
4. Confirm the PR appears in the target repo with the correct workflow and secrets set

---

## Repository structure

```
.github/
  ISSUE_TEMPLATE/
    onboard.yml          # Onboarding request form
    offboard.yml         # Offboarding request form
  workflows/
    onboard.yml          # Triggered on 'approved' label on onboarding issues
    offboard.yml         # Triggered on 'approved' label on offboarding issues

scripts/
  open-onboarding-pr.sh  # Clones target repo, writes workflow + config, opens PR
  open-offboarding-pr.sh # Clones target repo, removes workflow + config, opens PR
```

## Issue labels

| Label | Applied by | Meaning |
|---|---|---|
| `onboarding` | Issue template | Marks issue as an onboarding request |
| `offboarding` | Issue template | Marks issue as an offboarding request |
| `pending-approval` | Issue template | Awaiting admin approval |
| `approved` | Admin (manual) | Triggers the onboarding or offboarding workflow |
| `onboarded` | Workflow | Repo successfully onboarded |
| `offboarded` | Workflow | Repo successfully offboarded |

## Security notes

- `SONAR_ADMIN_TOKEN` is only used to generate and revoke **project-scoped** tokens. Each target repo receives a token valid only for its own SonarCloud project.
- `ONBOARDING_PAT` is used only to set secrets/variables and open PRs — it never reads existing secret values.
- Neither token is ever echoed or logged; the SonarCloud token is written to a temp file (`/tmp/sonar_token.txt`) and deleted immediately after the `gh secret set` call.
