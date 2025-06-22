# CLAUDE.md

このファイルは、Claude Code (claude.ai/code) がこのリポジトリで作業する際のガイダンスを提供します。

## 言語とドキュメントポリシー

**重要**: このプロジェクトでは、すべてのドキュメント、コメント、プルリクエスト、コミットメッセージを日本語で作成してください。

- プルリクエストのタイトルと説明は日本語で記述する
- コミットメッセージは日本語で記述する
- ドキュメントファイル（README、CLAUDE.md等）は日本語で記述する
- コード内のコメントは日本語で記述する

### ドキュメント作成ルール

**必須**: すべてのドキュメント作成・更新時は `project-document-rule.md` に従ってください。

- ドキュメントの構造、メタデータ、品質基準に準拠する
- 更新フローと依存関係を遵守する
- 必須チェックリストを完了してから公開する
- 詳細は [project-document-rule.md](./project-document-rule.md) を参照

## 開発環境のセットアップ

### 環境設定

このリポジトリは機密設定に環境変数を使用します：

1. `.env.example` を `.env` にコピー：
   ```bash
   cp .env.example .env
   ```

2. 実際の認証情報で `.env` を編集：
   - `GITHUB_TOKEN`: あなたのGitHubパーソナルアクセストークン
   - `GITHUB_USERNAME`: あなたのGitHubユーザー名
   - `GITHUB_EMAIL`: あなたのGitHubメールアドレス
   - `PLAYWRIGHT_MCP_URL`: Playwright MCPサーバーのURL（デフォルト: http://localhost:5001/sse）
   - `PLAYWRIGHT_MCP_ENABLED`: Playwright MCP使用の有効/無効（デフォルト: true）

3. `.env` ファイルはセキュリティのためgitに自動的に無視されます。

## プロジェクト構造

このリポジトリは、PMBOKに基づいたプロジェクト管理ドキュメントの標準テンプレートを提供します：

### ディレクトリ構成

```
PMRoad/
├── 01_要件定義/              # 要件分析・定義フェーズ
│   ├── project-charter.md                    # プロジェクト憲章
│   ├── requirements-register.md              # 要求事項管理台帳
│   ├── scope-statement.md                    # スコープ定義書
│   ├── wbs.md                               # 作業分解構造
│   ├── functional-requirements.md           # 機能要件定義書
│   ├── non-functional-requirements.md       # 非機能要件定義書
│   └── requirements-traceability-matrix.md  # 要件トレーサビリティマトリックス
├── 02_設計書/                # システム設計フェーズ  
│   ├── system-architecture.md               # システム全体アーキテクチャ設計書
│   ├── system-design.md                     # システム方式設計書
│   ├── database-design.md                   # データベース設計書
│   ├── interface-design.md                  # インターフェース設計書
│   ├── security-design.md                   # セキュリティ設計書
│   └── performance-design.md                # 性能設計書
├── 03_開発ドキュメント/       # 開発実装フェーズ
│   └── README.md                            # 開発ドキュメント概要
├── 04_運用ドキュメント/       # 運用保守フェーズ
│   └── (今後作成予定)
├── 05_テスト/                # テスト・品質管理フェーズ
│   ├── README.md                            # テスト概要
│   └── test-strategy.md                     # テスト戦略書
├── project-document-rule.md  # ドキュメント作成統一ルール
├── .env.example              # 環境変数テンプレート
├── .gitignore               # Git除外設定
└── CLAUDE.md                # Claude Code向けガイダンス
```

### 各フェーズの概要

- **01_要件定義**: PMBOKのスコープ管理、要求事項管理に基づく要件定義関連文書
- **02_設計書**: PMBOKの品質管理に基づくシステム設計関連文書
- **03_開発ドキュメント**: PMBOKの品質管理に基づく開発実装関連文書
- **04_運用ドキュメント**: PMBOKのリスク管理、調達管理に基づく運用保守関連文書
- **05_テスト**: PMBOKの品質管理に基づくテスト・品質保証関連文書

## プルリクエストの作成

**重要**: 指示を正常に完了した際は毎回プルリクエストまで出してください。

プルリクエストを作成するには、環境が設定されていることを確認してください：

1. GitHub認証情報で `.env` ファイルをセットアップ（上記の環境設定を参照）
2. 認証に環境変数を使用：
   ```bash
   export GITHUB_TOKEN=$(grep GITHUB_TOKEN .env | cut -d '=' -f2)
   ```
3. プルリクエスト用の機能ブランチを作成してプッシュ
4. **マージ済みの古いブランチはついでに削除してください**

## テスト環境とPlaywright MCP

詳細なテスト実行手順とエビデンス収集ルールについては、[05_テスト/test-strategy.md](./05_テスト/test-strategy.md) を参照してください。

### 基本設定

**重要**: E2Eテスト実行時は必ずPlaywright MCPを使用してください。

- サーバー起動: `playwright-mcp-server --port 5001`
- 設定確認: `.env`ファイルで`PLAYWRIGHT_MCP_ENABLED=true`に設定
- エビデンス収集: Given-When-Thenパターンでスクリーンショット取得

## Claude Code向けの注意事項

- **PRを出す前に今回の修正がこのファイルに影響するかを確認し、必要に応じてこのファイルも更新してください**
- 必要な環境変数については常に `.env` を参照してください
- 実際の認証情報をgitリポジトリにコミットしないでください
- **E2Eテスト実行時は必ずPlaywright MCPを使用してください**
- **すべてのドキュメント、PR、コミットメッセージは日本語で記述してください**