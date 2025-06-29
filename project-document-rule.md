# プロジェクトドキュメント更新ルール

## 概要

このドキュメントは、プロジェクトドキュメントを更新する際の統一ルールを定義します。
一貫性を保ち、品質を維持するため、全てのドキュメント作成・更新時にこのルールに従ってください。

## 更新フローの重要原則

### 1. 依存関係マトリックスに従った更新

ドキュメント間の依存関係を理解し、以下の順序で更新を行います：

```
要件定義 → 設計書 → 開発ドキュメント → 運用ドキュメント
```

上流から下流に向けて更新し、依存関係を正しく維持します。

### 2. メタデータの同期

更新時は必ずメタデータを同期更新します。

## 必須メタデータ項目

各ドキュメントには以下のメタデータを必ず含めます：

- **タイトル**: ドキュメントの正式名称
- **バージョン**: セマンティックバージョニング（例：1.0.0）
- **最終更新日**: YYYY-MM-DD形式
- **作成者**: 担当者名
- **レビュアー**: 確認者名（複数可）
- **関連ドキュメント**: 依存関係のあるドキュメントのリスト
- **ステータス**: draft / review / approved / deprecated

## 更新手順

### 1. 事前確認

- [ ] 影響範囲の特定
- [ ] 依存ドキュメントの確認
- [ ] 既存内容との整合性チェック

### 2. 更新作業

- [ ] ドキュメント内容の更新
- [ ] メタデータの更新
- [ ] リンクの検証
- [ ] フォーマットの統一

### 3. 品質チェック

- [ ] 内容の論理的一貫性
- [ ] 文章の明確性
- [ ] 技術的正確性
- [ ] 関連ドキュメントとの整合性

### 4. レビュー

- [ ] ピアレビューの実施
- [ ] フィードバックの反映
- [ ] 最終承認の取得

### 5. 公開・配布

- [ ] ステータスの更新（approved）
- [ ] 関係者への通知
- [ ] バージョン管理システムへのコミット

## ドキュメント品質基準

### 構造

- 明確な階層構造
- 論理的な章立て
- 適切な見出しレベル

### 内容

- 目的と対象読者の明記
- 具体的で実行可能な記述
- 例示やサンプルの提供

### 表現

- 統一された用語の使用
- 簡潔で理解しやすい文章
- 専門用語の定義や説明

## 自動化とチェック機能

### 1. 一貫性チェック

- ディレクトリ構造との整合性自動チェック
- ドキュメント間リンクの検証
- メタデータの妥当性確認

### 2. 更新履歴

- 更新履歴ログの自動生成
- 変更点の追跡
- バージョン管理

### 3. 不明瞭な項目の処理

- 未確定事項は [TODO] でマーキング
- 定期的な TODO 項目の見直し
- 課題管理システムとの連携

## 重要な考慮事項

### 依存関係の維持

- 常に依存関係を意識した更新
- 上流ドキュメントの変更影響を下流に伝播
- 循環依存の回避

### ディレクトリ構造の維持

- 決められた配置ルールに従う
- ファイル名の命名規則を遵守
- 不要ファイルの定期的な整理

### バックアップとバージョン管理

- 更新前の必須バックアップ
- 変更履歴の詳細記録
- ロールバック手順の確立

## 禁止事項

- [ ] メタデータなしでの公開
- [ ] 依存関係を無視した単独更新
- [ ] レビューなしでの重要ドキュメント更新
- [ ] リンク切れの放置
- [ ] TODO項目の長期間放置

## チェックリスト

### 更新前

- [ ] project-config.yamlの確認
- [ ] 影響範囲の特定
- [ ] 関連ドキュメントの最新版確認

### 更新中

- [ ] メタデータの同期更新
- [ ] フォーマットの統一
- [ ] リンクの妥当性確認

### 更新後

- [ ] 全体整合性の確認
- [ ] レビューの完了
- [ ] ステータス更新
- [ ] 関係者への通知

---

**注意**: このルールは全てのドキュメント作成・更新時に適用されます。不明な点がある場合は、プロジェクトマネージャーまたはドキュメント管理者に確認してください。