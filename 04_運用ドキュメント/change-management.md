# 変更管理手順書

**タイトル**: 変更管理手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: operation-procedures.md, incident-response-procedures.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの変更を安全かつ計画的に実施するための管理手順を定義し、変更によるリスクを最小化することを目的とする。

### 1.2 適用範囲
本手順は、以下の変更に適用される：
- システム設定変更
- アプリケーション更新・デプロイ
- インフラストラクチャ変更
- セキュリティ設定変更

### 1.3 変更管理方針
- **計画的実施**: 事前計画に基づく段階的変更
- **リスク評価**: 変更影響の事前評価
- **承認プロセス**: 適切な権限者による承認
- **ロールバック計画**: 変更失敗時の復旧計画

## 2. 変更分類

### 2.1 変更カテゴリ

#### 2.1.1 影響度による分類

| 影響度 | 定義 | 例 | 承認者 |
|---|---|---|---|
| **Critical** | システム全体に重大な影響 | OS更新、DB構成変更 | CTO + 運用責任者 |
| **High** | 重要な機能に影響 | アプリケーション機能追加 | 運用責任者 + 開発責任者 |
| **Medium** | 限定的な機能に影響 | 設定ファイル変更 | 運用責任者 |
| **Low** | 最小限の影響 | ログ設定変更 | 運用エンジニア |

#### 2.1.2 緊急度による分類

| 緊急度 | 定義 | 実施タイミング | 承認プロセス |
|---|---|---|---|
| **Emergency** | 緊急対応が必要 | 即座 | 事後承認可 |
| **Urgent** | 24時間以内に実施 | 計画実施 | 簡略承認 |
| **Standard** | 通常計画で実施 | 計画実施 | 標準承認 |
| **Scheduled** | 定期メンテナンス | メンテナンス窓 | 標準承認 |

### 2.2 変更タイプ

#### 2.2.1 アプリケーション変更

**対象**
- アプリケーションコード
- 設定ファイル
- データベーススキーマ
- APIインターフェース

**承認フロー**
```
開発者 → 開発リーダー → 運用責任者 → 実施
```

#### 2.2.2 インフラ変更

**対象**
- サーバー設定
- ネットワーク設定
- ミドルウェア設定
- 監視設定

**承認フロー**
```
インフラエンジニア → インフラ責任者 → 運用責任者 → 実施
```

#### 2.2.3 セキュリティ変更

**対象**
- セキュリティ設定
- アクセス制御
- 認証設定
- 暗号化設定

**承認フロー**
```
セキュリティエンジニア → セキュリティ責任者 → CTO → 実施
```

## 3. 変更要求プロセス

### 3.1 変更要求書

#### 3.1.1 変更要求書テンプレート

```
=== 変更要求書 ===

■ 基本情報
変更ID: CHG-YYYYMMDD-XXX
要求者: [要求者名]
要求日: YYYY/MM/DD
実施希望日: YYYY/MM/DD
緊急度: [Emergency/Urgent/Standard/Scheduled]
影響度: [Critical/High/Medium/Low]

■ 変更内容
変更タイトル: [変更の概要]
変更対象: [対象システム・コンポーネント]
変更理由: [変更が必要な理由]
変更内容詳細:
[具体的な変更内容の詳細記述]

■ 影響分析
影響範囲: [影響を受けるシステム・サービス・ユーザー]
停止時間: [予想される停止時間]
影響開始時刻: [影響が開始される時刻]
影響終了時刻: [影響が終了する予定時刻]

■ リスク評価
技術的リスク: [技術的な問題発生の可能性]
ビジネスリスク: [業務への影響]
セキュリティリスク: [セキュリティへの影響]
リスク軽減策: [リスク軽減のための対策]

■ 実施計画
実施手順:
1. [手順1]
2. [手順2]
3. [手順3]

必要リソース: [人員・時間・設備]
事前準備: [事前に必要な準備]
テスト計画: [変更後のテスト方法]

■ ロールバック計画
ロールバック条件: [ロールバックが必要な状況]
ロールバック手順:
1. [ロールバック手順1]
2. [ロールバック手順2]
3. [ロールバック手順3]

ロールバック所要時間: [予想される時間]

■ 承認
開発責任者: [署名] [日付]
運用責任者: [署名] [日付]
CTO: [署名] [日付] (Critical/High影響度の場合)
```

### 3.2 変更承認フロー

#### 3.2.1 標準承認フロー

```bash
#!/bin/bash
# change_approval_workflow.sh

CHANGE_ID="$1"
CHANGE_FILE="$2"

if [ -z "$CHANGE_ID" ] || [ -z "$CHANGE_FILE" ]; then
    echo "Usage: $0 <change_id> <change_file>"
    exit 1
fi

echo "=== 変更承認ワークフロー開始 ==="
echo "変更ID: $CHANGE_ID"
echo "変更要求書: $CHANGE_FILE"

# 変更ディレクトリ作成
CHANGE_DIR="/var/log/changes/$CHANGE_ID"
mkdir -p $CHANGE_DIR
cp "$CHANGE_FILE" $CHANGE_DIR/

# 変更内容解析
IMPACT=$(grep "影響度:" "$CHANGE_FILE" | cut -d: -f2 | xargs)
URGENCY=$(grep "緊急度:" "$CHANGE_FILE" | cut -d: -f2 | xargs)

echo "影響度: $IMPACT"
echo "緊急度: $URGENCY"

# 承認フロー決定
case $IMPACT in
    "Critical")
        APPROVERS="dev_manager ops_manager cto"
        ;;
    "High")
        APPROVERS="dev_manager ops_manager"
        ;;
    "Medium")
        APPROVERS="ops_manager"
        ;;
    "Low")
        APPROVERS="ops_engineer"
        ;;
    *)
        echo "Error: Invalid impact level"
        exit 1
        ;;
esac

echo "必要な承認者: $APPROVERS"

# 承認依頼送信
for approver in $APPROVERS; do
    echo "承認依頼送信: $approver"
    mail -s "変更承認依頼: $CHANGE_ID" "${approver}@example.com" << EOF
変更承認依頼

変更ID: $CHANGE_ID
影響度: $IMPACT
緊急度: $URGENCY

詳細は以下のファイルを確認してください:
$CHANGE_DIR/$(basename $CHANGE_FILE)

承認または却下の回答をお願いします。
承認: ./approve_change.sh $CHANGE_ID $approver approve
却下: ./approve_change.sh $CHANGE_ID $approver reject
EOF
    
    # 承認状況ファイル作成
    echo "$approver:pending:$(date)" >> $CHANGE_DIR/approval_status.txt
done

echo "承認依頼送信完了"
echo "承認状況確認: cat $CHANGE_DIR/approval_status.txt"
```

#### 3.2.2 承認処理スクリプト

```bash
#!/bin/bash
# approve_change.sh

CHANGE_ID="$1"
APPROVER="$2"
DECISION="$3"  # approve or reject
COMMENT="$4"

if [ -z "$CHANGE_ID" ] || [ -z "$APPROVER" ] || [ -z "$DECISION" ]; then
    echo "Usage: $0 <change_id> <approver> <approve|reject> [comment]"
    exit 1
fi

CHANGE_DIR="/var/log/changes/$CHANGE_ID"
APPROVAL_FILE="$CHANGE_DIR/approval_status.txt"

if [ ! -f "$APPROVAL_FILE" ]; then
    echo "Error: Change ID not found"
    exit 1
fi

echo "=== 変更承認処理 ==="
echo "変更ID: $CHANGE_ID"
echo "承認者: $APPROVER"
echo "決定: $DECISION"

# 承認状況更新
sed -i "s/${APPROVER}:pending:.*/${APPROVER}:${DECISION}:$(date)/" $APPROVAL_FILE

# コメント記録
if [ -n "$COMMENT" ]; then
    echo "$(date): $APPROVER - $DECISION - $COMMENT" >> $CHANGE_DIR/approval_comments.txt
fi

# 全承認者の状況確認
PENDING_COUNT=$(grep ":pending:" $APPROVAL_FILE | wc -l)
REJECTED_COUNT=$(grep ":reject:" $APPROVAL_FILE | wc -l)

if [ $REJECTED_COUNT -gt 0 ]; then
    echo "変更が却下されました"
    echo "状況: REJECTED" >> $CHANGE_DIR/change_status.txt
    
    # 却下通知
    REQUESTER=$(grep "要求者:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    mail -s "変更却下: $CHANGE_ID" "${REQUESTER}@example.com" << EOF
変更要求が却下されました。

変更ID: $CHANGE_ID
却下者: $APPROVER
理由: $COMMENT

詳細は $CHANGE_DIR を確認してください。
EOF

elif [ $PENDING_COUNT -eq 0 ]; then
    echo "全ての承認が完了しました"
    echo "状況: APPROVED" >> $CHANGE_DIR/change_status.txt
    
    # 承認完了通知
    REQUESTER=$(grep "要求者:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    mail -s "変更承認完了: $CHANGE_ID" "${REQUESTER}@example.com" << EOF
変更要求が承認されました。

変更ID: $CHANGE_ID
承認完了時刻: $(date)

実施スケジュールの調整を開始してください。
詳細は $CHANGE_DIR を確認してください。
EOF

    # 自動実施スケジューリング（緊急度による）
    URGENCY=$(grep "緊急度:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    case $URGENCY in
        "Emergency")
            echo "緊急変更として即座に実施スケジュールを設定します"
            ./schedule_change.sh $CHANGE_ID "$(date -d '+1 hour' +'%Y-%m-%d %H:%M:%S')"
            ;;
        "Urgent")
            echo "24時間以内の実施スケジュールを設定します"
            ./schedule_change.sh $CHANGE_ID "$(date -d '+12 hours' +'%Y-%m-%d %H:%M:%S')"
            ;;
    esac
    
else
    echo "承認待ち: $PENDING_COUNT件"
fi

echo "承認処理完了"
cat $APPROVAL_FILE
```

## 4. 変更実施

### 4.1 変更実施計画

#### 4.1.1 実施スケジューリング

```python
#!/usr/bin/env python3
# change_scheduler.py

import json
import sqlite3
from datetime import datetime, timedelta
import argparse

class ChangeScheduler:
    def __init__(self):
        self.db_file = "/var/log/changes/change_schedule.db"
        self.init_database()
        
    def init_database(self):
        """データベース初期化"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS change_schedule (
                change_id TEXT PRIMARY KEY,
                title TEXT NOT NULL,
                impact_level TEXT NOT NULL,
                urgency_level TEXT NOT NULL,
                scheduled_time TEXT NOT NULL,
                estimated_duration INTEGER NOT NULL,
                assigned_engineer TEXT,
                status TEXT DEFAULT 'scheduled',
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                updated_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS maintenance_windows (
                window_id INTEGER PRIMARY KEY AUTOINCREMENT,
                start_time TEXT NOT NULL,
                end_time TEXT NOT NULL,
                window_type TEXT NOT NULL,
                description TEXT,
                is_active INTEGER DEFAULT 1
            )
        ''')
        
        # デフォルトメンテナンス窓設定
        cursor.execute('''
            INSERT OR IGNORE INTO maintenance_windows 
            (window_id, start_time, end_time, window_type, description)
            VALUES 
            (1, '02:00', '04:00', 'daily', '日次メンテナンス窓'),
            (2, '02:00', '06:00', 'weekly_sunday', '週次メンテナンス窓（日曜日）'),
            (3, '02:00', '08:00', 'monthly_first_sunday', '月次メンテナンス窓（第1日曜日）')
        ''')
        
        conn.commit()
        conn.close()
    
    def schedule_change(self, change_id, change_file):
        """変更スケジューリング"""
        # 変更要求書読み込み
        change_info = self.parse_change_request(change_file)
        
        # 最適な実施時刻決定
        scheduled_time = self.find_optimal_schedule(change_info)
        
        # データベースに登録
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        cursor.execute('''
            INSERT OR REPLACE INTO change_schedule 
            (change_id, title, impact_level, urgency_level, 
             scheduled_time, estimated_duration, assigned_engineer)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        ''', (
            change_id,
            change_info['title'],
            change_info['impact'],
            change_info['urgency'],
            scheduled_time.isoformat(),
            change_info['duration'],
            change_info.get('engineer', 'TBD')
        ))
        
        conn.commit()
        conn.close()
        
        print(f"変更 {change_id} を {scheduled_time} にスケジュールしました")
        return scheduled_time
    
    def parse_change_request(self, change_file):
        """変更要求書解析"""
        with open(change_file, 'r', encoding='utf-8') as f:
            content = f.read()
        
        # 簡易パーサー（実際は詳細な解析が必要）
        info = {}
        
        for line in content.split('\n'):
            if line.startswith('変更タイトル:'):
                info['title'] = line.split(':', 1)[1].strip()
            elif line.startswith('影響度:'):
                info['impact'] = line.split(':', 1)[1].strip()
            elif line.startswith('緊急度:'):
                info['urgency'] = line.split(':', 1)[1].strip()
            elif line.startswith('停止時間:'):
                duration_str = line.split(':', 1)[1].strip()
                # 時間文字列を分に変換（簡易実装）
                if '時間' in duration_str:
                    hours = int(duration_str.split('時間')[0])
                    info['duration'] = hours * 60
                elif '分' in duration_str:
                    info['duration'] = int(duration_str.split('分')[0])
                else:
                    info['duration'] = 60  # デフォルト1時間
        
        return info
    
    def find_optimal_schedule(self, change_info):
        """最適スケジュール決定"""
        now = datetime.now()
        
        # 緊急度による基本スケジューリング
        if change_info['urgency'] == 'Emergency':
            # 緊急：1時間後
            return now + timedelta(hours=1)
        elif change_info['urgency'] == 'Urgent':
            # 急用：次のメンテナンス窓または12時間後
            next_window = self.find_next_maintenance_window('daily')
            urgent_time = now + timedelta(hours=12)
            return min(next_window, urgent_time)
        else:
            # 通常：適切なメンテナンス窓
            if change_info['impact'] in ['Critical', 'High']:
                return self.find_next_maintenance_window('weekly_sunday')
            else:
                return self.find_next_maintenance_window('daily')
    
    def find_next_maintenance_window(self, window_type):
        """次のメンテナンス窓検索"""
        now = datetime.now()
        
        if window_type == 'daily':
            # 次の午前2時
            next_window = now.replace(hour=2, minute=0, second=0, microsecond=0)
            if next_window <= now:
                next_window += timedelta(days=1)
        elif window_type == 'weekly_sunday':
            # 次の日曜日午前2時
            days_until_sunday = (6 - now.weekday()) % 7
            if days_until_sunday == 0 and now.hour >= 2:
                days_until_sunday = 7
            next_window = (now + timedelta(days=days_until_sunday)).replace(
                hour=2, minute=0, second=0, microsecond=0)
        else:
            # デフォルト：次の日次窓
            next_window = self.find_next_maintenance_window('daily')
        
        return next_window
    
    def get_schedule(self, date=None):
        """スケジュール取得"""
        conn = sqlite3.connect(self.db_file)
        cursor = conn.cursor()
        
        if date:
            cursor.execute('''
                SELECT * FROM change_schedule 
                WHERE date(scheduled_time) = ?
                ORDER BY scheduled_time
            ''', (date,))
        else:
            cursor.execute('''
                SELECT * FROM change_schedule 
                WHERE status = 'scheduled'
                ORDER BY scheduled_time
            ''')
        
        schedules = cursor.fetchall()
        conn.close()
        
        return schedules

if __name__ == "__main__":
    parser = argparse.ArgumentParser(description='Change Scheduler')
    parser.add_argument('command', choices=['schedule', 'list'])
    parser.add_argument('--change-id', help='Change ID')
    parser.add_argument('--change-file', help='Change request file')
    parser.add_argument('--date', help='Date (YYYY-MM-DD)')
    
    args = parser.parse_args()
    
    scheduler = ChangeScheduler()
    
    if args.command == 'schedule':
        if not args.change_id or not args.change_file:
            print("Error: --change-id and --change-file are required for schedule command")
            exit(1)
        scheduler.schedule_change(args.change_id, args.change_file)
    elif args.command == 'list':
        schedules = scheduler.get_schedule(args.date)
        print("スケジュール一覧:")
        for schedule in schedules:
            print(f"  {schedule[1]} - {schedule[4]} ({schedule[2]})")
```

### 4.2 変更実施手順

#### 4.2.1 実施前チェック

```bash
#!/bin/bash
# pre_change_check.sh

CHANGE_ID="$1"

if [ -z "$CHANGE_ID" ]; then
    echo "Usage: $0 <change_id>"
    exit 1
fi

CHANGE_DIR="/var/log/changes/$CHANGE_ID"
CHECK_RESULT="$CHANGE_DIR/pre_check_result.txt"

echo "=== 変更実施前チェック ===" | tee $CHECK_RESULT
echo "変更ID: $CHANGE_ID" | tee -a $CHECK_RESULT
echo "チェック実施時刻: $(date)" | tee -a $CHECK_RESULT

# 1. 承認状況確認
echo "## 承認状況確認" | tee -a $CHECK_RESULT
if [ -f "$CHANGE_DIR/approval_status.txt" ]; then
    REJECTED_COUNT=$(grep ":reject:" $CHANGE_DIR/approval_status.txt | wc -l)
    PENDING_COUNT=$(grep ":pending:" $CHANGE_DIR/approval_status.txt | wc -l)
    
    if [ $REJECTED_COUNT -gt 0 ]; then
        echo "FAIL: 変更が却下されています" | tee -a $CHECK_RESULT
        exit 1
    elif [ $PENDING_COUNT -gt 0 ]; then
        echo "FAIL: 承認が完了していません" | tee -a $CHECK_RESULT
        exit 1
    else
        echo "PASS: 承認完了" | tee -a $CHECK_RESULT
    fi
else
    echo "FAIL: 承認状況ファイルが見つかりません" | tee -a $CHECK_RESULT
    exit 1
fi

# 2. バックアップ状況確認
echo "## バックアップ状況確認" | tee -a $CHECK_RESULT
LATEST_BACKUP=$(find /backup -name "*$(date +%Y%m%d)*" -type f | head -1)
if [ -n "$LATEST_BACKUP" ]; then
    echo "PASS: 本日のバックアップ存在確認済み ($LATEST_BACKUP)" | tee -a $CHECK_RESULT
else
    echo "WARNING: 本日のバックアップが見つかりません" | tee -a $CHECK_RESULT
    read -p "バックアップなしで続行しますか？ (y/N): " confirm
    if [ "$confirm" != "y" ]; then
        echo "ABORT: バックアップ取得後に再実行してください" | tee -a $CHECK_RESULT
        exit 1
    fi
fi

# 3. システム状態確認
echo "## システム状態確認" | tee -a $CHECK_RESULT

# CPU使用率確認
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')
if (( $(echo "$CPU_USAGE > 80" | bc -l) )); then
    echo "WARNING: CPU使用率が高い状態です ($CPU_USAGE%)" | tee -a $CHECK_RESULT
else
    echo "PASS: CPU使用率正常 ($CPU_USAGE%)" | tee -a $CHECK_RESULT
fi

# メモリ使用率確認
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.1f", ($3/$2) * 100.0}')
if (( $(echo "$MEMORY_USAGE > 85" | bc -l) )); then
    echo "WARNING: メモリ使用率が高い状態です ($MEMORY_USAGE%)" | tee -a $CHECK_RESULT
else
    echo "PASS: メモリ使用率正常 ($MEMORY_USAGE%)" | tee -a $CHECK_RESULT
fi

# ディスク使用率確認
DISK_USAGE=$(df -h / | awk 'NR==2{print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 90 ]; then
    echo "WARNING: ディスク使用率が高い状態です ($DISK_USAGE%)" | tee -a $CHECK_RESULT
else
    echo "PASS: ディスク使用率正常 ($DISK_USAGE%)" | tee -a $CHECK_RESULT
fi

# 4. サービス状態確認
echo "## サービス状態確認" | tee -a $CHECK_RESULT
CRITICAL_SERVICES="nginx postgresql application"

for service in $CRITICAL_SERVICES; do
    if systemctl is-active --quiet $service; then
        echo "PASS: $service サービス正常稼働中" | tee -a $CHECK_RESULT
    else
        echo "FAIL: $service サービスが停止しています" | tee -a $CHECK_RESULT
        exit 1
    fi
done

# 5. ネットワーク確認
echo "## ネットワーク確認" | tee -a $CHECK_RESULT
if ping -c 3 8.8.8.8 > /dev/null 2>&1; then
    echo "PASS: 外部ネットワーク接続正常" | tee -a $CHECK_RESULT
else
    echo "FAIL: 外部ネットワーク接続に問題があります" | tee -a $CHECK_RESULT
    exit 1
fi

# 6. データベース接続確認
echo "## データベース接続確認" | tee -a $CHECK_RESULT
if psql -c "SELECT version();" > /dev/null 2>&1; then
    echo "PASS: データベース接続正常" | tee -a $CHECK_RESULT
else
    echo "FAIL: データベース接続に問題があります" | tee -a $CHECK_RESULT
    exit 1
fi

# 7. ロードバランサー確認（該当する場合）
echo "## ロードバランサー確認" | tee -a $CHECK_RESULT
if command -v curl &> /dev/null; then
    if curl -f http://localhost/health > /dev/null 2>&1; then
        echo "PASS: ヘルスチェックエンドポイント正常" | tee -a $CHECK_RESULT
    else
        echo "FAIL: ヘルスチェックエンドポイントに問題があります" | tee -a $CHECK_RESULT
        exit 1
    fi
fi

echo "## チェック結果サマリー" | tee -a $CHECK_RESULT
echo "全ての事前チェックが完了しました" | tee -a $CHECK_RESULT
echo "変更実施を開始できます" | tee -a $CHECK_RESULT

# チェック完了の記録
echo "pre_check_completed:$(date)" >> $CHANGE_DIR/change_status.txt

echo "事前チェック完了: $CHECK_RESULT"
```

#### 4.2.2 変更実施実行

```bash
#!/bin/bash
# execute_change.sh

CHANGE_ID="$1"
DRY_RUN="$2"  # --dry-run オプション

if [ -z "$CHANGE_ID" ]; then
    echo "Usage: $0 <change_id> [--dry-run]"
    exit 1
fi

CHANGE_DIR="/var/log/changes/$CHANGE_ID"
EXECUTION_LOG="$CHANGE_DIR/execution.log"

echo "=== 変更実施実行 ===" | tee $EXECUTION_LOG
echo "変更ID: $CHANGE_ID" | tee -a $EXECUTION_LOG
echo "実行開始時刻: $(date)" | tee -a $EXECUTION_LOG

if [ "$DRY_RUN" = "--dry-run" ]; then
    echo "DRY RUN モード: 実際の変更は行いません" | tee -a $EXECUTION_LOG
fi

# 1. 事前チェック完了確認
if ! grep -q "pre_check_completed" $CHANGE_DIR/change_status.txt; then
    echo "ERROR: 事前チェックが完了していません" | tee -a $EXECUTION_LOG
    exit 1
fi

# 2. 変更内容の読み込み
CHANGE_TYPE=$(grep "変更タイプ:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
CHANGE_TARGET=$(grep "変更対象:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)

echo "変更タイプ: $CHANGE_TYPE" | tee -a $EXECUTION_LOG
echo "変更対象: $CHANGE_TARGET" | tee -a $EXECUTION_LOG

# 3. ロールバックポイント作成
echo "## ロールバックポイント作成" | tee -a $EXECUTION_LOG
ROLLBACK_DIR="$CHANGE_DIR/rollback"
mkdir -p $ROLLBACK_DIR

# 設定ファイルのバックアップ
if [ "$DRY_RUN" != "--dry-run" ]; then
    cp -r /etc/nginx $ROLLBACK_DIR/ 2>/dev/null || echo "Nginx設定バックアップ失敗" | tee -a $EXECUTION_LOG
    cp -r /opt/application/config $ROLLBACK_DIR/ 2>/dev/null || echo "アプリケーション設定バックアップ失敗" | tee -a $EXECUTION_LOG
    
    # データベーススキーマバックアップ
    pg_dump --schema-only main_db > $ROLLBACK_DIR/schema_backup.sql 2>/dev/null || echo "スキーマバックアップ失敗" | tee -a $EXECUTION_LOG
fi

echo "ロールバックポイント作成完了" | tee -a $EXECUTION_LOG

# 4. 変更実施
echo "## 変更実施" | tee -a $EXECUTION_LOG

# 変更タイプ別の実施処理
case $CHANGE_TYPE in
    "アプリケーション")
        execute_application_change "$DRY_RUN" | tee -a $EXECUTION_LOG
        ;;
    "インフラ")
        execute_infrastructure_change "$DRY_RUN" | tee -a $EXECUTION_LOG
        ;;
    "セキュリティ")
        execute_security_change "$DRY_RUN" | tee -a $EXECUTION_LOG
        ;;
    "データベース")
        execute_database_change "$DRY_RUN" | tee -a $EXECUTION_LOG
        ;;
    *)
        echo "ERROR: 未対応の変更タイプです: $CHANGE_TYPE" | tee -a $EXECUTION_LOG
        exit 1
        ;;
esac

CHANGE_EXIT_CODE=$?

# 5. 実施後確認
echo "## 実施後確認" | tee -a $EXECUTION_LOG

if [ $CHANGE_EXIT_CODE -eq 0 ]; then
    echo "変更実施成功" | tee -a $EXECUTION_LOG
    
    if [ "$DRY_RUN" != "--dry-run" ]; then
        # ヘルスチェック
        sleep 30
        if ./post_change_check.sh $CHANGE_ID; then
            echo "実施後確認成功" | tee -a $EXECUTION_LOG
            echo "change_completed:$(date)" >> $CHANGE_DIR/change_status.txt
        else
            echo "実施後確認失敗 - ロールバック実行" | tee -a $EXECUTION_LOG
            ./rollback_change.sh $CHANGE_ID
            exit 1
        fi
    else
        echo "DRY RUN 完了 - 実際の変更は実施されていません" | tee -a $EXECUTION_LOG
    fi
else
    echo "変更実施失敗 - ロールバック実行" | tee -a $EXECUTION_LOG
    if [ "$DRY_RUN" != "--dry-run" ]; then
        ./rollback_change.sh $CHANGE_ID
    fi
    exit 1
fi

echo "実行終了時刻: $(date)" | tee -a $EXECUTION_LOG
echo "変更実施ログ: $EXECUTION_LOG"

# 変更タイプ別実施関数
execute_application_change() {
    local dry_run=$1
    echo "アプリケーション変更を実施します"
    
    if [ "$dry_run" = "--dry-run" ]; then
        echo "DRY RUN: アプリケーションサービス停止"
        echo "DRY RUN: アプリケーションファイル更新"
        echo "DRY RUN: アプリケーションサービス開始"
        return 0
    fi
    
    # 実際のアプリケーション変更
    systemctl stop application
    # 実際の更新処理をここに実装
    systemctl start application
    return $?
}

execute_infrastructure_change() {
    local dry_run=$1
    echo "インフラ変更を実施します"
    
    if [ "$dry_run" = "--dry-run" ]; then
        echo "DRY RUN: インフラ設定変更"
        return 0
    fi
    
    # 実際のインフラ変更
    # 設定変更処理をここに実装
    return $?
}

execute_security_change() {
    local dry_run=$1
    echo "セキュリティ変更を実施します"
    
    if [ "$dry_run" = "--dry-run" ]; then
        echo "DRY RUN: セキュリティ設定変更"
        return 0
    fi
    
    # 実際のセキュリティ変更
    # セキュリティ設定変更処理をここに実装
    return $?
}

execute_database_change() {
    local dry_run=$1
    echo "データベース変更を実施します"
    
    if [ "$dry_run" = "--dry-run" ]; then
        echo "DRY RUN: データベーススキーマ変更"
        return 0
    fi
    
    # 実際のデータベース変更
    # スキーマ変更処理をここに実装
    return $?
}
```

### 4.3 実施後確認・ロールバック

#### 4.3.1 実施後確認

```bash
#!/bin/bash
# post_change_check.sh

CHANGE_ID="$1"

if [ -z "$CHANGE_ID" ]; then
    echo "Usage: $0 <change_id>"
    exit 1
fi

CHANGE_DIR="/var/log/changes/$CHANGE_ID"
POST_CHECK_RESULT="$CHANGE_DIR/post_check_result.txt"

echo "=== 変更実施後確認 ===" | tee $POST_CHECK_RESULT
echo "変更ID: $CHANGE_ID" | tee -a $POST_CHECK_RESULT
echo "確認実施時刻: $(date)" | tee -a $POST_CHECK_RESULT

CHECK_PASSED=true

# 1. サービス状態確認
echo "## サービス状態確認" | tee -a $POST_CHECK_RESULT
CRITICAL_SERVICES="nginx postgresql application"

for service in $CRITICAL_SERVICES; do
    if systemctl is-active --quiet $service; then
        echo "PASS: $service サービス正常稼働中" | tee -a $POST_CHECK_RESULT
    else
        echo "FAIL: $service サービスが停止しています" | tee -a $POST_CHECK_RESULT
        CHECK_PASSED=false
    fi
done

# 2. ヘルスチェック
echo "## ヘルスチェック" | tee -a $POST_CHECK_RESULT
if curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "PASS: アプリケーションヘルスチェック成功" | tee -a $POST_CHECK_RESULT
else
    echo "FAIL: アプリケーションヘルスチェック失敗" | tee -a $POST_CHECK_RESULT
    CHECK_PASSED=false
fi

if curl -f http://localhost/health > /dev/null 2>&1; then
    echo "PASS: Webサーバーヘルスチェック成功" | tee -a $POST_CHECK_RESULT
else
    echo "FAIL: Webサーバーヘルスチェック失敗" | tee -a $POST_CHECK_RESULT
    CHECK_PASSED=false
fi

# 3. データベース接続確認
echo "## データベース接続確認" | tee -a $POST_CHECK_RESULT
if psql -c "SELECT 1;" > /dev/null 2>&1; then
    echo "PASS: データベース接続成功" | tee -a $POST_CHECK_RESULT
else
    echo "FAIL: データベース接続失敗" | tee -a $POST_CHECK_RESULT
    CHECK_PASSED=false
fi

# 4. パフォーマンステスト
echo "## パフォーマンステスト" | tee -a $POST_CHECK_RESULT
RESPONSE_TIME=$(curl -o /dev/null -s -w "%{time_total}" http://localhost:8080/health 2>/dev/null)
if [ $? -eq 0 ]; then
    if (( $(echo "$RESPONSE_TIME < 3.0" | bc -l) )); then
        echo "PASS: レスポンス時間正常 ($RESPONSE_TIME秒)" | tee -a $POST_CHECK_RESULT
    else
        echo "WARNING: レスポンス時間が遅い ($RESPONSE_TIME秒)" | tee -a $POST_CHECK_RESULT
    fi
else
    echo "FAIL: レスポンス時間測定失敗" | tee -a $POST_CHECK_RESULT
    CHECK_PASSED=false
fi

# 5. ログエラー確認
echo "## ログエラー確認" | tee -a $POST_CHECK_RESULT
ERROR_COUNT=$(grep -i error /var/log/application/*.log | grep "$(date +%Y-%m-%d)" | wc -l)
if [ $ERROR_COUNT -eq 0 ]; then
    echo "PASS: 新しいエラーログなし" | tee -a $POST_CHECK_RESULT
else
    echo "WARNING: $ERROR_COUNT 件のエラーログあり" | tee -a $POST_CHECK_RESULT
    tail -10 /var/log/application/error.log | tee -a $POST_CHECK_RESULT
fi

# 6. 機能テスト（変更内容に応じて）
echo "## 機能テスト" | tee -a $POST_CHECK_RESULT
# 変更内容に応じた機能テストを実装
# 例：API エンドポイントテスト
if curl -f "http://localhost:8080/api/users" > /dev/null 2>&1; then
    echo "PASS: ユーザーAPI正常" | tee -a $POST_CHECK_RESULT
else
    echo "FAIL: ユーザーAPI異常" | tee -a $POST_CHECK_RESULT
    CHECK_PASSED=false
fi

# 7. 総合判定
echo "## 総合判定" | tee -a $POST_CHECK_RESULT
if [ "$CHECK_PASSED" = true ]; then
    echo "SUCCESS: 全ての確認項目が正常です" | tee -a $POST_CHECK_RESULT
    echo "post_check_passed:$(date)" >> $CHANGE_DIR/change_status.txt
    
    # 成功通知
    REQUESTER=$(grep "要求者:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    mail -s "変更実施成功: $CHANGE_ID" "${REQUESTER}@example.com" << EOF
変更が正常に完了しました。

変更ID: $CHANGE_ID
完了時刻: $(date)

詳細は $POST_CHECK_RESULT を確認してください。
EOF
    
    exit 0
else
    echo "FAILURE: 確認項目に失敗があります" | tee -a $POST_CHECK_RESULT
    echo "post_check_failed:$(date)" >> $CHANGE_DIR/change_status.txt
    
    # 失敗通知
    REQUESTER=$(grep "要求者:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    mail -s "変更実施後確認失敗: $CHANGE_ID" "${REQUESTER}@example.com" << EOF
変更実施後の確認で問題が検出されました。

変更ID: $CHANGE_ID
失敗時刻: $(date)

ロールバックが必要な可能性があります。
詳細は $POST_CHECK_RESULT を確認してください。
EOF
    
    exit 1
fi
```

#### 4.3.2 ロールバック実行

```bash
#!/bin/bash
# rollback_change.sh

CHANGE_ID="$1"
FORCE_ROLLBACK="$2"  # --force オプション

if [ -z "$CHANGE_ID" ]; then
    echo "Usage: $0 <change_id> [--force]"
    exit 1
fi

CHANGE_DIR="/var/log/changes/$CHANGE_ID"
ROLLBACK_LOG="$CHANGE_DIR/rollback.log"
ROLLBACK_DIR="$CHANGE_DIR/rollback"

echo "=== 変更ロールバック実行 ===" | tee $ROLLBACK_LOG
echo "変更ID: $CHANGE_ID" | tee -a $ROLLBACK_LOG
echo "ロールバック開始時刻: $(date)" | tee -a $ROLLBACK_LOG

# 1. ロールバック可能性確認
if [ ! -d "$ROLLBACK_DIR" ]; then
    echo "ERROR: ロールバックデータが見つかりません" | tee -a $ROLLBACK_LOG
    exit 1
fi

# 2. 確認プロンプト（強制モードでない場合）
if [ "$FORCE_ROLLBACK" != "--force" ]; then
    echo "WARNING: ロールバックを実行します。これにより変更が取り消されます。"
    read -p "続行しますか？ (yes/no): " confirm
    if [ "$confirm" != "yes" ]; then
        echo "ロールバック中止" | tee -a $ROLLBACK_LOG
        exit 1
    fi
fi

# 3. ロールバック実行
echo "## ロールバック実行" | tee -a $ROLLBACK_LOG

# アプリケーション停止
echo "アプリケーション停止" | tee -a $ROLLBACK_LOG
systemctl stop application
systemctl stop nginx

# 設定ファイル復元
echo "設定ファイル復元" | tee -a $ROLLBACK_LOG
if [ -d "$ROLLBACK_DIR/nginx" ]; then
    rm -rf /etc/nginx.backup
    mv /etc/nginx /etc/nginx.backup
    cp -r $ROLLBACK_DIR/nginx /etc/
    echo "Nginx設定復元完了" | tee -a $ROLLBACK_LOG
fi

if [ -d "$ROLLBACK_DIR/config" ]; then
    rm -rf /opt/application/config.backup
    mv /opt/application/config /opt/application/config.backup
    cp -r $ROLLBACK_DIR/config /opt/application/
    echo "アプリケーション設定復元完了" | tee -a $ROLLBACK_LOG
fi

# データベーススキーマ復元
if [ -f "$ROLLBACK_DIR/schema_backup.sql" ]; then
    echo "データベーススキーマ復元" | tee -a $ROLLBACK_LOG
    psql main_db < $ROLLBACK_DIR/schema_backup.sql 2>&1 | tee -a $ROLLBACK_LOG
fi

# サービス再起動
echo "サービス再起動" | tee -a $ROLLBACK_LOG
systemctl start nginx
systemctl start application

# 4. ロールバック後確認
echo "## ロールバック後確認" | tee -a $ROLLBACK_LOG
sleep 30

# サービス状態確認
SERVICES_OK=true
for service in nginx postgresql application; do
    if systemctl is-active --quiet $service; then
        echo "PASS: $service サービス正常稼働中" | tee -a $ROLLBACK_LOG
    else
        echo "FAIL: $service サービスが停止しています" | tee -a $ROLLBACK_LOG
        SERVICES_OK=false
    fi
done

# ヘルスチェック
if curl -f http://localhost:8080/health > /dev/null 2>&1; then
    echo "PASS: アプリケーションヘルスチェック成功" | tee -a $ROLLBACK_LOG
else
    echo "FAIL: アプリケーションヘルスチェック失敗" | tee -a $ROLLBACK_LOG
    SERVICES_OK=false
fi

# 5. 結果判定
echo "## ロールバック結果" | tee -a $ROLLBACK_LOG
if [ "$SERVICES_OK" = true ]; then
    echo "SUCCESS: ロールバック成功" | tee -a $ROLLBACK_LOG
    echo "rollback_completed:$(date)" >> $CHANGE_DIR/change_status.txt
    
    # ロールバック成功通知
    REQUESTER=$(grep "要求者:" $CHANGE_DIR/*.txt | cut -d: -f2 | xargs)
    mail -s "ロールバック完了: $CHANGE_ID" "${REQUESTER}@example.com" << EOF
変更のロールバックが完了しました。

変更ID: $CHANGE_ID
ロールバック完了時刻: $(date)

システムは変更前の状態に復元されました。
詳細は $ROLLBACK_LOG を確認してください。
EOF
    
    exit 0
else
    echo "FAILURE: ロールバック後にサービス異常があります" | tee -a $ROLLBACK_LOG
    echo "rollback_failed:$(date)" >> $CHANGE_DIR/change_status.txt
    
    # ロールバック失敗通知
    mail -s "ロールバック失敗: $CHANGE_ID" admin@example.com << EOF
ロールバックに失敗しました。緊急対応が必要です。

変更ID: $CHANGE_ID
失敗時刻: $(date)

手動での復旧が必要な可能性があります。
詳細は $ROLLBACK_LOG を確認してください。
EOF
    
    exit 1
fi

echo "ロールバック終了時刻: $(date)" | tee -a $ROLLBACK_LOG
```

## 5. 変更管理レポート

### 5.1 月次レポート

#### 5.1.1 変更統計レポート

```python
#!/usr/bin/env python3
# change_management_report.py

import sqlite3
import json
from datetime import datetime, timedelta
from collections import Counter
import matplotlib.pyplot as plt

class ChangeManagementReporter:
    def __init__(self):
        self.db_file = "/var/log/changes/change_schedule.db"
        self.changes_dir = "/var/log/changes"
        
    def generate_monthly_report(self, year, month):
        """月次変更管理レポート生成"""
        report_data = {
            'period': f"{year}-{month:02d}",
            'generated_at': datetime.now().isoformat(),
            'statistics': self.get_change_statistics(year, month),
            'success_rate': self.calculate_success_rate(year, month),
            'impact_analysis': self.analyze_change_impact(year, month),
            'recommendations': self.generate_recommendations(year, month)
        }
        
        # レポートファイル生成
        report_file = f"/var/log/changes/monthly_report_{year}{month:02d}.json"
        with open(report_file, 'w', encoding='utf-8') as f:
            json.dump(report_data, f, indent=2, ensure_ascii=False)
        
        # テキストレポート生成
        text_report = f"/var/log/changes/monthly_report_{year}{month:02d}.txt"
        self.generate_text_report(report_data, text_report)
        
        return report_data
    
    def get_change_statistics(self, year, month):
        """変更統計取得"""
        # 実装簡略化：実際はデータベースから取得
        return {
            'total_changes': 24,
            'successful_changes': 22,
            'failed_changes': 1,
            'rolled_back_changes': 1,
            'by_impact': {
                'critical': 2,
                'high': 8,
                'medium': 10,
                'low': 4
            },
            'by_type': {
                'application': 12,
                'infrastructure': 6,
                'security': 4,
                'database': 2
            },
            'by_urgency': {
                'emergency': 1,
                'urgent': 3,
                'standard': 18,
                'scheduled': 2
            }
        }
    
    def calculate_success_rate(self, year, month):
        """成功率計算"""
        stats = self.get_change_statistics(year, month)
        total = stats['total_changes']
        successful = stats['successful_changes']
        
        if total > 0:
            success_rate = (successful / total) * 100
        else:
            success_rate = 0
        
        return {
            'overall_success_rate': success_rate,
            'target_success_rate': 95.0,
            'meets_target': success_rate >= 95.0
        }
    
    def analyze_change_impact(self, year, month):
        """変更影響分析"""
        return {
            'average_downtime_minutes': 15,
            'total_downtime_minutes': 180,
            'changes_with_downtime': 8,
            'major_incidents_caused': 0,
            'customer_impact_incidents': 1
        }
    
    def generate_recommendations(self, year, month):
        """推奨事項生成"""
        success_rate = self.calculate_success_rate(year, month)
        stats = self.get_change_statistics(year, month)
        
        recommendations = []
        
        if not success_rate['meets_target']:
            recommendations.append("成功率が目標を下回っています。変更プロセスの見直しを検討してください。")
        
        if stats['failed_changes'] > 2:
            recommendations.append("失敗した変更が多数あります。事前テストの強化を検討してください。")
        
        if stats['by_urgency']['emergency'] > 0:
            recommendations.append("緊急変更が発生しています。予防的メンテナンスの強化を検討してください。")
        
        return recommendations
    
    def generate_text_report(self, report_data, output_file):
        """テキストレポート生成"""
        with open(output_file, 'w', encoding='utf-8') as f:
            f.write("=== 月次変更管理レポート ===\n")
            f.write(f"対象期間: {report_data['period']}\n")
            f.write(f"レポート生成日時: {report_data['generated_at']}\n\n")
            
            # 統計情報
            stats = report_data['statistics']
            f.write("## 変更統計\n")
            f.write(f"総変更数: {stats['total_changes']}\n")
            f.write(f"成功: {stats['successful_changes']}\n")
            f.write(f"失敗: {stats['failed_changes']}\n")
            f.write(f"ロールバック: {stats['rolled_back_changes']}\n\n")
            
            # 成功率
            success = report_data['success_rate']
            f.write("## 成功率\n")
            f.write(f"全体成功率: {success['overall_success_rate']:.1f}%\n")
            f.write(f"目標成功率: {success['target_success_rate']:.1f}%\n")
            f.write(f"目標達成: {'はい' if success['meets_target'] else 'いいえ'}\n\n")
            
            # 影響分析
            impact = report_data['impact_analysis']
            f.write("## 影響分析\n")
            f.write(f"平均停止時間: {impact['average_downtime_minutes']}分\n")
            f.write(f"総停止時間: {impact['total_downtime_minutes']}分\n")
            f.write(f"停止を伴う変更数: {impact['changes_with_downtime']}\n")
            f.write(f"重大インシデント: {impact['major_incidents_caused']}件\n\n")
            
            # 推奨事項
            f.write("## 推奨事項\n")
            for i, rec in enumerate(report_data['recommendations'], 1):
                f.write(f"{i}. {rec}\n")

if __name__ == "__main__":
    reporter = ChangeManagementReporter()
    
    # 今月のレポート生成
    now = datetime.now()
    report = reporter.generate_monthly_report(now.year, now.month)
    
    print(f"月次変更管理レポートを生成しました: {now.year}-{now.month:02d}")
```

---

**注意**: この変更管理手順書は、システム変更の安全な実施のための重要な手順を定義しています。組織の要件に応じて承認フローや実施手順をカスタマイズしてください。