# Test

A Claude Code project that provides the `/publish-to-github` custom slash command, which automates publishing a project to GitHub.

[![CI](https://github.com/chanchowcheng-create/Test/actions/workflows/ci.yml/badge.svg)](https://github.com/chanchowcheng-create/Test/actions/workflows/ci.yml)

**Live site:** https://chanchowcheng-create.github.io/Test/

## Features

The `/publish-to-github` command works through these steps:

1. Scans files and git history for secrets before anything is pushed
2. Creates or updates the README
3. Pushes the code to the GitHub repo you specify
4. Sets up CI/CD with GitHub Actions
5. Creates or updates a GitHub Pages site
6. Updates the repo About section (description, topics, and the Pages link)

## Prerequisites

- [Claude Code](https://claude.com/claude-code)
- A GitHub repository you can push to
- Optional: the `gh` CLI, authenticated, for repo settings and Pages

## Usage

From this project in Claude Code:

```
/publish-to-github owner/repo [branch]
```

## Project structure

```
.claude/commands/publish-to-github.md   The custom command definition
.github/workflows/ci.yml                CI: validates files and scans for secrets
.github/workflows/pages.yml             Deploys docs/ to GitHub Pages
docs/index.html                         GitHub Pages site
```

## Configuration

No environment variables are required.

## License

No license has been chosen yet.
