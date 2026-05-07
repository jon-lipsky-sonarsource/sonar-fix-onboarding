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

## Setup (one-time, admin)

### 1. Secrets and variables in this repo

| Name | Type | Description |
|---|---|---|
| `SONAR_ADMIN_TOKEN` | Secret | SonarCloud token with org-level admin permissions (to generate and revoke project tokens) |
| `ONBOARDING_PAT` | Secret | GitHub PAT with `repo` scope on all target repos (to set secrets/variables and open PRs) |
| `SONAR_HOST_URL` | Variable | Your SonarCloud/SonarQube base URL (e.g. `https://sonarcloud.io`) |

### 2. AI agent secrets (org-level)

The agent API keys live at the **org level**, not per-repo. Set whichever agents you support:

| Secret | Agent |
|---|---|
| `ANTHROPIC_API_KEY` | Claude |
| `OPENAI_API_KEY` | Codex |
| `COPILOT_PAT` | GitHub Copilot |

### 3. Optional: `AGENT_PUSH_TOKEN` (org-level)

sonar-fix needs a PAT to push fix commits back to PR branches so that downstream workflows (e.g. CI) re-trigger. Set `AGENT_PUSH_TOKEN` at the org level with `repo` scope. Without it, fixes still land but the post-fix build won't auto-trigger.

### 4. GitHub Actions permissions

In this repo's **Settings → Actions → General**:
- Enable "Allow other repositories in the org" (so the reusable `sonar-fix` workflow is callable)
- The `ONBOARDING_PAT` must have write access to each target repo

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

| Label | Meaning |
|---|---|
| `onboarding` | Auto-applied to onboarding requests |
| `offboarding` | Auto-applied to offboarding requests |
| `pending-approval` | Auto-applied on issue open |
| `approved` | Admin adds this to trigger the workflow |
| `onboarded` | Added by workflow on successful onboarding |
| `offboarded` | Added by workflow on successful offboarding |

Create these labels in **Settings → Labels** before using the repo.

## Security notes

- The `SONAR_ADMIN_TOKEN` is only used to generate/revoke **project-scoped** tokens. Each target repo's token can only access its own project.
- The `ONBOARDING_PAT` is used only to set secrets/variables and open PRs — it never reads existing secrets.
- Both tokens are only ever present in the onboarding workflow's environment and are never logged.
