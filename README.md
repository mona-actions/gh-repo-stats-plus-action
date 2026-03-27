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

| Input                    | Description                                                                                                                                                             | Required | Default                  |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------- | ------------------------ |
| `type`                   | Type of stats gathering: `repository`, `organization`, `project-stats`, `app-install-stats`, or `combine`                                                               | No       | `repository`             |
| `github-token`           | GitHub token for authentication (e.g., `github.token`)                                                                                                                  | **Yes**  |                          |
| `ghec-token`             | GitHub Enterprise Cloud token (used to download dependencies from GHEC if not on github.com)                                                                            | No       | `""`                     |
| `access-token`           | Personal access token with repo access for gathering stats                                                                                                              | No       | `""`                     |
| `github-app-id`          | GitHub App ID for authentication (requires `github-app-private-key`)                                                                                                    | No       | `""`                     |
| `github-app-private-key` | GitHub App private key for authentication (requires `github-app-id`)                                                                                                    | No       | `""`                     |
| `organization`           | Organization or owner name                                                                                                                                              | **Yes**  |                          |
| `repository`             | Repository name (required if `type` is `repository`)                                                                                                                    | No       | `""`                     |
| `output-dir`             | Directory where output files will be stored                                                                                                                             | No       | `output`                 |
| `run-migration-audit`    | Whether to run migration audit (`true`/`false`)                                                                                                                         | No       | `false`                  |
| `node-version`           | Node.js version to use                                                                                                                                                  | No       | `25`                     |
| `base-url`               | GitHub API base URL                                                                                                                                                     | No       | `https://api.github.com` |
| `skip-tls-verification`  | Skip TLS certificate verification for the target GitHub instance (use for GHES with self-signed certs or IP-based access)                                               | No       | `false`                  |
| `retention-days`         | Number of days to retain uploaded artifacts                                                                                                                             | No       | `7`                      |
| `batch-size`             | Number of repositories per batch (enables batch processing for large organizations). **Cannot be combined with `repository`** — batch mode generates its own repo list. | No       | `""`                     |
| `batch-index`            | Zero-based batch index (used with `batch-size` for parallel matrix jobs)                                                                                                | No       | `""`                     |
| `batch-delay`            | Delay in seconds multiplied by batch index to stagger API requests and avoid rate limits                                                                                | No       | `""`                     |
| `resume-from-last-save`  | Resume from the last saved state. Auto-enabled when re-running failed jobs (`run_attempt > 1`). See [Resume Failed Runs](#resume-failed-runs).                          | No       | `false`                  |
| `resume-run-id`          | Workflow run ID to download state from (for cross-run resume). Defaults to the current run ID.                                                                          | No       | `""`                     |

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

### Batch Processing (Organization)

For large organizations, use batch processing with GitHub Actions matrix strategy to parallelize stats collection. This uses the `--batch-size`, `--batch-index`, and `--batch-delay` flags added in [gh-repo-stats-plus v3.0.0](https://github.com/mona-actions/gh-repo-stats-plus/releases/tag/v3.0.0).

A setup job calculates the number of batches, parallel collect jobs gather stats for each batch, and a final combine job merges results:

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.calc.outputs.matrix }}
    steps:
      - name: Calculate batch matrix
        id: calc
        run: |
          REPO_COUNT=$(gh api "orgs/${ORG}" --jq '(.public_repos // 0) + (.total_private_repos // 0)')
          BATCH_SIZE=${BATCH_SIZE_INPUT}
          TOTAL_BATCHES=$(( (REPO_COUNT + BATCH_SIZE - 1) / BATCH_SIZE ))
          echo "Org has ${REPO_COUNT} repos → ${TOTAL_BATCHES} batches of ${BATCH_SIZE}"
          INDICES=$(jq -nc "[range(${TOTAL_BATCHES})]")
          echo "matrix={\"batch-index\":${INDICES}}" >> "$GITHUB_OUTPUT"
        env:
          GH_TOKEN: ${{ secrets.ACCESS_TOKEN }}
          ORG: ${{ inputs.organization }}
          BATCH_SIZE_INPUT: ${{ inputs.batch-size }}

  collect:
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix: ${{ fromJson(needs.setup.outputs.matrix) }}
    steps:
      - uses: actions/checkout@v4
      - name: Gather Stats (batch ${{ matrix.batch-index }})
        uses: mona-actions/gh-repo-stats-plus-action@v1
        with:
          type: organization
          github-token: ${{ github.token }}
          access-token: ${{ secrets.ACCESS_TOKEN }}
          organization: ${{ inputs.organization }}
          batch-size: ${{ inputs.batch-size }}
          batch-index: ${{ matrix.batch-index }}
          batch-delay: "5"

  combine:
    needs: collect
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/download-artifact@v4
        with:
          pattern: repo-stats-organization-*
          path: output
          merge-multiple: true
      - name: Combine Results
        uses: mona-actions/gh-repo-stats-plus-action@v1
        with:
          type: combine
          github-token: ${{ github.token }}
          organization: ${{ inputs.organization }}
```

See [batch-organization-stats.yml](examples/batch-organization-stats.yml) for a complete workflow and the [batch processing docs](https://github.com/mona-actions/gh-repo-stats-plus/blob/main/docs/batch-processing.md) for more details.

### Combining Batch Results

Use `type: combine` to merge CSV files from multiple batch runs into a single file. This is typically used as the final job in a batched workflow after downloading all batch artifacts:

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: repo-stats-organization-*
    path: output
    merge-multiple: true

- name: Combine Results
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    type: combine
    github-token: ${{ github.token }}
    organization: my-org
```

### Resume Failed Runs

The action supports resuming from the last saved state when a stats collection run fails partway through. The underlying [gh-repo-stats-plus](https://github.com/mona-actions/gh-repo-stats-plus) CLI writes state files (`last_known_state_<org>.json`) and partial CSVs to the output directory after each successfully processed repository. When resume is enabled, the action restores these files and passes `--resume-from-last-save` to the CLI, which skips already-processed repositories.

**Automatic resume (re-run failed jobs):** When you click "Re-run failed jobs" in the GitHub Actions UI, the action detects `run_attempt > 1` and automatically downloads the state artifact from the previous attempt. No configuration needed:

```yaml
- name: Gather Organization Stats
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    type: organization
    github-token: ${{ github.token }}
    access-token: ${{ secrets.ACCESS_TOKEN }}
    organization: my-org
    # Resume is automatic when re-running failed jobs — no extra inputs needed
```

**Explicit resume (cross-run):** To resume from a completely different workflow run, provide the run ID:

```yaml
- name: Gather Organization Stats (resuming)
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    type: organization
    github-token: ${{ github.token }}
    access-token: ${{ secrets.ACCESS_TOKEN }}
    organization: my-org
    resume-from-last-save: "true"
    resume-run-id: "1234567890" # Run ID of the failed run to resume from
```

**How it works:**

1. When a stats collection step fails or is cancelled, the action uploads the output directory (containing state files and partial CSVs) as a state artifact.
2. On the next run (re-run or explicit), the action downloads the state artifact into the output directory before invoking the CLI.
3. The CLI reads the state file, identifies which repositories have already been processed, and resumes from where it left off.

**Limitations:**

- If the GitHub Actions runner is terminated abruptly (infrastructure issue, spot instance preemption), the failure upload step may not execute — no state is saved in that case.
- If the CLI fails before processing any repository, there is no state to resume from.
- Resume applies to `repo-stats` and `project-stats` commands. The `app-install-stats` and `migration-audit` commands do not support resume.
- The workflow must have `actions: read` permission for the `GITHUB_TOKEN` to download artifacts from previous runs. Add `permissions: actions: read` (see [examples/resume-stats.yml](examples/resume-stats.yml)).
- Resume is not supported on GitHub Enterprise Server (GHES) because `actions/download-artifact@v4+` [does not support GHES](https://github.com/actions/download-artifact#ghes-support).
- When a workflow is cancelled, state upload runs during GitHub Actions' cancellation grace period. If the output directory is very large, the upload may not complete in time.

**Runner compatibility:**

Resume works on both **GitHub-hosted** and **self-hosted** runners. On self-hosted runners with persistent workspaces, the CLI's state files may already exist on disk from a previous run. If the artifact download fails (e.g., no state artifact exists), the CLI will still pick up any local state files when `--resume-from-last-save` is active. Note that `actions/checkout@v4` cleans the workspace by default (`clean: true`), which removes local state files — to preserve them across runs on a self-hosted runner, set `output-dir` to an absolute path outside `$GITHUB_WORKSPACE`.

See [examples/resume-stats.yml](examples/resume-stats.yml) for a complete workflow example.

### Project Stats

Collect [ProjectsV2 statistics](https://github.com/mona-actions/gh-repo-stats-plus/blob/main/docs/commands/project-stats.md) for all repositories in an organization. This counts unique projects linked to repositories via issues and directly.

```yaml
- name: Gather Project Stats
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    type: project-stats
    github-token: ${{ github.token }}
    access-token: ${{ secrets.ACCESS_TOKEN }}
    organization: my-org
```

Project stats supports the same batch processing options as organization stats (`batch-size`, `batch-index`, `batch-delay`) and can also use GitHub App authentication.

### App Install Stats

Collect [GitHub App installation statistics](https://github.com/mona-actions/gh-repo-stats-plus/blob/main/docs/commands/app-install-stats.md) for an organization. Produces up to three CSV files showing which apps are installed on which repositories.

> **Important:** This command requires a Personal Access Token (PAT) with `read:org` scope. GitHub App tokens **cannot** be used because app tokens can only see their own installation, not other apps' installations.

```yaml
- name: Gather App Install Stats
  uses: mona-actions/gh-repo-stats-plus-action@v1
  with:
    type: app-install-stats
    github-token: ${{ github.token }}
    access-token: ${{ secrets.ACCESS_TOKEN }}
    organization: my-org
```

## Examples

The [examples/](examples/) directory contains complete workflow files and setups you can use as a starting point:

- [**Repository Stats**](examples/repository-stats.yml) — A simple workflow that gathers stats for a single repository on a weekly schedule, with optional migration audit.
- [**Organization Stats**](examples/organization-stats.yml) — A simple workflow that gathers stats for all repositories in an organization on a weekly schedule.
- [**Batch Organization Stats**](examples/batch-organization-stats.yml) — A workflow that gathers stats for all repositories in a large organization using batch processing with matrix strategy, then combines results.
- [**Project Stats**](examples/project-stats.yml) — A simple workflow that gathers ProjectsV2 statistics for all repositories in an organization on a weekly schedule.
- [**App Install Stats**](examples/app-install-stats.yml) — A simple workflow that gathers GitHub App installation statistics for an organization on a weekly schedule.
- [**Resume Stats**](examples/resume-stats.yml) — A workflow demonstrating how to resume failed stats collection runs, with both automatic (re-run) and explicit (cross-run) resume patterns.
- [**Issue Ops**](examples/issue-ops/) — A full [IssueOps](https://github.com/issue-ops) example that triggers stats gathering via `/run-stats` comments on GitHub Issues. Includes issue form templates for single-repository and organization-wide stats (with an operation dropdown to select repo stats, project stats, app installs, or migration audit), parses issue body and labels to determine the run type, posts results back as issue comments, and supports optional migration audits.

## CI

The CI workflow runs on every push to `main` and on pull requests. It performs static analysis only — no secrets or API calls required:

- **ShellCheck** — Lints the embedded bash scripts extracted from `action.yml`
- **yamllint** — Checks YAML syntax across all config files

See [.github/workflows/ci.yml](.github/workflows/ci.yml) for details.

For full end-to-end testing, use the **Integration Test** workflow which can be triggered manually via `workflow_dispatch`. It runs the action against a specified repository and verifies the outputs. See [.github/workflows/integration-test.yml](.github/workflows/integration-test.yml).

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
│   ├── batch-organization-stats.yml # Batch org stats with matrix strategy
│   ├── project-stats.yml       # Project statistics example
│   ├── app-install-stats.yml   # App installation stats example
│   ├── resume-stats.yml        # Resume failed runs example
│   └── issue-ops/              # Full IssueOps example
│       ├── gather-stats.yml
│       ├── issue-templates/
│       │   ├── 01-get-repo-stats.yml
│       │   └── 02-get-org-stats.yml
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

```bash
git tag -fa v1 -m "Update v1 tag"
git push origin v1 --force
```

## Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

See [LICENSE](LICENSE) for details.
