# Findy Team+ CI Analytics Action

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

CIワークフローの実行結果を[Findy Team+](https://jp.findy-team.io/)に送信し、開発パフォーマンスの分析を可能にするGitHub Actionです。

## 使い方

### 基本的な使用方法

既存のワークフローに以下のジョブを追加します:

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
      # サプライチェーンセキュリティのため、特定のコミットSHAを指定してください。
      # 各リリースのSHAは以下で確認できます:
      # https://github.com/Findy/team-plus-cicd-analytics/releases
      - uses: Findy/team-plus-cicd-analytics@<COMMIT_SHA> # v1.0.0
        with:
          organization-id: ${{ secrets.TEAM_PLUS_ORGANIZATION_ID }}
          api-token: ${{ secrets.TEAM_PLUS_WEBAPI_TOKEN }}
          status: ${{ needs.build.result }}
          github-token: ${{ github.token }}
```

### 複数ジョブのワークフロー

複数ジョブの結果をまとめて報告する場合:

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
      # サプライチェーンセキュリティのため、特定のコミットSHAを指定してください。
      # 各リリースのSHAは以下で確認できます:
      # https://github.com/Findy/team-plus-cicd-analytics/releases
      - uses: Findy/team-plus-cicd-analytics@<COMMIT_SHA> # v1.0.0
        with:
          organization-id: ${{ secrets.TEAM_PLUS_ORGANIZATION_ID }}
          api-token: ${{ secrets.TEAM_PLUS_WEBAPI_TOKEN }}
          status: ${{ (contains(needs.*.result, 'failure') && 'failure') || (contains(needs.*.result, 'cancelled') && 'cancelled') || 'success' }}
          github-token: ${{ github.token }}
```

## 入力パラメータ

| パラメータ | 説明 | 必須 | デフォルト |
|-----------|------|------|------------|
| `organization-id` | Findy Team+の組織ID | ✅ | - |
| `api-token` | Findy Team+のAPIトークン | ✅ | - |
| `status` | ワークフローの実行結果ステータス (`success`, `failure`, `cancelled`) | ✅ | - |
| `github-token` | ワークフロー実行データ取得用のGitHubトークン | ✅ | - |
| `include-jobs` | ジョブの詳細データを送信するか | ❌ | `true` |
| `job-timeout` | ジョブデータ取得のタイムアウト秒数 | ❌ | `5` |

## セットアップ

### 1. Findy Team+でAPIトークンを取得

[Findy Team+ CI Analytics](https://jp.findy-team.io/support/ci-analytics-setup/)のドキュメントに従い、組織IDとAPIトークンを取得してください。

### 2. GitHub Secretsに登録

リポジトリの Settings → Secrets and variables → Actions で以下を登録します:

- `TEAM_PLUS_ORGANIZATION_ID`: Findy Team+の組織ID
- `TEAM_PLUS_WEBAPI_TOKEN`: Findy Team+のAPIトークン

### 3. ワークフローに追加

上記の使用例を参考に、ワークフローにci-reportジョブを追加してください。

> **重要**: reportジョブには `if: always()` を指定してください。これにより、先行ジョブが失敗した場合でも結果が報告されます。

## 必要な権限

このActionには以下の権限が必要です:

```yaml
permissions:
  actions: read  # ワークフロー実行データの読み取り
```

`github-token`にデフォルトの`${{ github.token }}`を使用する場合、上記の権限が自動的に付与されます。

## 送信されるデータ

このActionは以下のデータをFindy Team+に送信します:

### ワークフローデータ

| フィールド | 説明 |
|-----------|------|
| `organization_name` | Findy Team+の組織ID |
| `schema_version` | Payloadスキーマのバージョン（現在は `2`） |
| `repo_name` | リポジトリ名（`owner/repo`形式） |
| `workflow_name` | ワークフロー名 |
| `run_id` | ワークフロー実行ID |
| `url` | ワークフロー実行のURL |
| `status` | 実行結果ステータス |
| `start_at` | ワークフロー開始時刻 |
| `end_at` | ワークフロー終了時刻 |
| `jobs` | ジョブの詳細データ配列（`include-jobs: true`の場合） |

### ジョブデータ（オプション）

`include-jobs: true`（デフォルト）の場合、完了した各ジョブについて以下のデータが含まれます:

| フィールド | 説明 |
|-----------|------|
| `name` | ジョブ名 |
| `status` | ジョブのステータス (`success`, `failure`, `cancelled`, `skipped`) |
| `start_at` | ジョブ開始時刻 |
| `end_at` | ジョブ終了時刻 |
| `job_id` | GitHubのジョブID（jobs APIの `.id`） |
| `run_attempt` | ジョブの実行試行回数 |
| `runner_id` | ランナーID（取得できない場合は `null`） |
| `runner_name` | ランナー名（取得できない場合は `null`） |
| `runner_group_id` | ランナーグループID（取得できない場合は `null`） |
| `runner_group_name` | ランナーグループ名（取得できない場合は `null`） |
| `labels` | ジョブが要求したランナーラベル配列（`runs-on` 相当。取得できない場合は `null`） |

> **注意**: 現在実行中のレポートジョブ自身は含まれません。完了したジョブのみが送信されます。ランナー系フィールド（`runner_id` / `runner_name` / `runner_group_id` / `runner_group_name`）は GitHub API がランナー情報を返さない場合（スキップされたジョブなど）に `null` になります。`labels` は通常空配列で返りますが、API 応答に含まれない場合は `null` になります。

## 貢献

バグ報告や機能リクエストは[Issues](https://github.com/Findy/team-plus-cicd-analytics/issues)まで。
プルリクエストも歓迎します！

## ライセンス

Apache License 2.0 - 詳細は[LICENSE](LICENSE)ファイルを参照してください。
