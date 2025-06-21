# 障害対応手順書

**タイトル**: 障害対応手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: operation-procedures.md, monitoring-configuration.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システム障害発生時の対応手順を定義し、迅速かつ適切な障害対応によりサービス停止時間を最小化することを目的とする。

### 1.2 適用範囲
本手順は、以下の障害に適用される：
- システム障害（ハードウェア・ソフトウェア）
- ネットワーク障害
- セキュリティインシデント
- データ破損・消失

### 1.3 基本方針
- **迅速性**: 障害検知から対応開始まで最短時間で実施
- **安全性**: 二次被害を防ぐ安全な対応手順
- **透明性**: 関係者への適切な情報共有
- **継続改善**: 障害原因分析と再発防止策の実施

## 2. 障害対応体制

### 2.1 対応チーム構成

#### 2.1.1 役割分担

| 役割 | 責任者 | 主な業務 | 連絡先 |
|---|---|---|---|
| **インシデント指揮官** | 運用責任者 | 全体指揮・意思決定 | [電話番号] |
| **技術リーダー** | 技術責任者 | 技術的判断・復旧作業指示 | [電話番号] |
| **運用エンジニア** | 運用担当者 | 障害対応・復旧作業 | [電話番号] |
| **コミュニケーター** | 広報担当者 | 情報発信・顧客対応 | [電話番号] |

#### 2.1.2 エスカレーション体制

```
┌─────────────────┐
│ Level 1         │ ← 初期対応（15分以内）
│ 運用エンジニア   │
└─────────────────┘
         │ 30分経過/重大障害
         ▼
┌─────────────────┐
│ Level 2         │ ← 技術対応（1時間以内）
│ 技術リーダー     │
└─────────────────┘
         │ 2時間経過/サービス停止
         ▼
┌─────────────────┐
│ Level 3         │ ← 経営判断（即座）
│ インシデント指揮官│
└─────────────────┘
```

### 2.2 連絡体制

#### 2.2.1 緊急連絡先

**24時間対応**
- 運用当番: [電話番号]
- インシデント指揮官: [電話番号]
- 技術リーダー: [電話番号]

**営業時間内対応**
- 運用チーム: [内線番号]
- 開発チーム: [内線番号]
- インフラチーム: [内線番号]

#### 2.2.2 外部連絡先

- **データセンター**: [電話番号]
- **ISP**: [電話番号]
- **ベンダー（サーバー）**: [電話番号]
- **ベンダー（ネットワーク）**: [電話番号]

## 3. 障害レベル分類

### 3.1 障害レベル定義

| レベル | 定義 | 影響範囲 | 対応時間 | 報告先 |
|---|---|---|---|---|
| **Critical** | サービス全停止、データ消失リスク | 全ユーザー | 15分以内 | 経営陣・全関係者 |
| **High** | 主要機能停止、重大な性能劣化 | 大部分のユーザー | 1時間以内 | 部門責任者・関係者 |
| **Medium** | 一部機能停止、軽微な性能劣化 | 一部のユーザー | 4時間以内 | チームリーダー |
| **Low** | 軽微な問題、将来的リスク | 限定的影響 | 24時間以内 | 担当者 |

### 3.2 障害分類マトリックス

| 影響度＼緊急度 | 高 | 中 | 低 |
|---|---|---|---|
| **高** | Critical | High | High |
| **中** | High | Medium | Medium |
| **低** | Medium | Low | Low |

## 4. 障害対応フロー

### 4.1 基本対応フロー

```
┌─────────────┐
│ 障害検知     │
└─────────────┘
       │
       ▼
┌─────────────┐
│ 初期対応     │ ← 15分以内
│ ・影響確認   │
│ ・関係者連絡 │
└─────────────┘
       │
       ▼
┌─────────────┐
│ 詳細調査     │ ← 30分以内
│ ・原因特定   │
│ ・対策検討   │
└─────────────┘
       │
       ▼
┌─────────────┐
│ 復旧作業     │ ← SLA に基づく
│ ・暫定対応   │
│ ・本復旧     │
└─────────────┘
       │
       ▼
┌─────────────┐
│ 事後対応     │ ← 24時間以内
│ ・原因分析   │
│ ・再発防止   │
└─────────────┘
```

### 4.2 詳細対応手順

#### 4.2.1 障害検知時の初期対応（0-15分）

**1. 障害確認**
```bash
# サービス稼働確認
systemctl status application
systemctl status nginx
systemctl status postgresql

# プロセス確認
ps aux | grep -E "(java|nginx|postgres)"

# ログ確認
tail -n 50 /var/log/application/error.log
tail -n 50 /var/log/nginx/error.log
```

**2. 影響範囲特定**
- [ ] 影響を受けるサービス・機能の特定
- [ ] 影響を受けるユーザー数の推定
- [ ] 継続時間の推定

**3. 障害レベル判定**
- [ ] 障害レベルの判定（Critical/High/Medium/Low）
- [ ] SLA目標時間の確認

**4. 関係者への第一報**
```
件名: 【障害発生】[レベル] システム障害が発生しました

発生時刻: YYYY/MM/DD HH:MM
障害レベル: [Critical/High/Medium/Low]
影響範囲: [具体的な影響内容]
推定復旧時間: [XX時間後]
担当者: [担当者名]

現在調査中です。詳細は30分後に報告いたします。
```

#### 4.2.2 詳細調査（15-45分）

**1. 根本原因分析**
```bash
# システムリソース確認
df -h
free -h
iostat -x 1 5
netstat -tuln

# ログ詳細分析
grep -i error /var/log/application/*.log | tail -100
grep -i error /var/log/nginx/*.log | tail -100
grep -i error /var/log/postgresql/*.log | tail -100

# 監視データ確認
# Grafanaダッシュボードで障害発生前後の状況確認
```

**2. 影響範囲詳細調査**
- [ ] データ整合性の確認
- [ ] セキュリティへの影響確認
- [ ] 他システムへの影響確認

**3. 対策方針決定**
- [ ] 暫定対応の検討
- [ ] 本復旧の計画立案
- [ ] 必要リソースの確保

#### 4.2.3 復旧作業

**暫定対応（緊急性が高い場合）**
```bash
# サービス再起動
sudo systemctl restart application
sudo systemctl restart nginx

# 負荷分散設定変更（障害サーバーの切り離し）
# nginx.conf の upstream 設定変更
sudo nginx -s reload

# データベース切り替え（スレーブへのフェイルオーバー）
sudo pg_ctl promote -D /var/lib/postgresql/data
```

**本復旧**
- [ ] 根本原因の除去
- [ ] 正常なサービス復旧の確認
- [ ] データ整合性の確認
- [ ] 性能・機能テストの実施

## 5. 障害パターン別対応手順

### 5.1 Webサーバー障害

#### 5.1.1 Nginx停止

**症状**
- Webサイトにアクセスできない
- 502 Bad Gateway エラー

**対応手順**
```bash
# 1. サービス状態確認
systemctl status nginx

# 2. 設定ファイル確認
nginx -t

# 3. ログ確認
tail -f /var/log/nginx/error.log

# 4. サービス再起動
systemctl restart nginx

# 5. 動作確認
curl -I http://localhost
```

#### 5.1.2 高負荷によるレスポンス遅延

**症状**
- レスポンス時間の著しい増加
- タイムアウトエラーの頻発

**対応手順**
```bash
# 1. 負荷状況確認
htop
iostat -x 1 5

# 2. プロセス確認
ps aux --sort=-%cpu | head -20
ps aux --sort=-%mem | head -20

# 3. ネットワーク接続確認
netstat -an | grep :80 | wc -l

# 4. 暫定対応（必要に応じて）
# - 不要プロセスの終了
# - 接続数制限の設定
# - キャッシュの有効化
```

### 5.2 アプリケーションサーバー障害

#### 5.2.1 アプリケーションプロセス停止

**症状**
- 502/503 エラーの発生
- アプリケーションからのレスポンスなし

**対応手順**
```bash
# 1. プロセス確認
ps aux | grep java

# 2. ログ確認
tail -f /var/log/application/application.log
tail -f /var/log/application/error.log

# 3. ヒープダンプ取得（OOMError の場合）
jmap -dump:format=b,file=/tmp/heapdump.hprof <PID>

# 4. アプリケーション再起動
systemctl restart application

# 5. 動作確認
curl -f http://localhost:8080/health
```

#### 5.2.2 メモリ不足（OutOfMemoryError）

**症状**
- OutOfMemoryError の発生
- アプリケーションの応答停止

**対応手順**
```bash
# 1. メモリ使用状況確認
free -h
cat /proc/meminfo

# 2. JVMヒープ状況確認
jstat -gc <PID>
jstat -gccapacity <PID>

# 3. ヒープダンプ取得
jmap -dump:live,format=b,file=/tmp/heapdump.hprof <PID>

# 4. 暫定対応
# - JVMヒープサイズの増加
# - 不要プロセスの終了
# - アプリケーション再起動

# 5. 長期対応
# - メモリリーク箇所の特定
# - コードの最適化
```

### 5.3 データベース障害

#### 5.3.1 PostgreSQL 接続不可

**症状**
- データベースに接続できない
- Connection refused エラー

**対応手順**
```bash
# 1. PostgreSQLプロセス確認
ps aux | grep postgres
systemctl status postgresql

# 2. ログ確認
tail -f /var/log/postgresql/postgresql.log

# 3. 接続確認
psql -h localhost -U postgres -c "SELECT version();"

# 4. 設定確認
postgres -D /var/lib/postgresql/data --config-file=/etc/postgresql/postgresql.conf

# 5. サービス再起動
systemctl restart postgresql

# 6. データ整合性確認
psql -c "SELECT pg_is_in_recovery();"
```

#### 5.3.2 データベースロック

**症状**
- クエリの応答時間増加
- タイムアウトエラーの発生

**対応手順**
```sql
-- 1. ロック状況確認
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 2. 長時間実行中のクエリ確認
SELECT pid, now() - pg_stat_activity.query_start AS duration, query 
FROM pg_stat_activity 
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';

-- 3. 必要に応じてセッション終了
SELECT pg_terminate_backend(<PID>);
```

### 5.4 ネットワーク障害

#### 5.4.1 外部ネットワーク通信不可

**症状**
- 外部APIへの接続不可
- DNS名前解決の失敗

**対応手順**
```bash
# 1. ネットワーク接続確認
ping 8.8.8.8
ping google.com

# 2. DNS確認
nslookup google.com
dig google.com

# 3. ルーティング確認
route -n
traceroute 8.8.8.8

# 4. ファイアウォール確認
iptables -L -n
ufw status

# 5. ネットワーク設定確認
ip addr show
ip route show
```

## 6. セキュリティインシデント対応

### 6.1 不正アクセス検知

#### 6.1.1 初期対応

**1. アクセス遮断**
```bash
# 不正IPアドレスのブロック
iptables -A INPUT -s <不正IP> -j DROP

# Webアプリケーションファイアウォール設定更新
# WAFルールの緊急追加
```

**2. 影響範囲調査**
```bash
# アクセスログ分析
grep "<不正IP>" /var/log/nginx/access.log
grep "POST" /var/log/nginx/access.log | grep "<不正IP>"

# セキュリティログ確認
grep "Failed password" /var/log/auth.log
grep "Invalid user" /var/log/auth.log
```

**3. データ漏洩調査**
- [ ] アクセスされたデータの特定
- [ ] ダウンロードされたファイルの確認
- [ ] データベースアクセス履歴の確認

### 6.2 マルウェア感染

#### 6.2.1 隔離措置

```bash
# 1. ネットワークからの隔離
iptables -P INPUT DROP
iptables -P OUTPUT DROP
iptables -P FORWARD DROP

# 2. プロセス確認・停止
ps aux | grep -E "(suspicious|malware)"
kill -9 <suspicious_PID>

# 3. ファイルシステム確認
find / -name "*.suspicious" -type f
find / -newermt "2024-01-01" -type f | grep -v /proc | grep -v /sys
```

## 7. 復旧後手順

### 7.1 サービス正常性確認

#### 7.1.1 機能テスト

**基本機能テスト**
- [ ] ユーザーログイン機能
- [ ] 主要業務機能
- [ ] データ更新機能
- [ ] 外部システム連携

**性能テスト**
- [ ] レスポンス時間確認
- [ ] 同時接続数確認
- [ ] スループット確認

```bash
# 自動テスト実行例
curl -f http://localhost/health
curl -f http://localhost/api/users/1
ab -n 100 -c 10 http://localhost/
```

### 7.2 事後処理

#### 7.2.1 ログ保全

```bash
# 障害時のログ保存
mkdir -p /backup/incident/$(date +%Y%m%d)
cp /var/log/application/*.log /backup/incident/$(date +%Y%m%d)/
cp /var/log/nginx/*.log /backup/incident/$(date +%Y%m%d)/
cp /var/log/postgresql/*.log /backup/incident/$(date +%Y%m%d)/
```

#### 7.2.2 関係者報告

**復旧完了報告**
```
件名: 【復旧完了】システム障害が復旧しました

復旧時刻: YYYY/MM/DD HH:MM
停止時間: XX時間XX分
根本原因: [原因の詳細]
暫定対応: [実施した暫定対応]
本復旧: [実施した本復旧]
再発防止策: [予定している対策]
```

### 7.3 事後分析・改善

#### 7.3.1 ポストモーテム会議

**検討項目**
- [ ] 障害の根本原因分析
- [ ] 対応手順の妥当性評価
- [ ] 検知・通知の適切性評価
- [ ] 復旧時間の分析

#### 7.3.2 改善策実施

**技術的改善**
- [ ] システム・プロセスの改善
- [ ] 監視・アラートの改善
- [ ] 自動化の拡充

**運用的改善**
- [ ] 手順書の更新
- [ ] 訓練の実施
- [ ] 体制の見直し

---

**注意**: この障害対応手順書は、迅速な障害復旧のための重要な文書です。定期的な訓練と手順の見直しを実施してください。