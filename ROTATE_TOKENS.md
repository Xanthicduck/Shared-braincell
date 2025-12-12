ROTATE_TOKENS.md

Overview
--------
This document provides a clear remediation procedure for when secrets (API tokens, keys, passwords) are accidentally committed to a repository. It covers immediate actions (rotating revoked tokens), how to remove secrets from repo history using git-filter-repo and the BFG Repo-Cleaner, notifying collaborators and CI, adding .env to .gitignore, and prevention strategies.

Immediate steps (first 15–60 minutes)
-------------------------------------
1. Identify the exposed secrets
   - Search commits and recent pushes to find files/tokens that were exposed.
   - Note every service and account the secret grants access to (GitHub tokens, cloud provider keys, third-party APIs, CI secrets).

2. Revoke and rotate the secrets immediately
   - Revoke the exposed token or key from the provider's console (GitHub, AWS, GCP, Azure, third-party API, etc.).
   - Create a new token/key with the minimum required permissions.
   - Update all systems that used the old token (CI, servers, applications, environment variables, secret stores).
   - Confirm the new token works and the old token is unusable.

3. Prevent accidental use of old tokens
   - Remove or disable the old token wherever possible (application configuration, CI variables, deployed environment variables).
   - If the token was embedded in running services, redeploy the services with updated secrets.

Removing secrets from git history
---------------------------------
Note: Rewriting history changes commit SHAs. If the repository is public or used by multiple collaborators, coordinate the rewrite carefully (see "Notifying collaborators").

Option A — git-filter-repo (recommended)
- Install: https://github.com/newren/git-filter-repo
- Example to remove a specific file or path (e.g., .env with secrets):

  git clone --mirror https://github.com/<owner>/<repo>.git
  cd <repo>.git
  git filter-repo --invert-paths --path .env

- To replace a specific secret string in all history (safer to target path when possible):

  git clone --mirror https://github.com/<owner>/<repo>.git
  cd <repo>.git
  git filter-repo --replace-text replacements.txt

  Where replacements.txt contains lines like:
  -----BEGIN REPLACEMENT MAP-----
  OLD_SECRET==>REDACTED
  -----END REPLACEMENT MAP-----

- After running git-filter-repo, force-push the rewritten history to the remote:

  git push --force --all
  git push --force --tags

Option B — BFG Repo-Cleaner (simpler for large repos)
- Install: https://rtyley.github.io/bfg-repo-cleaner/
- Example to remove a file or secrets matching a pattern:

  git clone --mirror https://github.com/<owner>/<repo>.git
  java -jar bfg.jar --delete-files .env <repo>.git
  cd <repo>.git
  git reflog expire --expire=now --all && git gc --prune=now --aggressive
  git push --force

- To replace passwords/keys with BFG's --replace-text:

  Create passwords.txt with the secrets/regex patterns to replace (one per line), then:
  java -jar bfg.jar --replace-text passwords.txt <repo>.git

Important notes on history rewriting
- All contributors will need to rebase or re-clone after the force-push. Communicate clearly (see below).
- Rewriting history does not remove secrets from forks or clones you do not control. Treat revoked tokens as compromised until rotation is completed.
- Some hosting services (e.g., GitHub) may keep caches/backups for a period; rotating credentials is required.

Notify collaborators and stakeholders
-------------------------------------
- Immediately inform the team and repository collaborators about the incident and the steps taken.
- If you will rewrite history, coordinate a time and provide clear instructions:
  - Everyone should back up local branches.
  - After the force-push, collaborators should either re-clone or follow these steps to realign:

    git fetch origin
    git checkout main
    git reset --hard origin/main

  - For feature branches, rebase onto the rewritten branch or rebase interactively onto the new main.

- Notify downstream systems and third parties that might have cached the secret.
- If the secret allowed access to customer data or production systems, follow your incident response and disclosure policies.

Updating CI/CD and other systems
--------------------------------
- Rotate secrets used by CI providers (GitHub Actions, CircleCI, Travis, GitLab CI, etc.).
- Remove exposed values from CI variables and add the new secret values.
- Redeploy services that used the secret and verify deployments are successful.

Add .env to .gitignore (example)
--------------------------------
Add the following lines to .gitignore to avoid committing local environment files:

  # Local environment files
  .env
  .env.*

If the repository uses a template env file, keep a safe example like .env.example with no real secrets.

Prevention suggestions
----------------------
1. Secret management
   - Use a secrets manager (AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault, Azure Key Vault) instead of committing secrets.
   - Store secrets in environment variables or secret stores at deploy time.

2. Limit token scope and lifetime
   - Create tokens with the minimum necessary scopes/permissions.
   - Use short-lived tokens where possible and automate rotation.

3. Automated scanning
   - Enable secret scanning on GitHub (or use tools such as TruffleHog, GitLeaks, detect-secrets) to catch secrets before they land in history.
   - Add pre-commit hooks (pre-commit framework) that run secret detection locally.

4. Protect branches and workflows
   - Enable branch protection rules, required reviews, and CI checks on protected branches.
   - Require pull requests for changes to sensitive files.

5. Developer training
   - Educate contributors about not committing secrets and how to use environment configuration safely.

6. Pre-commit & enforcement
   - Use pre-commit hooks (pre-commit.com) with secret-scanning plugins.
   - Use git hooks and/or platforms like git-secrets to prevent commits containing secrets.

Post-remediation validation
---------------------------
- Confirm that the revoked token is invalid and cannot be used.
- Verify no operational systems are broken after rotation.
- Confirm that repository history no longer contains the secret (search the rewritten mirror).
- Ensure all collaborators have re-synced to the rewritten history.

References
----------
- git-filter-repo: https://github.com/newren/git-filter-repo
- BFG Repo-Cleaner: https://rtyley.github.io/bfg-repo-cleaner/
- GitHub secret scanning and token revocation docs: https://docs.github.com/

If you want, I can also:
- Add a .env.example file with guidance for developers.
- Create a PR with the .gitignore update and .env.example on branch "-sharedbraincell" or another branch.
- Walk through rewriting history commands for this repository specifically (I will need repository remote URL and confirmation to proceed with a history rewrite).


