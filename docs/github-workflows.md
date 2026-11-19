# GitHub Actions ワークフロー設定ドキュメント

`.github/workflows/` 以下に定義されているワークフローの設定内容をまとめます。

| ファイル | ワークフロー名 | 目的 |
|---|---|---|
| `claude.yml` | Claude Code | Issue / PR で `@claude` にメンションして作業を依頼する |
| `claude-code-review.yml` | Claude Code Review | PR の自動コードレビュー |
| `jekyll-gh-pages.yml` | Deploy Jekyll with GitHub Pages dependencies preinstalled | GitHub Pages へのサイト公開 |

## 共通事項

- Claude 系ワークフローは [`anthropics/claude-code-action@v1`](https://github.com/anthropics/claude-code-action) を使用する。
- 認証にはリポジトリシークレット `CLAUDE_CODE_OAUTH_TOKEN` が必要。
- Claude 系ワークフローは `id-token: write` 権限を要求する（OIDC 認証のため）。
- GitHub App の権限上、Claude は `.github/workflows` 配下のファイルを変更できない。

---

## 1. Claude Code (`claude.yml`)

`@claude` メンションをトリガーに、Claude が Issue / PR 上の依頼（質問回答、実装、レビューなど）を実行する。

### トリガー

| イベント | タイプ | 発火条件 |
|---|---|---|
| `issue_comment` | `created` | コメント本文に `@claude` を含む |
| `pull_request_review_comment` | `created` | コメント本文に `@claude` を含む |
| `pull_request_review` | `submitted` | レビュー本文に `@claude` を含む |
| `issues` | `opened`, `assigned` | Issue 本文またはタイトルに `@claude` を含む |

上記の条件は job レベルの `if:` で判定される。

### ジョブ: `claude`

- 実行環境: `ubuntu-latest`
- 権限（`permissions`）:

| 権限 | レベル | 用途 |
|---|---|---|
| `contents` | read | リポジトリの読み取り |
| `pull-requests` | read | PR 情報の読み取り |
| `issues` | read | Issue 情報の読み取り |
| `id-token` | write | OIDC 認証 |
| `actions` | read | PR 上の CI 結果の参照 |

### ステップ

1. **Checkout repository** — `actions/checkout@v4`（`fetch-depth: 1`）
2. **Run Claude Code** — `anthropics/claude-code-action@v1`
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN`
   - `additional_permissions`: `actions: read`（CI 結果の読み取り許可）

### 任意設定（コメントアウト中）

- `prompt`: 固定プロンプトを指定。未指定ならコメントの指示に従う。
- `claude_args`: 追加の CLI 引数（例: `--allowed-tools Bash(gh pr *)`）。

### 使い方

Issue や PR のコメントに `@claude` を付けて依頼する。例: `@claude この Issue を実装して`

---

## 2. Claude Code Review (`claude-code-review.yml`)

PR の作成・更新時に、Claude が自動でコードレビューを行いコメントを投稿する。

### トリガー

| イベント | タイプ |
|---|---|
| `pull_request` | `opened`, `synchronize`, `ready_for_review`, `reopened` |

任意設定として、`paths:` による対象ファイルの絞り込みがコメントアウトされている。

### ジョブ: `claude-review`

- 実行環境: `ubuntu-latest`
- 権限: `contents: read` / `pull-requests: read` / `issues: read` / `id-token: write`
- 任意設定（コメントアウト中）: PR 作成者による `if:` フィルタ（外部コントリビューターのみ等）

### ステップ

1. **Checkout repository** — `actions/checkout@v4`（`fetch-depth: 1`）
2. **Run Claude Code Review** — `anthropics/claude-code-action@v1`
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN`
   - `plugin_marketplaces`: `https://github.com/anthropics/claude-code.git`
   - `plugins`: `code-review@claude-code-plugins`
   - `prompt`: `/code-review:code-review --comment <repo>/pull/<PR番号>`
   - `claude_args`: `--allowedTools "mcp__github_inline_comment__create_inline_comment"`（インラインコメント投稿を許可）

---

## 3. Deploy Jekyll with GitHub Pages (`jekyll-gh-pages.yml`)

リポジトリ内容を Jekyll でビルドし、GitHub Pages に公開する。GitHub 提供のサンプルワークフローがベース。

### トリガー

| イベント | 条件 |
|---|---|
| `push` | `main` ブランチ |
| `workflow_dispatch` | Actions タブからの手動実行 |

### 権限（ワークフロー全体）

| 権限 | レベル | 用途 |
|---|---|---|
| `contents` | read | リポジトリの読み取り |
| `pages` | write | Pages へのデプロイ |
| `id-token` | write | デプロイ元の検証 |

### 同時実行制御

- `group: "pages"`、`cancel-in-progress: false`
- 同時にデプロイは 1 つだけ。実行中のデプロイはキャンセルせず、待機中の実行は最新のものだけが残る。

### ジョブ

**`build`**（`ubuntu-latest`）

1. `actions/checkout@v4`
2. `actions/configure-pages@v5` — Pages の設定
3. `actions/jekyll-build-pages@v1` — `source: ./` → `destination: ./_site`
4. `actions/upload-pages-artifact@v3` — 成果物のアップロード

**`deploy`**（`ubuntu-latest`、`needs: build`）

- `environment`: `github-pages`（URL は `steps.deployment.outputs.page_url`）
- `actions/deploy-pages@v5` — Pages へデプロイ

### 前提

リポジトリ設定の Pages > Source を「GitHub Actions」にしておく必要がある。
