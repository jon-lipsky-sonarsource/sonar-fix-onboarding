# Admin Setup Guide

Follow these steps once when deploying sonar-fix-onboarding in your organization.

## Prerequisites

- GitHub CLI (`gh`) installed and authenticated as an org admin
- A SonarCloud account with org admin access
- The [sonar-fix](https://github.com/jon-lipsky-sonarsource/sonar-autofix-on-pr) central workflow repo already deployed in your org

## Step 1 — Fork or clone this repo into your org

```bash
gh repo create YOUR-ORG/sonar-fix-onboarding \
  --public \
  --description "Issue-ops self-service onboarding for sonar-fix" \
  --clone
```

Push the contents of this repo to it, or use GitHub's fork flow.

## Step 2 — Create issue labels

```bash
REPO="YOUR-ORG/sonar-fix-onboarding"

gh label create "onboarding"       --repo "$REPO" --color "0075ca" --description "Repo onboarding request"
gh label create "offboarding"      --repo "$REPO" --color "e4e669" --description "Repo offboarding request"
gh label create "pending-approval" --repo "$REPO" --color "d93f0b" --description "Awaiting admin approval"
gh label create "approved"         --repo "$REPO" --color "0e8a16" --description "Approved — triggers onboarding/offboarding workflow"
gh label create "onboarded"        --repo "$REPO" --color "006b75" --description "Repo successfully onboarded"
gh label create "offboarded"       --repo "$REPO" --color "5319e7" --description "Repo successfully offboarded"
```

## Step 3 — Create a SonarCloud admin token

In SonarCloud: **My Account → Security → Generate Token**

- Name: `sonar-fix-onboarding`
- Type: **User Token** (needs org-level permission to create and revoke project analysis tokens)

Copy the token value — you'll need it in Step 5.

## Step 4 — Create a GitHub PAT for onboarding

Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens** and create a token with:

- **Resource owner**: your org
- **Repository access**: All repositories (or the specific repos you want to allow onboarding for)
- **Permissions**:
  - `Actions variables` — Read and write
  - `Secrets` — Read and write
  - `Contents` — Read and write
  - `Pull requests` — Read and write

> For `AGENT_PUSH_TOKEN` (used by sonar-fix itself to push AI fix commits back to PR branches so that CI re-triggers), create a separate fine-grained PAT with `Contents` and `Pull requests` write access across all onboardable repos. Set it as an org-level secret.

## Step 5 — Set secrets and variables

```bash
REPO="YOUR-ORG/sonar-fix-onboarding"

# Secrets
gh secret set SONAR_ADMIN_TOKEN --repo "$REPO"   # SonarCloud token from Step 3
gh secret set ONBOARDING_PAT    --repo "$REPO"   # GitHub PAT from Step 4

# Variables
gh variable set SONAR_HOST_URL     --repo "$REPO" --body "https://sonarcloud.io"
# For SonarQube Server:            --body "https://your-sonarqube.example.com"

gh variable set SONAR_ORGANIZATION --repo "$REPO" --body "your-sonarcloud-org-slug"
# Found in SonarCloud → Organization → Settings

gh variable set GITHUB_ALM_SETTING --repo "$REPO" --body "GitHub"
# The name of the GitHub integration in SonarCloud → Organization → Settings → DevOps Platform Integrations

gh variable set SONAR_FIX_AGENT    --repo "$REPO" --body "copilot"
# Which AI agent to use for all onboarded repos: copilot | claude | codex
# Optional — defaults to "copilot" if not set
```

## Step 6 — Set AI agent secrets at the org level

```bash
ORG="YOUR-ORG"

gh secret set ANTHROPIC_API_KEY --org "$ORG" --visibility all   # Claude
gh secret set OPENAI_API_KEY    --org "$ORG" --visibility all   # Codex
gh secret set AGENT_PUSH_TOKEN  --org "$ORG" --visibility all   # All agents (enables CI re-trigger after fix)
# Copilot uses its own GitHub App identity — no API key needed
```

Set only the secrets for the agents your org supports.

## Step 7 — Configure GitHub Actions permissions

In the sonar-fix-onboarding repo: **Settings → Actions → General**
- Set **Workflow permissions** to **Read and write**

In the sonar-fix central workflow repo: **Settings → Actions → General**
- Enable **"Allow other repositories in the org to call this workflow"**

## Step 8 — Update the sonar-fix repo reference

Open `scripts/open-onboarding-pr.sh` and confirm the `SONAR_FIX_RAW` URL points to your org's copy of the sonar-fix repo:

```bash
SONAR_FIX_RAW="https://raw.githubusercontent.com/YOUR-ORG/sonar-fix/main/examples/${WORKFLOW_TEMPLATE}"
```

## Step 9 — Verify end-to-end

1. Open a test [Onboard Repo](.github/ISSUE_TEMPLATE/onboard.yml) issue for a repo in your org
2. Add the `approved` label
3. Watch the **Actions** tab — the onboarding workflow should run and post back to the issue with a link to the PR it opened in the target repo
