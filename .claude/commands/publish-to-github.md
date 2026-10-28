---
description: Publish the project to GitHub (push, README, About, Pages, CI/CD, secret scan)
argument-hint: <repo URL or owner/repo> [branch]
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, mcp__github__*
---

Publish this project to GitHub and set up its public presence.

**Repo info from the user:** `$ARGUMENTS`

If `$ARGUMENTS` is empty, ask the user for the target repo (`owner/repo` or URL), the branch to push (default: current branch), and whether the repo should be public or private. Do not guess.

Work through the steps in order. Security comes first: nothing is pushed until Step 1 passes. Report a short checklist of what was done at the end.

## Tooling

Prefer the `gh` CLI if it is installed and authenticated (`gh auth status`). If it is not available, use the `mcp__github__*` tools (load them with ToolSearch). If neither works, tell the user exactly which step could not be done and why; don't skip it silently.

## Step 1 - Make sure no sensitive data is uploaded (do this FIRST)

1. Inventory what would be published: `git ls-files` plus untracked files (`git status --porcelain`), and check the full history with `git log --all --oneline`.
2. Scan tracked files **and git history** for secrets. Use `gitleaks detect` or `trufflehog` if installed; otherwise grep for:
   - API keys/tokens/passwords: `AKIA[0-9A-Z]{16}`, `ghp_[A-Za-z0-9]{36}`, `github_pat_`, `sk-[A-Za-z0-9]{20,}`, `xox[baprs]-`, `-----BEGIN (RSA|EC|OPENSSH|PRIVATE) KEY-----`, `(api[_-]?key|secret|token|passwd|password)\s*[:=]\s*\S+`
   - Connection strings with credentials, e.g. `://user:pass@`
   - Personal data: emails, phone numbers, internal hostnames/IPs not meant to be public
3. Look for risky files: `.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `credentials*`, `*.sqlite`/`*.db`, `secrets.*`, `terraform.tfstate*`, cloud config dirs, `.claude/settings.local.json`, large data dumps.
4. Create or update `.gitignore` so these are excluded (add language-appropriate ignores too: `node_modules/`, `__pycache__/`, `dist/`, etc.). If a risky file is already tracked, untrack it with `git rm --cached <file>`.
5. Provide an `.env.example` with placeholder values if the project uses environment variables.
6. If a secret is found:
   - **Stop. Do not push.** List each finding (file, line, type) without printing the secret value itself.
   - If it is in history, tell the user it must be rotated/revoked (removing it from history is not enough once leaked) and offer to purge it (`git filter-repo`). Never rewrite history or force-push without explicit approval.
   - Move real values to environment variables / GitHub Actions secrets and reference them by name only.
7. Re-run the scan after fixes. Proceed only when clean. Tell the user what was scanned and the result.

## Step 2 - Create/update the README

1. Read the project (manifest files, source layout, existing README) to understand what it does.
2. Create `README.md`, or update the existing one while preserving accurate human-written content. Include: title and one-line description, badges (CI status, license), features, prerequisites, install, usage with examples, configuration (env var names only, never values), project structure, development/testing, contributing, license, and a link to the live GitHub Pages site (added after Step 5).
3. Only document things that exist. Verify commands you list actually work where practical.
4. If no LICENSE exists, ask the user which license they want rather than choosing one.

## Step 3 - Push the code

1. Parse `$ARGUMENTS` into `owner/repo` (accept `https://github.com/owner/repo(.git)`, `git@github.com:owner/repo.git`, or `owner/repo`) and optional branch.
2. Check whether the repo exists. If not, confirm with the user, then create it (visibility as specified).
3. Set the remote: `git remote add origin <url>` (or `git remote set-url origin <url>`). Never embed tokens in the URL.
4. Commit all changes from Steps 1-2 (and later steps) with clear messages. Don't commit anything Step 1 flagged.
5. `git push -u origin <branch>`. On network errors only, retry up to 4 times with exponential backoff (2s, 4s, 8s, 16s). If the push is rejected as non-fast-forward, stop and ask; never force-push without approval.

## Step 4 - Set up CI/CD with GitHub Actions

1. Detect the stack (package.json, pyproject.toml/requirements.txt, go.mod, Cargo.toml, pom.xml, etc.).
2. Create `.github/workflows/ci.yml` triggered on `push` and `pull_request` to the default branch. Include: checkout, language setup with dependency caching, install, lint (if configured), build, test. Pin actions to a major version (e.g. `actions/checkout@v4`) and set minimal `permissions: contents: read`.
3. If the project is deployable (package publish, container image, etc.), add a separate `cd.yml` (e.g. on tags `v*`) and only if the user wants it. Reference credentials solely via `${{ secrets.NAME }}`; tell the user which secrets to add under Settings > Secrets and variables > Actions. Never write secret values anywhere.
4. Add an optional secret-scanning job (e.g. gitleaks action) to CI.
5. Validate the YAML syntax locally before committing. After pushing, check the workflow run status and report the result; if it fails, read the logs and fix real problems.

## Step 5 - Create/update the GitHub Pages site

1. Choose the simplest fit:
   - Project already builds a static site (docs, SPA): deploy its build output.
   - Otherwise: create a `docs/` site (a single `index.html` or Markdown with Jekyll) from the README content: description, features, install/usage, link to the repo.
2. Prefer deployment via GitHub Actions: add `.github/workflows/pages.yml` using `actions/configure-pages`, `actions/upload-pages-artifact`, and `actions/deploy-pages`, with `permissions: pages: write, id-token: write, contents: read`. 
3. Enable Pages with source "GitHub Actions" (`gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`, or `PUT` if it already exists). If the API call isn't possible, tell the user to enable it under Settings > Pages.
4. Make sure no sensitive data is in the published site (re-apply Step 1 checks to the site files).
5. Push, wait for the deploy to finish, and determine the final URL (normally `https://<owner>.github.io/<repo>/`). Verify it responds with HTTP 200; Pages can take a minute or two.

## Step 6 - Create/update the About section and add the Pages link

Set these on the repo (`gh repo edit OWNER/REPO ...`, or the GitHub API `PATCH /repos/OWNER/REPO`, plus `PUT /repos/OWNER/REPO/topics`):

- **Description:** one concise sentence taken from the README (no secrets, no internal info).
- **Website (homepage):** the GitHub Pages URL from Step 5. Use `--homepage <url>`.
- **Topics:** 3-8 relevant lowercase keywords derived from the stack and purpose.

If the user already has a description or topics, keep what is accurate and only improve or add; tell them what changed.

Finally, add the live Pages link to the README (if not already there), commit, and push.

## Final report

Output a checklist with status for each step:

- [ ] Secret scan: result, what was scanned, any items needing user action (rotation etc.)
- [ ] README created/updated
- [ ] Code pushed: repo URL, branch, commit SHA
- [ ] CI/CD: workflow files and latest run status; secrets the user must add
- [ ] GitHub Pages: URL and whether it is live
- [ ] About section: description, website link, topics

Call out anything that was skipped, failed, or needs manual action. Do not open a pull request unless the user asks.
