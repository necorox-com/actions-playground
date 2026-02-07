# actions-playground

GitHub Actions のサンプル・実験用リポジトリ。

## リポジトリ構成

```
.github/
  workflows/           # ワークフローファイル（実際に動作する）
  actions/             # カスタムアクション（Composite/Docker/JS）
docs/
  github_actions/      # 各サンプルの詳細ドキュメント（IAM設定手順等）
```

### ワークフローファイル（`.github/workflows/`）

- このリポジトリ上で実際に実行可能
- 他リポジトリへは個別にコピーして使う
- ファイル先頭のコメントブロックに概要・前提・使い方を簡潔に記載する

```yaml
# -----------------------------------------------
# サンプル名 - 一行の概要
#
# 前提:
#   - 必要なSecretsや環境の箇条書き
#   - 詳細: docs/github_actions/xxx.md（あれば）
#
# 使い方:
#   .github/workflows/ にコピー
# -----------------------------------------------
```

### ドキュメント（`docs/github_actions/`）

- ワークフローの先頭コメントに収まらない詳細情報を記載する
  - 例: IAMポリシーの作成手順、アーキテクチャ図、トラブルシュート
- ファイル名はワークフローと対応させる（例: `deploy-cdn-invalidation.md`）
- コピー先リポジトリの `docs/` と衝突しにくいよう `github_actions/` サブフォルダに格納

## コーディング規約

### ワークフロー

- ファイル名: ケバブケース（例: `ci-build.yml`, `deploy-preview.yml`）
- 各ワークフローの先頭に `name:` を必ず設定する
- `on:` トリガーは用途に応じて最小限に絞る
- 実験的なワークフローには `workflow_dispatch` トリガーを付けて手動実行可能にする
- シークレットやトークンはハードコードせず `${{ secrets.XXX }}` を使う
- サードパーティアクションはSHAピン留め（`uses: actions/checkout@v4` ではなく `actions/checkout@<sha>`）を推奨
- パーミッションは最小権限で明示する（`permissions:` ブロック）
- 不要になったワークフローは削除する（無効化で放置しない）
