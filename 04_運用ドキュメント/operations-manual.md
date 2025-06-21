# 運用マニュアル

**タイトル**: 運用マニュアル  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: 運用ドキュメント配下の全ドキュメント  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの日常運用から緊急時対応まで、包括的な運用手順とガイドラインを提供し、安定したサービス提供を実現することを目的とする。

### 1.2 適用範囲
本マニュアルは、システム運用に関わる全ての活動に適用される：
- 日常運用作業
- 監視・アラート対応
- 障害対応・復旧
- 性能管理・チューニング
- セキュリティ運用
- 変更管理
- バックアップ・リストア

### 1.3 運用体制

#### 1.3.1 組織構成

```
                    CTO
                     │
            ┌────────┼────────┐
            │                 │
       運用責任者         開発責任者
            │                 │
    ┌───────┼───────┐         │
    │               │         │
セキュリティ      運用チーム    開発チーム
 責任者            │
    │        ┌─────┼─────┐
    │        │           │
 セキュリティ   運用      インフラ
  チーム    エンジニア   エンジニア
```

#### 1.3.2 役割・責任

| 役割 | 責任範囲 | 主な業務 |
|---|---|---|
| **CTO** | システム全体統括 | 戦略決定、重要インシデント対応 |
| **運用責任者** | 運用業務統括 | 運用計画、SLA管理、エスカレーション |
| **セキュリティ責任者** | セキュリティ統括 | セキュリティ戦略、インシデント対応 |
| **運用エンジニア** | 日常運用業務 | 監視、障害対応、定期作業 |
| **インフラエンジニア** | インフラ管理 | サーバー管理、ネットワーク管理 |
| **セキュリティエンジニア** | セキュリティ運用 | セキュリティ監視、脆弱性対応 |

### 1.4 勤務体制

#### 1.4.1 運用時間

| 時間区分 | 勤務体制 | 対応レベル | 担当者 |
|---|---|---|---|
| **営業時間**<br>(平日 9:00-18:00) | フル体制 | 全対応 | 運用チーム全員 |
| **夜間・休日** | オンコール体制 | 緊急対応のみ | 当番制 |
| **祝日・年末年始** | 最小体制 | 重要対応のみ | 指定要員 |

#### 1.4.2 当番制度

**当番ローテーション**
- 平日夜間当番：週単位でローテーション
- 休日当番：月単位でローテーション  
- 重要システム当番：四半期単位でローテーション

**当番連絡先**
- 当番携帯：[電話番号]
- 緊急連絡用Slack：#ops-emergency
- エスカレーション先：運用責任者

## 2. 日常運用業務

### 2.1 日次作業

#### 2.1.1 始業時チェックリスト（毎日 9:00）

```
□ 夜間アラート確認
  - アラート一覧の確認
  - 未対応アラートの処理
  - エスカレーション要否判断

□ システム稼働状況確認
  - 全サービスの稼働確認
  - レスポンス時間確認
  - エラー率確認

□ バックアップ状況確認
  - 前夜バックアップの完了確認
  - バックアップファイルサイズ確認
  - エラーログ確認

□ セキュリティ状況確認
  - セキュリティアラート確認
  - 不正アクセス試行確認
  - 脆弱性情報確認

□ 性能状況確認
  - CPU・メモリ使用率確認
  - ディスク使用量確認
  - ネットワーク使用状況確認

□ 申し送り確認
  - 前日の申し送り事項確認
  - 当日の予定作業確認
  - 注意事項の把握
```

#### 2.1.2 定時チェック（10:00, 14:00, 17:00）

```bash
#!/bin/bash
# hourly_check.sh

echo "=== 定時チェック $(date) ==="

# 1. サービス稼働確認
echo "## サービス稼働確認"
for service in nginx application postgresql redis; do
    if systemctl is-active --quiet $service; then
        echo "✓ $service: 稼働中"
    else
        echo "✗ $service: 停止中"
        # アラート送信
        mail -s "Service Down: $service" admin@example.com
    fi
done

# 2. ヘルスチェック
echo "## ヘルスチェック"
if curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "✓ アプリケーション: 正常"
else
    echo "✗ アプリケーション: 異常"
fi

if curl -f http://localhost/health > /dev/null 2>&1; then
    echo "✓ Webサーバー: 正常"
else
    echo "✗ Webサーバー: 異常"
fi

# 3. リソース使用率確認
echo "## リソース使用率"
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.1f", ($3/$2) * 100.0}')
DISK_USAGE=$(df -h / | awk 'NR==2{print $5}' | sed 's/%//')

echo "CPU: $CPU_USAGE%"
echo "Memory: $MEMORY_USAGE%"
echo "Disk: $DISK_USAGE%"

# 閾値チェック
if (( $(echo "$CPU_USAGE > 80" | bc -l) )); then
    echo "⚠ CPU使用率が高くなっています"
fi

if (( $(echo "$MEMORY_USAGE > 85" | bc -l) )); then
    echo "⚠ メモリ使用率が高くなっています"
fi

if [ $DISK_USAGE -gt 85 ]; then
    echo "⚠ ディスク使用率が高くなっています"
fi

# 4. エラーログ確認
echo "## エラーログ確認"
ERROR_COUNT=$(grep -i error /var/log/application/*.log | grep "$(date +%Y-%m-%d)" | wc -l)
echo "本日のエラー数: $ERROR_COUNT"

if [ $ERROR_COUNT -gt 10 ]; then
    echo "⚠ エラー数が多くなっています"
    tail -5 /var/log/application/error.log
fi

echo "定時チェック完了: $(date)" >> /var/log/operations/hourly_check.log
```

#### 2.1.3 終業時処理（毎日 18:00）

```
□ 日次作業完了確認
  - 予定作業の完了確認
  - 未完了作業の翌日引き継ぎ

□ 当番引き継ぎ
  - 当日の重要事項まとめ
  - 夜間当番への申し送り
  - 緊急連絡先確認

□ 日次レポート作成
  - システム稼働状況
  - 発生したアラート・インシデント
  - 実施した作業内容

□ 翌日準備
  - 翌日の予定作業確認
  - 必要リソースの準備
  - 注意事項の確認
```

### 2.2 週次作業

#### 2.2.1 月曜日の作業

```
□ 週次計画確認
  - 週の作業予定確認
  - リソース配分計画
  - リスク要因の確認

□ セキュリティ週次チェック
  - 脆弱性情報確認
  - セキュリティパッチ確認
  - アクセス権限監査

□ 性能トレンド分析
  - 前週の性能データ分析
  - 傾向分析
  - 改善が必要な項目の特定
```

#### 2.2.2 水曜日の作業

```
□ バックアップ検証
  - バックアップファイルの整合性確認
  - 復元テスト実施
  - 保存期間チェック

□ 監視設定見直し
  - アラート設定の妥当性確認
  - 閾値調整の要否確認
  - 新しい監視項目の検討
```

#### 2.2.3 金曜日の作業

```
□ 週次レポート作成
  - 週間稼働状況まとめ
  - インシデント分析
  - 改善提案

□ 来週の準備
  - 来週の作業計画確認
  - 必要な事前準備
  - 休日当番への引き継ぎ
```

### 2.3 月次作業

#### 2.3.1 月初作業（毎月第1営業日）

```
□ 月次計画策定
  - 月間作業計画作成
  - メンテナンス計画確認
  - 変更計画確認

□ SLA達成状況確認
  - 前月のSLA達成状況分析
  - 未達成項目の原因分析
  - 改善計画策定

□ 容量計画見直し
  - リソース使用状況分析
  - 成長予測更新
  - スケールアウト計画見直し
```

#### 2.3.2 月次メンテナンス（毎月第1日曜日）

```
□ システム全体メンテナンス
  - OS・ミドルウェア更新
  - セキュリティパッチ適用
  - 設定最適化

□ データ整理
  - 古いログファイル削除
  - 不要データクリーンアップ
  - データベース最適化

□ 災害復旧テスト
  - バックアップ復元テスト
  - フェイルオーバーテスト
  - 手順書の妥当性確認
```

## 3. 監視・アラート対応

### 3.1 監視体系

#### 3.1.1 監視階層

```
┌─────────────────────────────────────────────────────────┐
│                  Level 1: 基本監視                      │
│  ・サーバー稼働監視  ・サービス稼働監視  ・基本リソース監視│
└─────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────┐
│                  Level 2: 性能監視                      │
│  ・応答時間監視    ・スループット監視    ・エラー率監視  │
└─────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────┐
│                  Level 3: 業務監視                      │
│  ・業務プロセス監視  ・データ整合性監視  ・SLA監視      │
└─────────────────────────────────────────────────────────┘
```

#### 3.1.2 アラート分類

| 重要度 | 対応時間 | 通知方法 | 対応者 |
|---|---|---|---|
| **Critical** | 15分以内 | 電話 + SMS + メール | 運用エンジニア + 責任者 |
| **High** | 1時間以内 | SMS + メール | 運用エンジニア |
| **Medium** | 4時間以内 | メール | 運用エンジニア |
| **Low** | 24時間以内 | メール | 担当者 |

### 3.2 アラート対応フロー

#### 3.2.1 アラート受信時の基本フロー

```
1. アラート受信
   ↓
2. 重要度確認（15秒以内）
   ↓
3. 初期確認（2分以内）
   ・サービス影響の確認
   ・類似アラートの確認
   ↓
4. 対応判断（5分以内）
   ・自動復旧の確認
   ・手動対応の要否判断
   ・エスカレーションの要否判断
   ↓
5. 対応実施
   ・問題解決
   ・または適切な担当者へエスカレーション
   ↓
6. 対応完了報告
   ・アラートクリア
   ・対応記録作成
```

#### 3.2.2 Critical アラート対応手順

```bash
#!/bin/bash
# critical_alert_response.sh

ALERT_ID="$1"
ALERT_TYPE="$2"
AFFECTED_SERVICE="$3"

echo "=== Critical アラート対応開始 ==="
echo "アラートID: $ALERT_ID"
echo "アラート種別: $ALERT_TYPE"
echo "影響サービス: $AFFECTED_SERVICE"
echo "対応開始時刻: $(date)"

# 1. 即座に状況確認
echo "## 状況確認"
case $ALERT_TYPE in
    "service_down")
        systemctl status $AFFECTED_SERVICE
        ;;
    "high_cpu")
        top -bn1 | head -20
        ;;
    "high_memory")
        free -h
        ps aux --sort=-%mem | head -10
        ;;
    "disk_full")
        df -h
        du -sh /* 2>/dev/null | sort -hr | head -10
        ;;
esac

# 2. 自動復旧試行
echo "## 自動復旧試行"
case $ALERT_TYPE in
    "service_down")
        echo "サービス再起動を試行します"
        systemctl restart $AFFECTED_SERVICE
        sleep 30
        if systemctl is-active --quiet $AFFECTED_SERVICE; then
            echo "サービス復旧成功"
            # アラートクリア処理
            curl -X POST "http://monitoring-server/api/alerts/$ALERT_ID/resolve"
            exit 0
        fi
        ;;
    "high_cpu"|"high_memory")
        echo "リソース使用量の詳細調査が必要です"
        ;;
    "disk_full")
        echo "ディスク容量の緊急クリーンアップを実行します"
        # 一時ファイル削除
        find /tmp -type f -mtime +1 -delete
        # ログローテーション強制実行
        logrotate -f /etc/logrotate.conf
        ;;
esac

# 3. エスカレーション（自動復旧失敗時）
echo "## エスカレーション"
echo "自動復旧に失敗しました。上位者にエスカレーションします。"

# 緊急連絡
mail -s "CRITICAL ALERT: $ALERT_TYPE on $AFFECTED_SERVICE" ops-manager@example.com << EOF
Critical アラートが発生し、自動復旧に失敗しました。

アラートID: $ALERT_ID
種別: $ALERT_TYPE
影響サービス: $AFFECTED_SERVICE
発生時刻: $(date)

緊急対応が必要です。
EOF

# Slack通知
curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"🚨 CRITICAL ALERT: $ALERT_TYPE on $AFFECTED_SERVICE - 緊急対応が必要です\"}" \
    $SLACK_WEBHOOK_URL

echo "対応記録作成中..."
echo "$(date): Critical alert $ALERT_ID - $ALERT_TYPE on $AFFECTED_SERVICE" >> /var/log/operations/critical_alerts.log
```

### 3.3 監視ツール運用

#### 3.3.1 Grafana ダッシュボード管理

**主要ダッシュボード**
- システム概要ダッシュボード：全体的な稼働状況
- アプリケーション監視ダッシュボード：アプリケーション固有メトリクス
- インフラ監視ダッシュボード：サーバー・ネットワーク状況
- セキュリティ監視ダッシュボード：セキュリティ関連イベント

**ダッシュボード更新手順**
1. 変更要求書の作成
2. テスト環境での動作確認
3. 承認後の本番反映
4. 動作確認・文書化

#### 3.3.2 Prometheus ルール管理

```yaml
# アラートルール管理例
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
          team: operations
        annotations:
          summary: "High CPU usage detected on {{ $labels.instance }}"
          description: "CPU usage is above 80% for more than 5 minutes"
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"
```

## 4. 障害対応

### 4.1 障害対応体制

#### 4.1.1 障害対応チーム編成

```
┌─────────────────┐
│ インシデント指揮官│  ← 全体指揮・意思決定
└─────────────────┘
         │
    ┌────┼────┐
    │         │
┌─────────┐ ┌─────────┐
│技術チーム │ │広報チーム │
└─────────┘ └─────────┘
    │
┌───┼───┐
│       │
運用    開発
担当    担当
```

#### 4.1.2 障害レベル定義と対応方針

| レベル | 定義 | 対応時間 | 体制 |
|---|---|---|---|
| **Lv1 (Critical)** | サービス全停止 | 15分以内 | 全チーム招集 |
| **Lv2 (High)** | 主要機能停止 | 1時間以内 | 技術チーム + 指揮官 |
| **Lv3 (Medium)** | 一部機能停止 | 4時間以内 | 運用チーム |
| **Lv4 (Low)** | 軽微な問題 | 24時間以内 | 担当者 |

### 4.2 障害対応手順

#### 4.2.1 初期対応（0-15分）

```
1. 障害検知・報告受付
   - アラート確認またはユーザー報告受付
   - 障害レベルの暫定判定

2. 初期影響調査
   - 影響範囲の特定
   - ユーザー影響の確認
   - 類似事例の確認

3. 対応体制確立
   - 対応チームの招集
   - 役割分担の決定
   - コミュニケーション手段の確立

4. 関係者への第一報
   - 内部関係者への通知
   - 顧客向け第一報の準備
```

#### 4.2.2 詳細調査・対応（15分-2時間）

```bash
#!/bin/bash
# incident_investigation.sh

INCIDENT_ID="$1"
INCIDENT_TYPE="$2"

echo "=== 障害詳細調査開始 ==="
echo "インシデントID: $INCIDENT_ID"
echo "障害種別: $INCIDENT_TYPE"

# 証跡保全
EVIDENCE_DIR="/var/log/incidents/$INCIDENT_ID"
mkdir -p $EVIDENCE_DIR

# システム状態保存
ps auxf > $EVIDENCE_DIR/processes.txt
netstat -tulpn > $EVIDENCE_DIR/network.txt
df -h > $EVIDENCE_DIR/disk.txt
free -h > $EVIDENCE_DIR/memory.txt
uptime > $EVIDENCE_DIR/uptime.txt

# ログ保存
cp /var/log/application/*.log $EVIDENCE_DIR/
cp /var/log/nginx/*.log $EVIDENCE_DIR/
cp /var/log/postgresql/*.log $EVIDENCE_DIR/

# 詳細調査実行
case $INCIDENT_TYPE in
    "application_error")
        echo "## アプリケーション障害調査"
        # アプリケーション固有の調査
        curl -v http://localhost:8080/health > $EVIDENCE_DIR/health_check.txt 2>&1
        jstack $(pgrep java) > $EVIDENCE_DIR/java_stack.txt 2>/dev/null
        ;;
    "database_issue")
        echo "## データベース障害調査"
        # データベース固有の調査
        psql -c "SELECT * FROM pg_stat_activity;" > $EVIDENCE_DIR/db_activity.txt
        psql -c "SELECT * FROM pg_locks;" > $EVIDENCE_DIR/db_locks.txt
        ;;
    "network_issue")
        echo "## ネットワーク障害調査"
        # ネットワーク固有の調査
        ping -c 10 8.8.8.8 > $EVIDENCE_DIR/ping_test.txt
        traceroute 8.8.8.8 > $EVIDENCE_DIR/traceroute.txt
        ;;
esac

echo "証跡保全完了: $EVIDENCE_DIR"
```

### 4.3 復旧作業

#### 4.3.1 段階的復旧アプローチ

```
Phase 1: 緊急措置
- サービス影響の最小化
- 暫定回避策の実施
- 二次被害の防止

Phase 2: 根本対応
- 根本原因の除去
- 正常サービスの復旧
- 動作確認

Phase 3: 安定化
- 監視強化
- 再発防止策の実施
- 正常性の継続確認
```

#### 4.3.2 復旧確認チェックリスト

```
□ サービス機能確認
  - 主要機能の動作確認
  - エラー率の正常化確認
  - レスポンス時間の正常化確認

□ データ整合性確認
  - データベース整合性確認
  - バックアップとの差分確認
  - 業務データの妥当性確認

□ 監視状況確認
  - アラートの正常化確認
  - メトリクスの正常値確認
  - ログエラーの収束確認

□ ユーザー影響確認
  - ユーザーアクセスの正常化
  - 業務プロセスの正常化
  - 顧客からの問い合わせ状況
```

## 5. 性能管理

### 5.1 性能監視運用

#### 5.1.1 日常的な性能確認

```bash
#!/bin/bash
# daily_performance_check.sh

echo "=== 日次性能確認 $(date) ==="

# 1. レスポンス時間確認
echo "## レスポンス時間"
for endpoint in "/health" "/api/users" "/api/orders"; do
    response_time=$(curl -o /dev/null -s -w "%{time_total}" "http://localhost:8080$endpoint")
    echo "$endpoint: ${response_time}s"
    
    if (( $(echo "$response_time > 3.0" | bc -l) )); then
        echo "⚠ $endpoint のレスポンス時間が遅いです"
    fi
done

# 2. スループット確認
echo "## スループット"
current_hour=$(date +%H)
requests_count=$(awk -v hour="$current_hour" '$4 ~ "\\[".*hour":" {print}' /var/log/nginx/access.log | wc -l)
echo "過去1時間のリクエスト数: $requests_count"

# 3. エラー率確認
echo "## エラー率"
total_requests=$(wc -l < /var/log/nginx/access.log)
error_requests=$(awk '$9 >= 400 {print}' /var/log/nginx/access.log | wc -l)

if [ $total_requests -gt 0 ]; then
    error_rate=$(echo "scale=2; $error_requests * 100 / $total_requests" | bc)
    echo "エラー率: $error_rate%"
    
    if (( $(echo "$error_rate > 1.0" | bc -l) )); then
        echo "⚠ エラー率が高くなっています"
    fi
fi

# 4. リソース使用状況
echo "## リソース使用状況"
echo "CPU: $(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')%"
echo "Memory: $(free | grep Mem | awk '{printf "%.1f", ($3/$2) * 100.0}')%"
echo "Disk: $(df -h / | awk 'NR==2{print $5}')"

echo "日次性能確認完了"
```

#### 5.1.2 性能劣化時の対応

```
1. 即座の影響確認
   - ユーザー影響の確認
   - サービス機能の確認

2. 原因調査
   - リソース使用状況の確認
   - スロークエリの確認
   - アプリケーションログの確認

3. 暫定対応
   - 負荷分散設定の調整
   - 不要プロセスの停止
   - キャッシュの活用

4. 根本対応
   - コードの最適化
   - インデックスの追加
   - インフラのスケールアップ

5. 継続監視
   - 改善効果の確認
   - 再発防止策の実施
```

### 5.2 キャパシティ管理

#### 5.2.1 容量監視項目

| 項目 | 監視頻度 | 警告閾値 | 限界閾値 | 対応アクション |
|---|---|---|---|---|
| CPU使用率 | 1分 | 70% | 85% | スケールアウト検討 |
| メモリ使用率 | 1分 | 80% | 90% | メモリ増設検討 |
| ディスク使用率 | 5分 | 85% | 95% | ディスク増設 |
| ネットワーク帯域 | 1分 | 70% | 85% | 帯域増強検討 |
| データベース接続数 | 1分 | 80% | 95% | 接続プール調整 |

#### 5.2.2 成長予測とプランニング

```python
#!/usr/bin/env python3
# capacity_growth_analysis.py

import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import matplotlib.pyplot as plt

def analyze_growth_trend():
    """成長トレンド分析"""
    
    # 過去3ヶ月のデータを使用（実際は監視システムから取得）
    dates = pd.date_range(start='2024-01-01', end='2024-03-31', freq='D')
    
    # サンプルデータ生成（実際の実装では実データを使用）
    cpu_usage = np.random.normal(50, 10, len(dates)) + np.linspace(0, 20, len(dates))
    memory_usage = np.random.normal(60, 15, len(dates)) + np.linspace(0, 15, len(dates))
    user_count = np.random.normal(1000, 100, len(dates)) + np.linspace(0, 500, len(dates))
    
    df = pd.DataFrame({
        'date': dates,
        'cpu_usage': np.clip(cpu_usage, 0, 100),
        'memory_usage': np.clip(memory_usage, 0, 100),
        'user_count': np.clip(user_count, 0, None)
    })
    
    # トレンド分析
    from sklearn.linear_model import LinearRegression
    
    X = np.arange(len(df)).reshape(-1, 1)
    
    for metric in ['cpu_usage', 'memory_usage', 'user_count']:
        model = LinearRegression()
        model.fit(X, df[metric])
        
        # 3ヶ月後の予測
        future_X = np.arange(len(df), len(df) + 90).reshape(-1, 1)
        future_values = model.predict(future_X)
        
        print(f"{metric}:")
        print(f"  現在の値: {df[metric].iloc[-1]:.1f}")
        print(f"  月次成長率: {model.coef_[0] * 30:.1f}")
        print(f"  3ヶ月後予測: {future_values[-1]:.1f}")
        
        # 閾値到達予測
        if metric in ['cpu_usage', 'memory_usage']:
            threshold = 80
            current_value = df[metric].iloc[-1]
            if model.coef_[0] > 0 and current_value < threshold:
                days_to_threshold = (threshold - current_value) / model.coef_[0]
                if days_to_threshold > 0:
                    print(f"  閾値({threshold}%)到達予測: {days_to_threshold:.0f}日後")
        
        print()

if __name__ == "__main__":
    analyze_growth_trend()
```

## 6. セキュリティ運用

### 6.1 日常セキュリティ業務

#### 6.1.1 セキュリティ監視

```bash
#!/bin/bash
# daily_security_monitoring.sh

echo "=== 日次セキュリティ監視 $(date) ==="

# 1. 認証ログ確認
echo "## 認証ログ分析"
echo "本日のログイン失敗数: $(grep 'Failed password' /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)"
echo "本日の不正ユーザー試行: $(grep 'Invalid user' /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)"

# 不審なアクセス元IP抽出
echo "## 不審なアクセス元IP"
grep 'Failed password' /var/log/auth.log | grep "$(date +%b' '%d)" | \
    awk '{print $11}' | sort | uniq -c | sort -nr | head -5

# 2. Webアクセスログ分析
echo "## Webアクセス異常"
echo "4xx/5xxエラー数: $(awk '$9>=400 {print}' /var/log/nginx/access.log | grep "$(date +%d/%b/%Y)" | wc -l)"

# 不審なUser-Agent検出
echo "## 不審なUser-Agent"
grep -i "bot\|crawler\|scanner" /var/log/nginx/access.log | grep "$(date +%d/%b/%Y)" | \
    awk '{print $1, $12}' | sort | uniq -c | sort -nr | head -5

# 3. システムレベルセキュリティ
echo "## システムセキュリティ"
echo "新規ユーザー作成: $(grep 'new user' /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)"
echo "sudo使用回数: $(grep 'sudo:' /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)"

# 4. セキュリティアラート生成
FAILED_LOGINS=$(grep 'Failed password' /var/log/auth.log | grep "$(date +%b' '%d)" | wc -l)
if [ $FAILED_LOGINS -gt 50 ]; then
    echo "⚠ セキュリティアラート: ログイン失敗数が異常に多いです ($FAILED_LOGINS回)"
    mail -s "Security Alert: High Failed Login Attempts" security@example.com << EOF
本日のログイン失敗数が異常に多くなっています。

失敗回数: $FAILED_LOGINS
日付: $(date)

詳細調査を実施してください。
EOF
fi

echo "日次セキュリティ監視完了"
```

#### 6.1.2 脆弱性管理

```
週次脆弱性チェック：
□ CVE情報の確認
□ 使用ソフトウェアの脆弱性確認
□ セキュリティパッチの適用計画
□ 脆弱性スキャン実行

月次セキュリティ監査：
□ アクセス権限の妥当性確認
□ 不要アカウントの削除
□ パスワードポリシーの確認
□ セキュリティ設定の見直し
```

### 6.2 インシデント対応

#### 6.2.1 セキュリティインシデント分類

| 分類 | 例 | 対応時間 | 対応チーム |
|---|---|---|---|
| **重大** | データ漏洩、システム侵害 | 15分以内 | 全セキュリティチーム |
| **高** | マルウェア感染、不正アクセス | 1時間以内 | セキュリティチーム |
| **中** | 脆弱性発見、不審なアクセス | 4時間以内 | セキュリティ担当者 |
| **低** | ポリシー違反、軽微な異常 | 24時間以内 | 担当者 |

#### 6.2.2 インシデント対応フロー

```
1. インシデント検知
   ↓
2. 初期評価（5分以内）
   - 影響範囲の特定
   - 重要度の判定
   ↓
3. 封じ込め（15分以内）
   - 拡散防止措置
   - 証跡保全
   ↓
4. 根絶・復旧
   - 脅威の除去
   - システム復旧
   ↓
5. 事後対応
   - 原因分析
   - 再発防止策
```

## 7. 変更管理

### 7.1 変更プロセス

#### 7.1.1 変更分類と承認フロー

```
緊急変更 (Emergency)
└─ 緊急対応後、事後承認

急用変更 (Urgent)  
└─ 簡略承認 → 24時間以内実施

標準変更 (Standard)
└─ 要求 → 承認 → 計画 → 実施 → 確認

定期変更 (Scheduled)
└─ 年間計画 → 月次計画 → 実施
```

#### 7.1.2 変更実施チェックリスト

```
実施前チェック：
□ 承認完了確認
□ バックアップ取得確認
□ ロールバック計画確認
□ 影響範囲最終確認
□ 実施チーム準備完了

実施中チェック：
□ 手順書通りの実施
□ 各ステップの確認
□ 異常時のロールバック判断
□ 進捗の記録

実施後チェック：
□ 機能確認
□ 性能確認
□ セキュリティ確認
□ ユーザー影響確認
□ 完了報告
```

## 8. バックアップ・復旧

### 8.1 バックアップ運用

#### 8.1.1 バックアップスケジュール

| 種別 | 頻度 | 実行時刻 | 保持期間 | 責任者 |
|---|---|---|---|---|
| データベースフルバックアップ | 日次 | 01:00 | 30日 | 運用チーム |
| データベース増分バックアップ | 時間次 | 毎時 | 7日 | 自動 |
| ファイルバックアップ | 日次 | 02:00 | 14日 | 運用チーム |
| システム設定バックアップ | 週次 | 日曜03:00 | 4週 | 運用チーム |

#### 8.1.2 バックアップ確認手順

```bash
#!/bin/bash
# backup_verification.sh

echo "=== バックアップ確認 $(date) ==="

# 1. バックアップファイル存在確認
echo "## バックアップファイル確認"
backup_date=$(date +%Y%m%d)

db_backup="/backup/db/full_backup_${backup_date}.dump.gz"
file_backup="/backup/files/backup_${backup_date}"
system_backup="/backup/system/system_backup_${backup_date}.tar.gz"

for backup_file in "$db_backup" "$file_backup" "$system_backup"; do
    if [ -e "$backup_file" ]; then
        echo "✓ $(basename $backup_file): 存在"
        echo "  サイズ: $(du -h "$backup_file" | cut -f1)"
        echo "  作成時刻: $(stat -c %y "$backup_file")"
    else
        echo "✗ $(basename $backup_file): 存在しない"
    fi
done

# 2. チェックサム確認
echo "## チェックサム確認"
for backup_file in "$db_backup" "$file_backup" "$system_backup"; do
    if [ -e "$backup_file" ] && [ -e "$backup_file.sha256" ]; then
        if sha256sum -c "$backup_file.sha256" > /dev/null 2>&1; then
            echo "✓ $(basename $backup_file): チェックサム正常"
        else
            echo "✗ $(basename $backup_file): チェックサム異常"
        fi
    fi
done

# 3. リモートバックアップ同期確認
echo "## リモートバックアップ確認"
if rsync -n --delete backup-server:/backup/remote/ /backup/local/ | grep -q "deleting\|>"; then
    echo "✓ リモートバックアップ同期確認"
else
    echo "⚠ リモートバックアップ同期に問題がある可能性があります"
fi

echo "バックアップ確認完了"
```

### 8.2 復旧手順

#### 8.2.1 災害復旧計画

```
RTO (Recovery Time Objective):
- Critical システム: 2時間
- High システム: 4時間
- Medium システム: 8時間
- Low システム: 24時間

RPO (Recovery Point Objective):
- Critical データ: 1時間
- High データ: 4時間
- Medium データ: 24時間
- Low データ: 48時間
```

#### 8.2.2 復旧手順テスト

```
月次復旧テスト：
□ バックアップからの部分復旧テスト
□ データベース復旧テスト
□ アプリケーション復旧テスト
□ 手順書の妥当性確認

四半期災害復旧テスト：
□ 完全災害を想定した復旧テスト
□ 代替サイトでの復旧テスト
□ 復旧時間の測定
□ 手順の改善
```

## 9. トラブルシューティング

### 9.1 よくある問題と対処法

#### 9.1.1 システム系問題

| 症状 | 考えられる原因 | 対処法 |
|---|---|---|
| サーバーにアクセスできない | ネットワーク障害、サーバー停止 | ping確認、サーバー状態確認 |
| アプリケーションが応答しない | プロセス停止、メモリ不足 | プロセス確認、リソース確認 |
| データベース接続エラー | DB停止、接続数上限 | DB状態確認、接続数確認 |
| ディスク容量不足 | ログ蓄積、一時ファイル | 不要ファイル削除、ログローテーション |

#### 9.1.2 性能系問題

| 症状 | 考えられる原因 | 対処法 |
|---|---|---|
| 応答時間が遅い | 高負荷、スロークエリ | 負荷状況確認、クエリ分析 |
| CPU使用率が高い | 無限ループ、高負荷処理 | プロセス確認、負荷分散 |
| メモリ使用率が高い | メモリリーク、大量データ処理 | メモリ使用状況確認、プロセス再起動 |
| ネットワークが遅い | 帯域不足、パケットロス | ネットワーク状況確認、経路確認 |

### 9.2 診断コマンド集

#### 9.2.1 システム診断

```bash
# CPU・メモリ・プロセス確認
top
htop
ps aux --sort=-%cpu | head -20
ps aux --sort=-%mem | head -20

# ディスク・I/O確認
df -h
du -sh /* 2>/dev/null | sort -hr
iostat -x 1 5
iotop

# ネットワーク確認
netstat -tulpn
ss -tulpn
ifconfig
ip addr show
ping -c 5 target
traceroute target

# ログ確認
tail -f /var/log/messages
tail -f /var/log/syslog
journalctl -f
dmesg | tail -20
```

#### 9.2.2 アプリケーション診断

```bash
# Java アプリケーション
jps
jstat -gc [PID]
jmap -histo [PID]
jstack [PID]

# Web サーバー
nginx -t
apache2ctl -t
curl -I http://localhost
ab -n 100 -c 10 http://localhost/

# データベース
psql -c "SELECT * FROM pg_stat_activity;"
psql -c "SELECT * FROM pg_locks;"
mysql -e "SHOW PROCESSLIST;"
mysql -e "SHOW ENGINE INNODB STATUS;"
```

## 10. 連絡先・エスカレーション

### 10.1 緊急連絡先

#### 10.1.1 内部連絡先

| 役割 | 氏名 | 内線 | 携帯 | メール |
|---|---|---|---|---|
| CTO | [氏名] | [内線] | [携帯] | cto@example.com |
| 運用責任者 | [氏名] | [内線] | [携帯] | ops-manager@example.com |
| セキュリティ責任者 | [氏名] | [内線] | [携帯] | security-manager@example.com |
| 当番（平日夜間） | 確認要 | - | [当番携帯] | ops-oncall@example.com |
| 当番（休日） | 確認要 | - | [当番携帯] | ops-weekend@example.com |

#### 10.1.2 外部連絡先

| 区分 | 会社名 | 担当者 | 電話番号 | 対応時間 |
|---|---|---|---|---|
| データセンター | [DC会社] | [担当者] | [電話番号] | 24時間 |
| ISP | [ISP会社] | [担当者] | [電話番号] | 24時間 |
| ハードウェアベンダー | [HW会社] | [担当者] | [電話番号] | 営業時間+緊急対応 |
| ソフトウェアベンダー | [SW会社] | [担当者] | [電話番号] | 営業時間 |

### 10.2 エスカレーション基準

#### 10.2.1 時間ベースエスカレーション

```
Level 1: 運用エンジニア (0-30分)
└─ 30分で解決しない場合

Level 2: 運用責任者 (30分-2時間)  
└─ 2時間で解決しない場合

Level 3: CTO (2時間-)
└─ 経営判断が必要な場合
```

#### 10.2.2 影響ベースエスカレーション

```
Critical: 即座にLevel 3まで通知
High: 30分以内にLevel 2まで通知  
Medium: 2時間以内にLevel 2まで通知
Low: 翌営業日にLevel 1で対応
```

---

**注意**: この運用マニュアルは、システム運用の包括的なガイドとして作成されています。実際の運用環境に合わせて内容を調整し、定期的に更新してください。