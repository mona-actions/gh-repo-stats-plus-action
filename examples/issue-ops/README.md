# Issue Ops Gather Stats Example

This example demonstrates an [IssueOps](https://github.com/issue-ops) pattern for triggering `gh-repo-stats-plus-action` through GitHub Issues. Users open an issue using a form template, then comment `/run-stats` to kick off the stats gathering process. Results are posted back as issue comments.

## How It Works

1. A user creates a new issue using one of the provided issue templates
2. The workflow posts a welcome comment explaining how to trigger the run
3. The user comments `/run-stats` on the issue
4. The workflow parses the issue body to extract the organization/repository name and options
5. Stats are gathered using `gh-repo-stats-plus-action`
6. Results (and optional migration audit) are posted back as issue comments

## Files

```
issue-ops-gather-stats/
├── gather-stats.yml                        # Workflow file (copy to .github/workflows/)
├── issue-templates/
│   ├── 01-get-repo-stats.yml               # Issue template for single repository stats
│   └── 02-get-org-repo-stats.yml           # Issue template for organization-wide stats
└── README.md
```

### gather-stats.yml

The main workflow. Copy this to `.github/workflows/gather-stats.yml` in your repository.

It listens for:

- `issues` events (`opened`, `edited`, `reopened`) — to post the welcome comment
- `issue_comment` events (`created`) — to detect the `/run-stats` command

The workflow determines whether to run in `repository` or `organization` mode based on the issue labels (applied automatically by the issue template).

### Issue Templates

Copy the files in `issue-templates/` to `.github/ISSUE_TEMPLATE/` in your repository.

- [01-get-repo-stats.yml](issue-templates/01-get-repo-stats.yml) — Prompts for an `organization/repository` name. Applies the `stats` and `repository` labels.
- [02-get-org-repo-stats.yml](issue-templates/02-get-org-repo-stats.yml) — Prompts for an organization name. Applies the `stats` and `organization` labels.

Both templates include an optional **Migration Audit** checkbox.

## Setup

1. Copy `gather-stats.yml` to `.github/workflows/` in your target repository
2. Copy the issue templates to `.github/ISSUE_TEMPLATE/`
3. Configure the required secrets:
   - `ACCESS_TOKEN` — A PAT with `repo` scope, **or**
   - `GITHUB_APP_ID` and `GITHUB_APP_PRIVATE_KEY` — GitHub App credentials
4. Create the `stats`, `repository`, and `organization` labels in your repository if they don't already exist

## Required Permissions

The workflow needs the following permissions:

```yaml
permissions:
  contents: read
  issues: write
```
