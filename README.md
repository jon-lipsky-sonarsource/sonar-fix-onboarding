# sonar-fix-onboarding

Self-service onboarding and offboarding for [sonar-fix](https://github.com/jon-lipsky-sonarsource/sonar-autofix-on-pr) — automated SonarQube AI issue fixing on pull requests.

## Onboard a repository

1. Open an [Onboard Repo](.github/ISSUE_TEMPLATE/onboard.yml) issue
2. Fill in your repository name, preferred workflow trigger, and any config overrides
3. An admin will approve your request — everything else is automatic:
   - Your SonarCloud project is found or created and bound to your repo
   - A scoped analysis token is generated and stored as a secret in your repo
   - A PR is opened in your repo with the sonar-fix workflow ready to merge
4. Merge the PR — sonar-fix is live

## Offboard a repository

1. Open an [Offboard Repo](.github/ISSUE_TEMPLATE/offboard.yml) issue
2. An admin will approve your request — everything else is automatic:
   - The SonarCloud token is revoked
   - Secrets and variables are removed from your repo
   - A PR is opened in your repo to remove the sonar-fix workflow
3. Merge the PR — sonar-fix is fully removed

---

> **Admins:** To deploy sonar-fix-onboarding in your organization, see [SETUP.md](SETUP.md).
