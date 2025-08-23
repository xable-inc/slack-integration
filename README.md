# Slack Integration

このリポジトリは、他のプロジェクトからSlack通知を送信するための再利用可能なワークフローを提供します。

## 概要

3つの通知タイプをサポートしています：
1. **CI結果通知** - PR/コミットのCI実行結果を通知
2. **デプロイ開始通知** - デプロイ開始時の通知（スレッド作成）
3. **デプロイ結果通知** - デプロイ完了時の通知（スレッド返信）

## セットアップ

### セットアップ手順

#### 1. Slack Appの作成
1. https://api.slack.com/apps にアクセス
2. 「Create New App」→「From scratch」を選択
3. App名を入力（例：GitHub Actions Bot）
4. ワークスペースを選択して「Create App」

#### 2. 権限の設定
1. 左メニューの「OAuth & Permissions」をクリック
2. 「Bot Token Scopes」で以下を追加：
   - `chat:write` （メッセージ送信用）
   - `chat:write.public` （パブリックチャンネルへの投稿用）

#### 3. ワークスペースへインストール
1. 「Install to Workspace」をクリック
2. 権限を確認して「許可する」
3. 表示される「Bot User OAuth Token」をコピー（`xoxb-`で始まる）

#### 4. チャンネルIDの取得
1. Slackで投稿したいチャンネルを右クリック
2. 「チャンネル詳細を表示」→ 最下部の「チャンネルID」をコピー（例：`C05XXXXXX`）

#### 5. GitHubシークレットの設定
```
SLACK_BOT_TOKEN: xoxb-xxxxx... （Slack Appから取得）
SLACK_CHANNEL_ID: C05XXXXXX     （投稿先チャンネルID）
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
    secrets: inherit
    with:
      bot_token: ${{ secrets.SLACK_BOT_TOKEN }}
      channel_id: ${{ secrets.SLACK_CHANNEL_ID }}
      status: ${{ needs.test.result }}
      project_name: "MyProject"
      commit_message: ${{ github.event.head_commit.message || github.event.pull_request.title }}
      author: ${{ github.actor }}
      action_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

### デプロイ通知（スレッド返信対応）

デプロイワークフロー：

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
    uses: {ORGANIZATION}/slack-integration/.github/workflows/notify-deploy-start.yml@v1
    secrets: inherit
    with:
      bot_token: ${{ secrets.SLACK_BOT_TOKEN }}
      channel_id: ${{ secrets.SLACK_CHANNEL_ID }}
      project_name: "MyProject"
      environment: ${{ github.event.inputs.environment || 'development' }}
      pr_title: ${{ github.event.head_commit.message }}
      author: ${{ github.actor }}
      action_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}

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
    secrets: inherit
    with:
      bot_token: ${{ secrets.SLACK_BOT_TOKEN }}
      channel_id: ${{ secrets.SLACK_CHANNEL_ID }}
      status: ${{ needs.deploy.result }}
      project_name: "MyProject"
      environment: ${{ github.event.inputs.environment || 'development' }}
      thread_ts: ${{ needs.notify-start.outputs.thread_ts }}  # 自動的にスレッド返信になる
      author: ${{ github.actor }}
      action_url: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
```

## パラメータ詳細

### notify-ci-result.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| bot_token | ❌* | - | Slack Bot Token（推奨） |
| channel_id | ❌* | - | Slack Channel ID（推奨） |
| status | ✅ | - | CI結果 (success/failure/cancelled) |
| project_name | ✅ | - | プロジェクト名 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |
| commit_message | ❌ | PR title | コミットメッセージ |
| author | ❌ | github.actor | PR作成者 |

bot_token + channel_idが必須

### notify-deploy-start.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| bot_token | ❌* | - | Slack Bot Token（推奨） |
| channel_id | ❌* | - | Slack Channel ID（推奨） |
| project_name | ✅ | - | プロジェクト名 |
| environment | ✅ | - | デプロイ環境 (development/production) |
| pr_title | ❌ | PR title | PRタイトル |
| pr_url | ❌ | PR URL | PR URL |
| author | ❌ | github.actor | デプロイ実行者 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |

bot_token + channel_idが必須

**出力:**
- `thread_ts`: Slackスレッドタイムスタンプ（結果通知で使用、Web API使用時のみ）

### notify-deploy-result.yml

| パラメータ | 必須 | デフォルト値 | 説明 |
|-----------|------|-------------|------|
| bot_token | ❌* | - | Slack Bot Token（推奨） |
| channel_id | ❌* | - | Slack Channel ID（推奨） |
| status | ✅ | - | デプロイ結果 (success/failure/cancelled) |
| project_name | ✅ | - | プロジェクト名 |
| environment | ✅ | - | デプロイ環境 |
| thread_ts | ✅ | - | 開始通知のスレッドタイムスタンプ |
| pr_title | ❌ | PR title | PRタイトル |
| pr_url | ❌ | PR URL | PR URL |
| author | ❌ | github.actor | デプロイ実行者 |
| action_url | ❌ | 現在のワークフローURL | 実行アクションのURL |

bot_token + channel_idが必須

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

このリポジトリでは**自動でメジャーバージョンタグが管理されます**。

- `v1.0.0`をリリースすると、自動で`v1`タグも作成/更新されます
- `v1.0.1`をリリースすると、`v1`タグが自動で最新版に移動します
- 利用者は常に`@v1`で最新の互換バージョンを使用できます

**手動での操作は不要です。**

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
- **Webhook URLではスレッド返信は使用できません** - Bot Token + Channel IDを使用してください
- デプロイ開始通知の`thread_ts`出力が正しく渡されているか確認
- `needs`依存関係が正しく設定されているか確認
- `secrets: inherit`が設定されているか確認