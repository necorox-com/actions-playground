# デプロイパイプライン - repository_dispatch チェーン方式

## Context

GitHub Actions でセルフホストランナーを使ったデプロイパイプラインを構築する。各ステップを独立したワークフローとし、`repository_dispatch` で連鎖実行することで、柔軟にパイプラインを組み替え可能にする。**どのリポジトリにもコピーして使える汎用テンプレート**として作成する。

## 決定事項サマリ

| 項目 | 決定 |
|------|------|
| 連携方式 | repository_dispatch チェーン |
| トリガー | workflow_dispatch（手動）+ push（自動、ブランチはパラメータ化） |
| タグ命名 | `release-{yyyyMMdd-HHmmss}-{GITHUB_RUN_NUMBER}` |
| デプロイ | git fetch + checkout のみ（clone済み前提） |
| ロールバック | gitタグ一覧から前のタグを取得→checkout |
| Chat形式 | Google Chat Card V2（ステータス色分け+メッセージ+リンク+タイムスタンプ） |
| CDNキャッシュ | Cloudflare API で全パージ |
| 同時実行制御 | concurrency グループで制御 |
| 失敗通知 | 各ステップで個別にエラー通知 |
| PAT | Classic PAT（`repo` スコープ） |
| ランナー | VPS上に直接インストール（OS非依存ドキュメント） |
| 個別実行 | 各ワークフローに workflow_dispatch 追加 |
| マルチ環境 | 今は単一、将来拡張可能な設計 |

## パイプラインフロー

```
deploy-pipeline (入口: workflow_dispatch / push)
  │ repository_dispatch: create-tag
  ▼
_create-tag (GitHub-hosted)
  │ タグ作成: release-20260208-143052-42
  │ repository_dispatch: notify-chat
  ▼
_notify-chat (GitHub-hosted)
  │ Card V2: "✓ タグ作成完了: release-20260208-143052-42"
  │ repository_dispatch: deploy-checkout (next_event)
  ▼
_deploy-checkout (self-hosted @ VPS)
  │ /var/www/app で git fetch + git checkout tags/release-...
  │ 失敗時: 前のタグにロールバック + エラー通知
  │ repository_dispatch: notify-chat
  ▼
_notify-chat (GitHub-hosted)
  │ Card V2: "✓ デプロイ完了" or "✗ デプロイ失敗（ロールバック済み）"
  │ repository_dispatch: clear-cdn-cache (next_event)
  ▼
_clear-cdn-cache (GitHub-hosted)
  │ Cloudflare API purge_cache
  │ repository_dispatch: notify-chat
  ▼
_notify-chat (GitHub-hosted)
  │ Card V2: "✓ CDNキャッシュ削除完了" (next_event: なし → 終了)
```

---

## 作成ファイル一覧

### Composite Actions（`.github/actions/`）

| # | ファイル | 概要 |
|---|---------|------|
| 1 | `create-tag/action.yml` | ブランチの最新コミットにタグを作成・プッシュ |
| 2 | `notify-chat/action.yml` | Google Chat Card V2 送信 + 次アクションdispatch |
| 3 | `deploy-checkout/action.yml` | 指定ディレクトリでタグをfetch+checkout（ロールバック付き） |
| 4 | `clear-cdn-cache/action.yml` | Cloudflare CDNキャッシュ全パージ |

### Workflows（`.github/workflows/`）

| # | ファイル | トリガー | ランナー |
|---|---------|---------|---------|
| 5 | `deploy-pipeline.yml` | `workflow_dispatch` + `push` | `ubuntu-latest` |
| 6 | `_create-tag.yml` | `repository_dispatch` + `workflow_dispatch` | `ubuntu-latest` |
| 7 | `_notify-chat.yml` | `repository_dispatch` + `workflow_dispatch` | `ubuntu-latest` |
| 8 | `_deploy-checkout.yml` | `repository_dispatch` + `workflow_dispatch` | `[self-hosted, deploy]` |
| 9 | `_clear-cdn-cache.yml` | `repository_dispatch` + `workflow_dispatch` | `ubuntu-latest` |

### ドキュメント

| # | ファイル |
|---|---------|
| 10 | `docs/github_actions/deploy-pipeline.md` |

---

## 詳細設計

### 1. `create-tag/action.yml`

```yaml
inputs:
  tag_name:   # required - 作成するタグ名
  branch:     # required - タグを作成するブランチ (default: main)
```

**処理フロー:**
1. `actions/checkout` でリポジトリをチェックアウト（対象ブランチ）
2. `git tag ${tag_name}` を作成
3. `git push origin ${tag_name}`

**outputs:**
- `tag_name`: 作成されたタグ名
- `commit_sha`: タグが指すコミットSHA

### 2. `notify-chat/action.yml`（チェーンの中核）

```yaml
inputs:
  webhook_url:   # required - Google Chat webhook URL
  status:        # required - success / failure / info
  title:         # required - カードタイトル
  message:       # required - メッセージ本文
  run_url:       # optional - GitHub Actions Run URL (default: 自動取得)
  next_event:    # optional - 次にトリガーするrepository_dispatch event_type
  payload:       # optional - 次のワークフローに渡すJSON payload
  github_token:  # required if next_event - dispatch発火用トークン
  repository:    # optional - dispatch先リポジトリ (default: current)
```

**処理フロー:**
1. Google Chat Card V2 JSONを組み立て
   - ヘッダー: タイトル + ステータスアイコン（✓成功=緑、✗失敗=赤、ℹ情報=青）
   - セクション: メッセージ本文 + タイムスタンプ
   - ボタン: GitHub Actions Runへのリンク
2. `curl` でwebhook URLにPOST
3. `next_event` が指定されていれば:
   - `curl` で GitHub API `POST /repos/{owner}/{repo}/dispatches` を呼び出し
   - `event_type: ${next_event}`, `client_payload: ${payload}` を送信
4. `next_event` が空ならパイプライン終了

### 3. `deploy-checkout/action.yml`

```yaml
inputs:
  tag_name:     # required - チェックアウトするタグ
  deploy_dir:   # required - デプロイ先ディレクトリパス
```

**処理フロー:**
1. `cd ${deploy_dir}`
2. 現在のHEADを記録: `previous_ref=$(git describe --tags --always)`
3. `git fetch origin --tags --force`
4. `git checkout tags/${tag_name}` を実行
5. **失敗時ロールバック:**
   - `release-*` タグを日付降順でソートし、current tag の1つ前のタグを取得
   - `git checkout tags/${previous_tag}`
   - outputs に `rollback: true` と `rollback_tag` を設定

**outputs:**
- `success`: true/false
- `previous_ref`: チェックアウト前のref
- `rollback`: ロールバックしたかどうか
- `rollback_tag`: ロールバック先のタグ

### 4. `clear-cdn-cache/action.yml`

```yaml
inputs:
  cloudflare_zone_id:    # required
  cloudflare_api_token:  # required
```

**処理フロー:**
1. Cloudflare API 呼び出し:
   ```
   POST https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache
   {"purge_everything": true}
   ```
2. レスポンスの `success` フィールドを確認

**outputs:**
- `success`: true/false

### 5. `deploy-pipeline.yml`（入口）

```yaml
on:
  workflow_dispatch:
    inputs:
      branch:
        description: 'デプロイ対象ブランチ'
        default: 'main'
      deploy_dir:
        description: 'デプロイ先ディレクトリ'
        required: true
      # tag_prefix は固定 "release"
  push:
    branches:
      - main  # ※テンプレート利用者がカスタマイズ

concurrency:
  group: deploy-pipeline
  cancel-in-progress: false  # 実行中のデプロイはキャンセルしない（待機）
```

**処理:**
1. タグ名を生成: `release-$(date +%Y%m%d-%H%M%S)-${GITHUB_RUN_NUMBER}`
2. パイプラインコンテキストを組み立て:
   ```json
   {
     "tag": "release-20260208-143052-42",
     "branch": "main",
     "deploy_dir": "/var/www/app",
     "chat_webhook_url": "${{ secrets.GOOGLE_CHAT_WEBHOOK_URL }}",
     "cloudflare_zone_id": "${{ secrets.CLOUDFLARE_ZONE_ID }}",
     "pipeline_id": "run-${GITHUB_RUN_ID}",
     "repository": "${GITHUB_REPOSITORY}"
   }
   ```
3. `repository_dispatch` で `create-tag` イベントを発火

### 6-9. 各ステップワークフロー（共通パターン）

各ワークフローは同じパターンに従う:

```yaml
on:
  repository_dispatch:
    types: [<event-type>]
  workflow_dispatch:
    inputs: ...  # 個別実行用

permissions:
  contents: write  # タグ操作やdispatch用

concurrency:
  group: deploy-pipeline
  cancel-in-progress: false

jobs:
  <step-name>:
    runs-on: <runner>
    steps:
      - uses: ./.github/actions/<action>  # メイン処理
      - uses: ./.github/actions/notify-chat  # 成功時: 次ステップへ
        if: success()
      - uses: ./.github/actions/notify-chat  # 失敗時: エラー通知（next_event空）
        if: failure()
```

**注意:** `repository_dispatch` でトリガーされた場合、ワークフロー内で `./.github/actions/` を参照するには `actions/checkout` が先に必要。

### 10. `docs/github_actions/deploy-pipeline.md`

**内容:**
1. パイプライン概要・フロー図
2. 必要なSecrets一覧と設定手順
   - `PIPELINE_GITHUB_TOKEN`: Classic PAT（`repo`スコープ）の作成手順
   - `GOOGLE_CHAT_WEBHOOK_URL`: Google Chat webhook設定手順
   - `CLOUDFLARE_API_TOKEN`: Cloudflare APIトークン作成手順
   - `CLOUDFLARE_ZONE_ID`: Zone ID取得方法
3. セルフホストランナーのセットアップ手順（OS非依存）
   - GitHub公式ドキュメントへのリンク
   - ラベル `deploy` の設定
   - サービスとしての登録
   - デプロイディレクトリへのアクセス権設定
4. カスタマイズガイド
   - ブランチパターンの変更
   - パイプラインステップの追加・削除
   - マルチ環境への拡張方法
5. トラブルシューティング

---

## 必要なSecrets

| Secret名 | 用途 | 設定値 |
|----------|------|--------|
| `PIPELINE_GITHUB_TOKEN` | repository_dispatch発火用 | Classic PAT（`repo`スコープ） |
| `GOOGLE_CHAT_WEBHOOK_URL` | Google Chat通知 | Webhook URL |
| `CLOUDFLARE_API_TOKEN` | CDNキャッシュパージ | APIトークン（Zone.Cache Purge権限） |
| `CLOUDFLARE_ZONE_ID` | 対象ゾーン指定 | Zone ID |

## 実装順序

1. **notify-chat** action + workflow（チェーンの中核）
2. **create-tag** action + workflow
3. **deploy-checkout** action + workflow
4. **clear-cdn-cache** action + workflow
5. **deploy-pipeline.yml**（入口ワークフロー）
6. **deploy-pipeline.md**（ドキュメント）

## 検証方法

1. `_notify-chat.yml` を workflow_dispatch で単体テスト → Google Chatに通知が届くか
2. `_create-tag.yml` を workflow_dispatch で単体テスト → タグが作成されるか
3. セルフホストランナーをVPSにセットアップ
4. `_deploy-checkout.yml` を workflow_dispatch で単体テスト → 指定ディレクトリでcheckoutされるか
5. `_clear-cdn-cache.yml` を workflow_dispatch で単体テスト → Cloudflareキャッシュがパージされるか
6. `deploy-pipeline.yml` を workflow_dispatch で全体テスト → チェーン全体が動作するか
