# gh-repo-stats-plus-action

A GitHub Action wrapper for [gh-repo-stats-plus](https://github.com/mona-actions/gh-repo-stats-plus) that allows you to gather repository statistics directly from your GitHub workflows. When targeting a single repository, the action also generates a markdown summary of the collected stats. Optionally, a migration audit can be run alongside stats collection.

> **Note:** This action is compatible with GitHub Enterprise (GHE) environments. To support GHE usage, all third-party actions referenced internally are pinned to specific commit SHAs to help avoid caching issues that can occur when resolving tags.

## Authentication

You must provide credentials using **one** of the following methods:

### Personal Access Token (PAT)

Provide a PAT via the `access-token` input. The token needs `repo` scope (or appropriate permissions for the target repositories).

### GitHub App

Provide both `github-app-id` and `github-app-private-key`. The action will automatically generate an installation token scoped to the target organization using [actions/create-github-app-token](https://github.com/actions/create-github-app-token).

In both cases, the `github-token` input (typically `${{ secrets.GITHUB_TOKEN }}`) is always required for runner-level operations. If you are running in a GHE environment and need to download dependencies from github.com, you can also provide `ghec-token`.

## Inputs

| Input                    | Description                                                                                  | Required | Default                  |
| ------------------------ | -------------------------------------------------------------------------------------------- | -------- | ------------------------ |
| `type`                   | Type of stats gathering: `repository` or `organization`                                      | No       | `repository`             |
| `github-token`           | GitHub token for authentication (e.g., `github.token`)                                       | **Yes**  |                          |
| `ghec-token`             | GitHub Enterprise Cloud token (used to download dependencies from GHEC if not on github.com) | No       | `""`                     |
| `access-token`           | Personal access token with repo access for gathering stats                                   | No       | `""`                     |
| `github-app-id`          | GitHub App ID for authentication (requires `github-app-private-key`)                         | No       | `""`                     |
| `github-app-private-key` | GitHub App private key for authentication (requires `github-app-id`)                         | No       | `""`                     |
| `organization`           | Organization or owner name                                                                   | **Yes**  |                          |
| `repository`             | Repository name (required if `type` is `repository`)                                         | No       | `""`                     |
| `output-dir`             | Directory where output files will be stored                                                  | No       | `output`                 |
| `run-migration-audit`    | Whether to run migration audit (`true`/`false`)                                              | No       | `false`                  |
| `node-version`           | Node.js version to use                                                                       | No       | `25`                     |
| `base-url`               | GitHub API base URL                                                                          | No       | `https://api.github.com` |
| `retention-days`         | Number of days to retain uploaded artifacts                                                  | No       | `7`                      |

## Outputs

| Output            | Description                                      |
| ----------------- | ------------------------------------------------ |
| `output-dir`      | Directory where output files are stored          |
| `organization`    | Organization name                                |
| `repository`      | Repository name                                  |
| `stats-file`      | Path to the generated stats markdown file        |
| `audit-file`      | Path to the generated audit markdown file        |
| `migration-audit` | Whether migration audit was run (`true`/`false`) |
| `artifact-id`     | ID of the uploaded artifact containing the stats |

## Usage

### Single Repository

```yaml
name: Repo Stats

on:
  schedule:
    - cron: "0 0 * * 1" # weekly on Monday
  workflow_dispatch:

permissions:
  contents: read

jobs:
  stats:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Gather Repository Stats
        id: stats
        uses: mona-actions/gh-repo-stats-plus-action@v1
        with:
          github-token: ${{ github.token }}
          access-token: ${{ secrets.ACCESS_TOKEN }}
          organization: my-org
          repository: my-repo
          run-migration-audit: "true"

      - name: Print stats file path
        run: echo "Stats file: ${{ steps.stats.outputs.stats-file }}"
```

### Organization

```yaml
name: Org Stats

on:
  schedule:
    - cron: "0 0 * * 1"
  workflow_dispatch:

permissions:
  contents: read

jobs:
  stats:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Gather Organization Stats
        id: stats
        uses: mona-actions/gh-repo-stats-plus-action@v1
        with:
          type: organization
          github-token: ${{ github.token }}
          access-token: ${{ secrets.ACCESS_TOKEN }}
          organization: my-org

      - name: Print artifact ID
        run: echo "Artifact ID: ${{ steps.stats.outputs.artifact-id }}"
```

### Using a GitHub App for Authentication

```yaml
- name: Gather Repository Stats
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    github-token: ${{ github.token }}
    github-app-id: ${{ secrets.APP_ID }}
    github-app-private-key: ${{ secrets.APP_PRIVATE_KEY }}
    organization: my-org
    repository: my-repo
```

### GitHub Enterprise

To use with a GitHub Enterprise instance, set the `base-url` input to your GHE API endpoint. If your GHE instance cannot reach github.com to download CLI extensions, provide `ghec-token` as well.

```yaml
- name: Gather Repository Stats (GHE)
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    github-token: ${{ github.token }}
    ghec-token: ${{ secrets.GHEC_TOKEN }}
    access-token: ${{ secrets.ACCESS_TOKEN }}
    organization: my-org
    repository: my-repo
    base-url: https://github.example.com/api/v3
```

## Examples

The [examples/](examples/) directory contains complete workflow files and setups you can use as a starting point:

- [**Repository Stats**](examples/repository-stats.yml) — A simple workflow that gathers stats for a single repository on a weekly schedule, with optional migration audit.
- [**Organization Stats**](examples/organization-stats.yml) — A simple workflow that gathers stats for all repositories in an organization on a weekly schedule.
- [**Issue Ops**](examples/issue-ops/) — A full [IssueOps](https://github.com/issue-ops) example that triggers stats gathering via `/run-stats` comments on GitHub Issues. Includes issue form templates for single-repository and organization-wide stats, parses issue body and labels to determine the run type, posts results back as issue comments, and supports optional migration audits.

## Development

### File Structure

```
gh-repo-stats-plus-action/
├── action.yml                  # Action metadata (required for Marketplace)
├── .github/
│   ├── pull_request_template.md
│   ├── release-drafter.yml     # Release Drafter configuration
│   └── workflows/
│       ├── ci.yml              # Lint: ShellCheck, yamllint
│       ├── integration-test.yml # Manual end-to-end action test
│       └── release-drafter.yml # Drafts releases on merge to main
├── examples/
│   ├── repository-stats.yml    # Simple single-repo example
│   ├── organization-stats.yml  # Simple org-wide example
│   └── issue-ops/              # Full IssueOps example
│       ├── gather-stats.yml
│       ├── issue-templates/
│       │   ├── 01-get-repo-stats.yml
│       │   └── 02-get-org-repo-stats.yml
│       └── README.md
├── CODEOWNERS
├── CONTRIBUTING.md
├── .gitignore
├── LICENSE
└── README.md
```

### Releases

This project uses [Release Drafter](https://github.com/release-drafter/release-drafter) for automated release management. When pull requests are merged to `main`, a draft release is automatically created or updated with generated release notes.

1. **Label your PRs** to control version bumps:
   - `major` or `breaking` → major version bump
   - `minor`, `enhancement`, or `feature` → minor version bump
   - `patch`, `bug`, `fix`, `chore`, `maintenance`, `dependencies` → patch version bump (default)
2. **Review the draft release** on the [Releases page](https://github.com/mona-actions/gh-repo-stats-plus-action/releases)
3. **Publish** when ready — check **"Publish this Action to the GitHub Marketplace"**
4. **Update the major version tag** after publishing:

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

See [LICENSE](LICENSE) for details.
