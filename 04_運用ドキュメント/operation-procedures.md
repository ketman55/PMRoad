# 運用手順書

**タイトル**: 運用手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: system-architecture.md, monitoring-configuration.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの日常運用に必要な手順とチェックリストを定義し、安定したサービス提供を確保することを目的とする。

### 1.2 適用範囲
本運用手順は、システム運用に関わる全ての作業に適用される：
- 日常監視作業
- 定期メンテナンス作業
- バックアップ作業
- ログ監査作業

### 1.3 運用体制

#### 1.3.1 運用チーム構成
- **運用責任者**: システム全体の運用責任
- **運用エンジニア**: 日常運用・監視業務
- **保守エンジニア**: システム保守・改善業務
- **セキュリティ担当**: セキュリティ監視・対応

#### 1.3.2 勤務体制
| 時間帯 | 平日 | 土日祝 | 担当者 | 対応レベル |
|---|---|---|---|---|
| 09:00-18:00 | フル体制 | オンコール | 運用チーム全員 | フル対応 |
| 18:00-09:00 | オンコール | オンコール | 当番制 | 緊急対応のみ |

## 2. 日常運用手順

### 2.1 始業時チェック

#### 2.1.1 システム状態確認（毎日 9:00）

**チェック項目**
- [ ] 全サービスの稼働状況確認
- [ ] 夜間バッチ処理の完了確認
- [ ] アラート発生状況の確認
- [ ] システムリソース使用状況の確認
- [ ] セキュリティログの確認

**確認コマンド**
```bash
# サービス状態確認
systemctl status nginx
systemctl status application
systemctl status postgresql
systemctl status redis

# リソース確認
df -h
free -h
top -b -n1
```

#### 2.1.2 監視ダッシュボード確認

**確認項目**
- [ ] CPU使用率 < 80%
- [ ] メモリ使用率 < 80%
- [ ] ディスク使用率 < 85%
- [ ] ネットワーク使用率 < 70%
- [ ] アプリケーション応答時間 < 3秒
- [ ] データベース接続数 < 80%

**異常値発見時の対応**
1. 詳細ログの確認
2. 傾向分析の実施
3. 必要に応じて関係者への連絡
4. 改善施策の検討・実施

### 2.2 定期チェック

#### 2.2.1 時間別チェック（毎時）

**10:00, 14:00, 17:00 実施**
- [ ] サービス応答確認
- [ ] エラーログ確認
- [ ] パフォーマンス指標確認

```bash
# ヘルスチェック
curl -f https://api.example.com/health
curl -f https://web.example.com/health

# エラーログ確認
tail -n 100 /var/log/application/error.log | grep ERROR
tail -n 100 /var/log/nginx/error.log
```

#### 2.2.2 日次チェック（毎日 17:00）

**バックアップ状況確認**
- [ ] データベースバックアップの完了確認
- [ ] ファイルバックアップの完了確認
- [ ] バックアップファイルサイズの妥当性確認
- [ ] リモートバックアップの同期確認

```bash
# バックアップ確認
ls -la /backup/db/$(date +%Y%m%d)*
ls -la /backup/files/$(date +%Y%m%d)*

# バックアップサイズ確認
du -sh /backup/db/$(date +%Y%m%d)*
du -sh /backup/files/$(date +%Y%m%d)*
```

**ログローテーション確認**
- [ ] アプリケーションログのローテーション
- [ ] Webサーバーログのローテーション
- [ ] システムログのローテーション

### 2.3 週次チェック（毎週金曜日）

#### 2.3.1 システム全体チェック

**性能分析**
- [ ] 1週間の性能トレンド分析
- [ ] スロークエリの分析
- [ ] リソース使用状況の分析
- [ ] ユーザーアクセス傾向の分析

**セキュリティチェック**
- [ ] 不正アクセス試行の確認
- [ ] 脆弱性スキャン結果の確認
- [ ] アクセス権限の定期監査
- [ ] セキュリティパッチ適用状況の確認

```bash
# セキュリティログ分析
grep "Failed password" /var/log/auth.log | wc -l
grep "Invalid user" /var/log/auth.log | head -20
```

#### 2.3.2 データ整合性チェック

**データベース整合性**
- [ ] 外部キー制約の確認
- [ ] データ重複の確認
- [ ] 統計情報の更新

```sql
-- データベース整合性チェック
SELECT table_name, constraint_name 
FROM information_schema.table_constraints 
WHERE constraint_type = 'FOREIGN KEY';

-- 統計情報更新
ANALYZE;
```

### 2.4 月次チェック（毎月第1営業日）

#### 2.4.1 システム最適化

**データベース最適化**
- [ ] インデックスの使用状況分析
- [ ] テーブル統計の更新
- [ ] VACUUM処理の実行
- [ ] 不要データの削除

```sql
-- インデックス使用状況確認
SELECT schemaname, tablename, indexname, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_tup_read DESC;

-- VACUUM実行
VACUUM ANALYZE;
```

**ログファイル管理**
- [ ] 古いログファイルのアーカイブ
- [ ] ディスク容量の最適化
- [ ] ログ分析レポートの作成

#### 2.4.2 監査作業

**アクセス監査**
- [ ] 管理者アクセスログの確認
- [ ] 特権操作の監査
- [ ] 異常なアクセスパターンの分析

**設定監査**
- [ ] システム設定の変更履歴確認
- [ ] セキュリティ設定の妥当性確認
- [ ] 運用手順の適切性確認

## 3. メンテナンス手順

### 3.1 定期メンテナンス

#### 3.1.1 週次メンテナンス（毎週日曜日 2:00-4:00）

**実施内容**
1. システム全体の再起動
2. 不要プロセスの終了
3. 一時ファイルのクリーンアップ
4. ログファイルのローテーション

```bash
# メンテナンススクリプト例
#!/bin/bash
# weekly_maintenance.sh

echo "=== 週次メンテナンス開始 ===" | tee -a /var/log/maintenance.log

# サービス停止
systemctl stop application
systemctl stop nginx

# クリーンアップ
find /tmp -type f -mtime +7 -delete
find /var/log -name "*.log.1" -mtime +30 -delete

# サービス開始
systemctl start nginx
systemctl start application

# ヘルスチェック
sleep 30
curl -f https://api.example.com/health

echo "=== 週次メンテナンス完了 ===" | tee -a /var/log/maintenance.log
```

#### 3.1.2 月次メンテナンス（毎月第1日曜日 2:00-6:00）

**実施内容**
1. OS及びミドルウェアの更新
2. セキュリティパッチの適用
3. データベースの最適化
4. 監視設定の見直し

### 3.2 緊急メンテナンス

#### 3.2.1 緊急メンテナンス手順

**実施判断基準**
- システム障害の発生
- セキュリティインシデントの発生
- 重要な脆弱性の発見

**実施手順**
1. 影響範囲の特定
2. 関係者への連絡
3. メンテナンス内容の決定
4. 作業実施
5. 動作確認
6. 完了報告

## 4. バックアップ手順

### 4.1 日次バックアップ

#### 4.1.1 データベースバックアップ（毎日 1:00）

```bash
#!/bin/bash
# daily_db_backup.sh

BACKUP_DIR="/backup/db"
DB_NAME="main_db"
DATE=$(date +%Y%m%d)

# バックアップ実行
pg_dump -h localhost -U backup_user -d $DB_NAME > $BACKUP_DIR/backup_$DATE.sql

# 圧縮
gzip $BACKUP_DIR/backup_$DATE.sql

# 古いバックアップの削除（30日以上前）
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +30 -delete

# バックアップ結果の確認
if [ -f "$BACKUP_DIR/backup_$DATE.sql.gz" ]; then
    echo "バックアップ成功: $DATE" | tee -a /var/log/backup.log
else
    echo "バックアップ失敗: $DATE" | tee -a /var/log/backup.log
    # アラート送信
    mail -s "バックアップ失敗" admin@example.com < /dev/null
fi
```

#### 4.1.2 ファイルバックアップ（毎日 2:00）

```bash
#!/bin/bash
# daily_file_backup.sh

BACKUP_DIR="/backup/files"
SOURCE_DIR="/var/www/uploads"
DATE=$(date +%Y%m%d)

# rsyncでバックアップ
rsync -av --delete $SOURCE_DIR/ $BACKUP_DIR/current/

# 日次スナップショット作成
cp -al $BACKUP_DIR/current $BACKUP_DIR/snapshot_$DATE

# 古いスナップショットの削除（7日以上前）
find $BACKUP_DIR -name "snapshot_*" -mtime +7 -exec rm -rf {} \;
```

### 4.2 週次バックアップ

#### 4.2.1 システム全体バックアップ（毎週日曜日 0:00）

```bash
#!/bin/bash
# weekly_system_backup.sh

BACKUP_DIR="/backup/system"
DATE=$(date +%Y%m%d)

# システム設定のバックアップ
tar -czf $BACKUP_DIR/system_config_$DATE.tar.gz \
    /etc \
    /var/spool/cron \
    /home/*/.ssh

# アプリケーション設定のバックアップ
tar -czf $BACKUP_DIR/app_config_$DATE.tar.gz \
    /opt/application/config \
    /etc/nginx \
    /etc/postgresql

# リモートバックアップサーバーへの同期
rsync -av $BACKUP_DIR/ backup-server:/backup/remote/
```

## 5. 監視・アラート対応

### 5.1 アラート対応手順

#### 5.1.1 アラート分類

| レベル | 説明 | 対応時間 | 対応者 |
|---|---|---|---|
| Critical | サービス停止、データ消失リスク | 15分以内 | 運用責任者 |
| High | 性能劣化、部分機能停止 | 1時間以内 | 運用エンジニア |
| Medium | 軽微な問題、予防的対応 | 4時間以内 | 運用エンジニア |
| Low | 情報通知、計画的対応 | 24時間以内 | 運用チーム |

#### 5.1.2 Critical アラート対応

**CPU使用率 > 90%**
1. プロセス一覧の確認
2. リソース消費プロセスの特定
3. 必要に応じてプロセス終了
4. 根本原因の調査

```bash
# CPU使用率確認
htop
ps aux --sort=-%cpu | head -20

# 必要に応じてプロセス終了
kill -TERM <PID>
```

**メモリ使用率 > 95%**
1. メモリ使用状況の確認
2. メモリリークの可能性調査
3. 不要プロセスの終了
4. 必要に応じてサービス再起動

```bash
# メモリ使用確認
free -h
ps aux --sort=-%mem | head -20

# メモリ使用量上位プロセス確認
pmap -x <PID>
```

### 5.2 ログ監視

#### 5.2.1 エラーログ監視

**監視対象ログ**
- アプリケーションエラーログ
- Webサーバーエラーログ
- データベースエラーログ
- システムログ

```bash
# リアルタイムログ監視
tail -f /var/log/application/error.log | grep -i error
tail -f /var/log/nginx/error.log
tail -f /var/log/postgresql/postgresql.log | grep ERROR
```

#### 5.2.2 セキュリティログ監視

```bash
# 不正アクセス試行の監視
grep "Failed password" /var/log/auth.log | tail -20
grep "Invalid user" /var/log/auth.log | tail -20

# 異常なAPIアクセスの監視
awk '$9>=400 {print $1, $7, $9}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```

## 6. 報告・記録

### 6.1 日次報告

#### 6.1.1 日次運用報告書

**報告項目**
- システム稼働状況
- 発生したアラート・インシデント
- 実施した作業内容
- 注意事項・申し送り

**報告書テンプレート**
```
【日次運用報告書】
日付: YYYY/MM/DD
担当者: [担当者名]

■ システム稼働状況
- サービス稼働時間: XX時間XX分 (稼働率: XX.X%)
- 主要なアラート: [件数] 件
- パフォーマンス: 正常/要注意/異常

■ 実施作業
- 定期チェック作業
- [その他実施した作業]

■ 発生事象
- [発生したインシデント・問題]

■ 申し送り事項
- [翌日への申し送り]
```

### 6.2 月次報告

#### 6.2.1 月次運用レポート

**レポート内容**
- システム稼働率統計
- 性能トレンド分析
- インシデント発生状況
- 改善提案

---

**注意**: この運用手順書は、システムの安定運用のための重要な文書です。手順の変更時は十分な検証を行ってください。