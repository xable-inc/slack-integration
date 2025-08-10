# Slack Integration

このリポジトリは、他のプロジェクトからSlack通知を送信するための再利用可能なワークフローを提供します。

## 概要

3つの通知タイプをサポートしています：
1. **CI結果通知** - PR/コミットのCI実行結果を通知
2. **デプロイ開始通知** - デプロイ開始時の通知（スレッド作成）
3. **デプロイ結果通知** - デプロイ完了時の通知（スレッド返信）

## セットアップ

### 1. Slack Webhook URLの準備
Slackアプリを作成し、Incoming WebhookのURLを取得してください。

### 2. GitHubシークレットの設定
プロジェクトの設定で以下のシークレットを追加：
```
SLACK_WEBHOOK_URL: your_slack_webhook_url_here
```

## バージョン指定について

**推奨順序：**
1. **特定バージョン** (最安全):
```yaml
uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-ci-result.yml@v1.0.0
```

2. **Major version** (便利だが運用が必要):
```yaml
uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-ci-result.yml@v1
```
※ `v1`タグは手動で最新の`v1.x.x`に移動する必要があります

3. **ブランチ** (開発時のみ):
```yaml
uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-ci-result.yml@main
```

## 使用方法

### CI結果通知

CIワークフローの最後に以下を追加：

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      # あなたのテスト処理
      - name: Run tests
        run: npm test

  notify:
    needs: [test]
    if: always()
    uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-ci-result.yml@v1
    with:
      webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
      status: ${{ needs.test.result }}
      project_name: "MyProject"
      commit_message: ${{ github.event.head_commit.message || github.event.pull_request.title }}
      author: ${{ github.actor }}
      action_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

### デプロイ開始通知

デプロイワークフローの開始時：

```yaml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'development'
        type: choice
        options:
        - development
        - production

jobs:
  notify-start:
    runs-on: ubuntu-latest
    outputs:
      thread_ts: ${{ steps.slack.outputs.thread_ts }}
    steps:
      - name: Notify deploy start
        id: slack
        uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-deploy-start.yml@v1
        with:
          webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
          project_name: "MyProject"
          environment: ${{ github.event.inputs.environment || 'development' }}
          pr_title: ${{ github.event.head_commit.message }}
          author: ${{ github.actor }}

  deploy:
    needs: [notify-start]
    runs-on: ubuntu-latest
    steps:
      # あなたのデプロイ処理
      - name: Deploy to environment
        run: echo "Deploying..."

  notify-result:
    needs: [notify-start, deploy]
    if: always()
    uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-deploy-result.yml@v1
    with:
      webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
      status: ${{ needs.deploy.result }}
      project_name: "MyProject"
      environment: ${{ github.event.inputs.environment || 'development' }}
      thread_ts: ${{ needs.notify-start.outputs.thread_ts }}
      author: ${{ github.actor }}
```

## パラメータ詳細

### notify-ci-result.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| webhook_url | ✅ | - | Slack Webhook URL |
| status | ✅ | - | CI結果 (success/failure/cancelled) |
| project_name | ✅ | - | プロジェクト名 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |
| commit_message | ❌ | PR title | コミットメッセージ |
| author | ❌ | github.actor | PR作成者 |

### notify-deploy-start.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| webhook_url | ✅ | - | Slack Webhook URL |
| project_name | ✅ | - | プロジェクト名 |
| environment | ✅ | - | デプロイ環境 (development/production) |
| pr_title | ❌ | PR title | PRタイトル |
| pr_url | ❌ | PR URL | PR URL |
| author | ❌ | github.actor | デプロイ実行者 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |

**出力:**
- `thread_ts`: Slackスレッドタイムスタンプ（結果通知で使用）

### notify-deploy-result.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| webhook_url | ✅ | - | Slack Webhook URL |
| status | ✅ | - | デプロイ結果 (success/failure/cancelled) |
| project_name | ✅ | - | プロジェクト名 |
| environment | ✅ | - | デプロイ環境 |
| thread_ts | ✅ | - | 開始通知のスレッドタイムスタンプ |
| pr_title | ❌ | PR title | PRタイトル |
| pr_url | ❌ | PR URL | PR URL |
| author | ❌ | github.actor | デプロイ実行者 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |

## ユーザーマッピング

GitHubユーザーIDをSlackユーザーIDにマッピングするには、`.github/actions/slack-notify/action.yml`の`Map GitHub user to Slack user`ステップを編集してください：

```yaml
- name: Map GitHub user to Slack user
  id: user_mapping
  shell: bash
  run: |
    case "${{ inputs.user_id }}" in
      "github_username1")
        echo "slack_user=<@SLACK_USER_ID1>" >> $GITHUB_OUTPUT
        ;;
      "github_username2")
        echo "slack_user=<@SLACK_USER_ID2>" >> $GITHUB_OUTPUT
        ;;
      # 他のユーザーマッピングをここに追加
      *)
        echo "slack_user=${{ inputs.user_id }}" >> $GITHUB_OUTPUT
        ;;
    esac
```

## Slackメッセージ例

### CI結果通知
```
MyProject - ✅ 成功

担当者: @username
ワークフロー: CI検証

PR: Fix bug in user authentication

CI結果をお知らせします
```

### デプロイ開始通知
```
MyProject (本番環境) - 🚀 開始

担当者: @username
ワークフロー: デプロイ開始

PR: Release v1.2.0

🚀 本番環境へのデプロイを開始します！
```

### デプロイ結果通知（スレッド返信）
```
MyProject (本番環境) - ✅ 成功

担当者: @username
ワークフロー: デプロイ結果

PR: Release v1.2.0

✅ 本番環境へのデプロイが完了しました！
```

## リリース管理

### Major versionタグの管理
`v1`タグを最新の`v1.x.x`に移動するには：

```bash
# 新しいリリースを作成
git tag v1.2.0
git push origin v1.2.0

# v1タグを移動
git tag -f v1 v1.2.0
git push origin v1 --force
```

### Dependabotによる自動更新
`.github/dependabot.yml`で依存関係を自動更新：

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
```

## トラブルシューティング

### Webhook URLが無効
- Slackアプリの設定を確認
- GitHubシークレットが正しく設定されているか確認

### ユーザーメンションが機能しない
- Slackユーザーマッピングが正しく設定されているか確認
- SlackユーザーIDの形式: `<@U1234567890>`

### スレッド返信が機能しない
- デプロイ開始通知の`thread_ts`出力が正しく渡されているか確認
- `needs`依存関係が正しく設定されているか確認