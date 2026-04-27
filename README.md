# Findy Team+ CI Analytics Action

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

A GitHub Action that sends CI workflow execution results to [Findy Team+](https://jp.findy-team.io/) for development performance analytics.

**[日本語版 README はこちら](README.ja.md)**

## Usage

### Basic

Add the following job to your existing workflow:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: echo "build"

  ci-report:
    runs-on: ubuntu-slim
    if: always()
    needs: [build]
    permissions:
      actions: read
    steps:
      # Pin to a specific commit SHA for supply chain security.
      # Find the SHA for each release at:
      # https://github.com/Findy/team-plus-cicd-analytics/releases
      - uses: Findy/team-plus-cicd-analytics@<COMMIT_SHA> # v1.0.0
        with:
          organization-id: ${{ secrets.TEAM_PLUS_ORGANIZATION_ID }}
          api-token: ${{ secrets.TEAM_PLUS_WEBAPI_TOKEN }}
          status: ${{ needs.build.result }}
          github-token: ${{ github.token }}
```

### Multi-job workflow

To report combined results from multiple jobs:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm test

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run lint

  ci-report:
    runs-on: ubuntu-slim
    if: always()
    needs: [test, lint]
    permissions:
      actions: read
    steps:
      # Pin to a specific commit SHA for supply chain security.
      # Find the SHA for each release at:
      # https://github.com/Findy/team-plus-cicd-analytics/releases
      - uses: Findy/team-plus-cicd-analytics@<COMMIT_SHA> # v1.0.0
        with:
          organization-id: ${{ secrets.TEAM_PLUS_ORGANIZATION_ID }}
          api-token: ${{ secrets.TEAM_PLUS_WEBAPI_TOKEN }}
          status: ${{ (contains(needs.*.result, 'failure') && 'failure') || (contains(needs.*.result, 'cancelled') && 'cancelled') || 'success' }}
          github-token: ${{ github.token }}
```

## Inputs

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `organization-id` | Findy Team+ organization ID | Yes | - |
| `api-token` | Findy Team+ API token | Yes | - |
| `status` | Workflow execution result (`success`, `failure`, `cancelled`) | Yes | - |
| `github-token` | GitHub token for fetching workflow run data | Yes | - |
| `include-jobs` | Whether to include job-level detail data | No | `true` |
| `job-timeout` | Timeout in seconds for fetching job data | No | `5` |

## Setup

### 1. Get an API token from Findy Team+

Follow the [Findy Team+ CI Analytics](https://jp.findy-team.io/support/ci-analytics-setup/) documentation to obtain your organization ID and API token.

### 2. Register GitHub Secrets

Go to your repository's **Settings > Secrets and variables > Actions** and add:

- `TEAM_PLUS_ORGANIZATION_ID` — Your Findy Team+ organization ID
- `TEAM_PLUS_WEBAPI_TOKEN` — Your Findy Team+ API token

### 3. Add to your workflow

Add a `ci-report` job to your workflow using the examples above.

> **Important**: Always set `if: always()` on the report job so results are sent even when preceding jobs fail.

## Required permissions

This Action requires the following permissions:

```yaml
permissions:
  actions: read  # Read workflow run data
```

When using the default `${{ github.token }}` for `github-token`, these permissions are granted automatically.

## Data sent

### Workflow data

| Field | Description |
|-------|-------------|
| `organization_name` | Findy Team+ organization ID |
| `repo_name` | Repository name (`owner/repo` format) |
| `workflow_name` | Workflow name |
| `run_id` | Workflow run ID |
| `url` | Workflow run URL |
| `status` | Execution result status |
| `start_at` | Workflow start time |
| `end_at` | Workflow end time |
| `jobs` | Array of job detail data (when `include-jobs: true`) |

### Job data (optional)

When `include-jobs: true` (default), the following data is included for each completed job:

| Field | Description |
|-------|-------------|
| `name` | Job name |
| `status` | Job status (`success`, `failure`, `cancelled`, `skipped`) |
| `start_at` | Job start time |
| `end_at` | Job end time |

> **Note**: The currently running report job itself is excluded. Only completed jobs are sent.

## Contributing

Bug reports and feature requests are welcome via [Issues](https://github.com/Findy/team-plus-cicd-analytics/issues).
Pull requests are also welcome!

## License

Apache License 2.0 — see [LICENSE](LICENSE) for details.
