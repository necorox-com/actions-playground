# デプロイパイプライン

repository_dispatch チェーン方式によるデプロイパイプライン。各ステップが独立したワークフローとして実行され、完了後に次のワークフローを自動的にトリガーする。

## パイプラインフロー

```
deploy-pipeline (入口: workflow_dispatch / push)
  │ repository_dispatch: create-tag
  ▼
_create-tag (GitHub-hosted runner)
  │ タグ作成: release-20260208-143052-42
  │ → notify-chat: 「タグ作成完了」
  │   → repository_dispatch: deploy-checkout
  ▼
_deploy-checkout (self-hosted runner @ VPS)
  │ 指定ディレクトリで git fetch + checkout
  │ 失敗時: 前のタグにロールバック
  │ → notify-chat: 「デプロイ完了」or「デプロイ失敗」
  │   → repository_dispatch: clear-cdn-cache
  ▼
_clear-cdn-cache (GitHub-hosted runner)
  │ Cloudflare CDNキャッシュ全パージ
  │ → notify-chat: 「パイプライン完了」(終了)
```

## セットアップ手順

### 1. Secrets の設定

リポジトリの Settings → Secrets and variables → Actions で以下を設定:

| Secret名 | 説明 | 取得方法 |
|----------|------|---------|
| `PIPELINE_GITHUB_TOKEN` | repository_dispatch 発火用 Personal Access Token | [PAT作成手順](#pat-の作成) |
| `GOOGLE_CHAT_WEBHOOK_URL` | Google Chat の Incoming Webhook URL | [Webhook設定手順](#google-chat-webhook-の設定) |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API トークン | [APIトークン作成手順](#cloudflare-api-トークンの作成) |
| `CLOUDFLARE_ZONE_ID` | 対象サイトの Cloudflare Zone ID | [Zone ID取得方法](#cloudflare-zone-id-の取得) |

### 2. セルフホストランナーのセットアップ

デプロイ先の VPS に GitHub Actions self-hosted runner をインストールする。

#### インストール

1. リポジトリの Settings → Actions → Runners → New self-hosted runner
2. GitHub の指示に従ってランナーをインストール
   - 公式ドキュメント: https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/adding-self-hosted-runners

#### ラベル設定

ランナーに `deploy` ラベルを追加:

```bash
./config.sh --labels deploy
```

既にインストール済みの場合は Settings → Actions → Runners から編集可能。

#### サービスとして登録

ランナーをサービスとして登録し、サーバー再起動後も自動起動するようにする:

```bash
sudo ./svc.sh install
sudo ./svc.sh start
```

#### デプロイディレクトリの準備

ランナーの実行ユーザーがデプロイ先ディレクトリにアクセスできることを確認:

```bash
# デプロイ先にリポジトリを clone
git clone https://github.com/<owner>/<repo>.git /var/www/app

# ランナーユーザーに権限を付与
sudo chown -R <runner-user>:<runner-group> /var/www/app
```

### 3. pushトリガーの設定（任意）

`deploy-pipeline.yml` の push トリガーを有効にする場合は、コメントを解除してブランチパターンを設定:

```yaml
on:
  workflow_dispatch:
    # ...
  push:
    branches:
      - main         # mainブランチへのpushでデプロイ
      # - release/*  # releaseブランチでもデプロイ
```

---

## 各Secret の詳細設定手順

### PAT の作成

1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 「Generate new token (classic)」をクリック
3. 設定:
   - Note: `deploy-pipeline-dispatch`
   - Expiration: 必要に応じて設定
   - Scopes: `repo` にチェック
4. トークンを生成してコピー
5. リポジトリの Secret `PIPELINE_GITHUB_TOKEN` に設定

> **注意**: `GITHUB_TOKEN` では `repository_dispatch` イベントをトリガーできないため、Classic PAT が必要です。

### Google Chat Webhook の設定

1. Google Chat でスペースを開く
2. スペース名の横の「▼」→「アプリと統合」→「Webhook を管理」
3. 「Webhook を追加」で名前を入力して作成
4. Webhook URL をコピー
5. リポジトリの Secret `GOOGLE_CHAT_WEBHOOK_URL` に設定

### Cloudflare API トークンの作成

1. Cloudflare ダッシュボード → My Profile → API Tokens
2. 「Create Token」→「Custom token」
3. 設定:
   - Token name: `deploy-pipeline-cache-purge`
   - Permissions: Zone → Cache Purge → Purge
   - Zone Resources: 対象のゾーンを選択
4. トークンを作成してコピー
5. リポジトリの Secret `CLOUDFLARE_API_TOKEN` に設定

### Cloudflare Zone ID の取得

1. Cloudflare ダッシュボード → 対象サイト → Overview
2. 右サイドバーの「API」セクションに Zone ID が表示される
3. リポジトリの Secret `CLOUDFLARE_ZONE_ID` に設定

---

## 使い方

### 手動実行

1. GitHub → Actions → 「デプロイパイプライン」
2. 「Run workflow」をクリック
3. パラメータを入力:
   - **branch**: デプロイ対象ブランチ（デフォルト: `main`）
   - **deploy_dir**: デプロイ先ディレクトリ（例: `/var/www/app`）
4. 「Run workflow」で実行

### 個別ステップの実行

各ワークフローは `workflow_dispatch` で個別に実行可能:

- **タグ作成のみ**: Actions → 「タグ作成」→ Run workflow
- **Chat通知のテスト**: Actions → 「Chat通知」→ Run workflow
- **デプロイのみ**: Actions → 「デプロイ」→ Run workflow
- **キャッシュ削除のみ**: Actions → 「CDNキャッシュ削除」→ Run workflow

---

## カスタマイズ

### パイプラインステップの変更

`_notify-chat.yml` の `next_event` パラメータを変更することで、パイプラインの順序を変更できる。

例: CDNキャッシュ削除をスキップする場合、`_deploy-checkout.yml` の成功通知で `next_event` を空にする。

### ステップの追加

1. `.github/actions/` に新しい composite action を作成
2. `.github/workflows/` に新しいワークフロー（`repository_dispatch` + `workflow_dispatch` トリガー）を作成
3. 前のステップの `notify-chat` で新しいステップの `event_type` を指定

### マルチ環境への拡張

1. 環境ごとにセルフホストランナーをセットアップ（ラベル例: `deploy-staging`, `deploy-production`）
2. `_deploy-checkout.yml` を環境ごとに作成、または `runs-on` のラベルをパラメータ化
3. `deploy-pipeline.yml` に環境選択の input を追加

---

## トラブルシューティング

### repository_dispatch が発火しない

- `PIPELINE_GITHUB_TOKEN` が有効か確認（期限切れの可能性）
- PAT のスコープに `repo` が含まれているか確認
- ワークフローファイルの `repository_dispatch.types` が正しいか確認

### セルフホストランナーが応答しない

- ランナーのステータス確認: Settings → Actions → Runners
- ランナーサービスのステータス確認: `sudo ./svc.sh status`
- ランナーのログ確認: `_diag/` ディレクトリ内のログファイル

### デプロイが失敗する

- デプロイディレクトリの権限確認: `ls -la /var/www/app`
- git の状態確認: `cd /var/www/app && git status`
- ローカルの変更がある場合は `git stash` または `git reset --hard`

### Google Chat通知が届かない

- Webhook URL が有効か確認（スペースからWebhookが削除されていないか）
- Secretの値にスペースや改行が含まれていないか確認

### CDNキャッシュ削除が失敗する

- API トークンの権限確認（Zone.Cache Purge）
- Zone ID が正しいか確認
- API トークンの有効期限を確認
