# 性能管理手順書

**タイトル**: 性能管理手順書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: performance-design.md, monitoring-configuration.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの性能目標達成と維持のための管理手順を定義し、継続的な性能改善を実現することを目的とする。

### 1.2 適用範囲
本手順は、以下の性能管理活動に適用される：
- 性能監視・分析
- 性能劣化の検知・対応
- 性能チューニング
- キャパシティプランニング

### 1.3 性能管理方針
- **予防的管理**: 性能問題の事前検知と対策
- **継続監視**: リアルタイム性能監視
- **データ駆動**: メトリクスに基づく意思決定
- **段階的改善**: 計画的かつ段階的な性能向上

## 2. 性能目標・SLA

### 2.1 性能目標値

#### 2.1.1 レスポンス時間目標

| サービス | 目標値 | 警告閾値 | 限界値 | 測定条件 |
|---|---|---|---|---|
| **Webページ表示** | 2秒以内 | 3秒 | 5秒 | 95%ile |
| **API レスポンス** | 500ms以内 | 1秒 | 2秒 | 95%ile |
| **データベースクエリ** | 100ms以内 | 200ms | 500ms | 平均 |
| **バッチ処理** | 1時間以内 | 1.5時間 | 2時間 | 最大 |

#### 2.1.2 スループット目標

| 項目 | 目標値 | 警告閾値 | 限界値 | 測定期間 |
|---|---|---|---|---|
| **同時接続ユーザー** | 500ユーザー | 800ユーザー | 1000ユーザー | ピーク時 |
| **API リクエスト/秒** | 1000 RPS | 1500 RPS | 2000 RPS | 継続 |
| **データ処理量** | 10万件/時 | 8万件/時 | 5万件/時 | バッチ処理 |

#### 2.1.3 リソース使用率目標

| リソース | 通常運用 | 警告閾値 | 限界閾値 | 対応要否 |
|---|---|---|---|---|
| **CPU使用率** | 60%以下 | 80% | 90% | 要 |
| **メモリ使用率** | 70%以下 | 85% | 95% | 要 |
| **ディスク使用率** | 75%以下 | 85% | 95% | 要 |
| **ネットワーク使用率** | 50%以下 | 70% | 85% | 要 |

### 2.2 可用性目標

| システム | 目標稼働率 | 最大停止時間/月 | 計画停止時間 |
|---|---|---|---|
| **本番システム** | 99.9% | 43分 | 月次メンテナンス4時間 |
| **データベース** | 99.95% | 22分 | 四半期メンテナンス2時間 |
| **監視システム** | 99.5% | 3時間36分 | なし |

## 3. 性能監視

### 3.1 日常監視

#### 3.1.1 リアルタイム監視項目

**監視スクリプト（毎分実行）**
```bash
#!/bin/bash
# realtime_performance_monitoring.sh

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
METRICS_FILE="/var/log/performance/realtime_metrics.log"

# CPU使用率
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')

# メモリ使用率
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.2f", ($3/$2) * 100.0}')

# ディスク使用率
DISK_USAGE=$(df -h / | awk 'NR==2{print $5}' | sed 's/%//')

# ロードアベレージ
LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)

# アクティブ接続数
ACTIVE_CONNECTIONS=$(netstat -an | grep :80 | grep ESTABLISHED | wc -l)

# データベース接続数
DB_CONNECTIONS=$(psql -t -c "SELECT count(*) FROM pg_stat_activity;" 2>/dev/null || echo "N/A")

# アプリケーション応答時間
APP_RESPONSE_TIME=$(curl -o /dev/null -s -w "%{time_total}" http://localhost:8080/health 2>/dev/null || echo "N/A")

# ログ出力
echo "$TIMESTAMP,CPU:$CPU_USAGE%,Memory:$MEMORY_USAGE%,Disk:$DISK_USAGE%,Load:$LOAD_AVG,Connections:$ACTIVE_CONNECTIONS,DB_Conn:$DB_CONNECTIONS,Response:${APP_RESPONSE_TIME}s" >> $METRICS_FILE

# 閾値チェック
if (( $(echo "$CPU_USAGE > 80" | bc -l) )); then
    echo "WARNING: High CPU usage: $CPU_USAGE%" | mail -s "Performance Alert: High CPU" admin@example.com
fi

if (( $(echo "$MEMORY_USAGE > 85" | bc -l) )); then
    echo "WARNING: High memory usage: $MEMORY_USAGE%" | mail -s "Performance Alert: High Memory" admin@example.com
fi

if (( $(echo "$APP_RESPONSE_TIME > 3" | bc -l) )); then
    echo "WARNING: Slow application response: ${APP_RESPONSE_TIME}s" | mail -s "Performance Alert: Slow Response" admin@example.com
fi
```

#### 3.1.2 アプリケーション性能監視

```python
#!/usr/bin/env python3
# application_performance_monitor.py

import time
import requests
import statistics
import json
from datetime import datetime
import psycopg2

class ApplicationPerformanceMonitor:
    def __init__(self):
        self.base_url = "http://localhost:8080"
        self.endpoints = [
            "/health",
            "/api/users",
            "/api/orders",
            "/api/products"
        ]
        self.db_config = {
            'host': 'localhost',
            'database': 'main_db',
            'user': 'monitor_user',
            'password': 'monitor_pass'
        }
    
    def measure_endpoint_performance(self, endpoint, iterations=5):
        """エンドポイント性能測定"""
        response_times = []
        errors = 0
        
        for i in range(iterations):
            try:
                start_time = time.time()
                response = requests.get(f"{self.base_url}{endpoint}", timeout=10)
                end_time = time.time()
                
                response_time = (end_time - start_time) * 1000  # ms
                response_times.append(response_time)
                
                if response.status_code != 200:
                    errors += 1
                    
            except Exception as e:
                errors += 1
                print(f"Error measuring {endpoint}: {e}")
        
        if response_times:
            return {
                'endpoint': endpoint,
                'avg_response_time': statistics.mean(response_times),
                'median_response_time': statistics.median(response_times),
                'min_response_time': min(response_times),
                'max_response_time': max(response_times),
                'error_count': errors,
                'success_rate': ((iterations - errors) / iterations) * 100
            }
        else:
            return {
                'endpoint': endpoint,
                'error': 'All requests failed'
            }
    
    def measure_database_performance(self):
        """データベース性能測定"""
        try:
            conn = psycopg2.connect(**self.db_config)
            cursor = conn.cursor()
            
            # 接続数確認
            cursor.execute("SELECT count(*) FROM pg_stat_activity;")
            connection_count = cursor.fetchone()[0]
            
            # スロークエリ確認
            cursor.execute("""
                SELECT count(*) FROM pg_stat_activity 
                WHERE state = 'active' AND query_start < now() - interval '5 minutes';
            """)
            slow_queries = cursor.fetchone()[0]
            
            # データベースサイズ
            cursor.execute("SELECT pg_size_pretty(pg_database_size(current_database()));")
            db_size = cursor.fetchone()[0]
            
            # 簡単なクエリの実行時間測定
            start_time = time.time()
            cursor.execute("SELECT version();")
            cursor.fetchone()
            end_time = time.time()
            
            query_time = (end_time - start_time) * 1000  # ms
            
            cursor.close()
            conn.close()
            
            return {
                'connection_count': connection_count,
                'slow_queries': slow_queries,
                'database_size': db_size,
                'simple_query_time_ms': query_time
            }
            
        except Exception as e:
            return {'error': str(e)}
    
    def generate_performance_report(self):
        """性能レポート生成"""
        timestamp = datetime.now()
        report = {
            'timestamp': timestamp.isoformat(),
            'application_performance': {},
            'database_performance': {}
        }
        
        # アプリケーション性能測定
        for endpoint in self.endpoints:
            report['application_performance'][endpoint] = self.measure_endpoint_performance(endpoint)
        
        # データベース性能測定
        report['database_performance'] = self.measure_database_performance()
        
        # レポート保存
        report_file = f"/var/log/performance/app_performance_{timestamp.strftime('%Y%m%d_%H%M%S')}.json"
        with open(report_file, 'w') as f:
            json.dump(report, f, indent=2)
        
        # アラートチェック
        self.check_performance_alerts(report)
        
        return report
    
    def check_performance_alerts(self, report):
        """性能アラートチェック"""
        alerts = []
        
        # アプリケーション性能チェック
        for endpoint, metrics in report['application_performance'].items():
            if 'avg_response_time' in metrics:
                if metrics['avg_response_time'] > 1000:  # 1秒以上
                    alerts.append(f"Slow response time for {endpoint}: {metrics['avg_response_time']:.2f}ms")
                
                if metrics['success_rate'] < 95:  # 成功率95%未満
                    alerts.append(f"Low success rate for {endpoint}: {metrics['success_rate']:.1f}%")
        
        # データベース性能チェック
        db_metrics = report['database_performance']
        if 'connection_count' in db_metrics and db_metrics['connection_count'] > 80:
            alerts.append(f"High database connection count: {db_metrics['connection_count']}")
        
        if 'slow_queries' in db_metrics and db_metrics['slow_queries'] > 0:
            alerts.append(f"Slow queries detected: {db_metrics['slow_queries']}")
        
        # アラート送信
        if alerts:
            alert_message = "\n".join(alerts)
            self.send_alert(alert_message)
    
    def send_alert(self, message):
        """アラート送信"""
        import smtplib
        from email.mime.text import MimeText
        
        # メール送信（簡易実装）
        print(f"PERFORMANCE ALERT: {message}")
        # 実際の実装では適切なメール送信処理を行う

if __name__ == "__main__":
    monitor = ApplicationPerformanceMonitor()
    report = monitor.generate_performance_report()
    print("Performance monitoring completed")
```

### 3.2 週次性能分析

#### 3.2.1 トレンド分析

```bash
#!/bin/bash
# weekly_performance_analysis.sh

ANALYSIS_DATE=$(date +%Y%m%d)
REPORT_DIR="/var/log/performance/weekly"
mkdir -p $REPORT_DIR

echo "=== 週次性能分析 $(date) ===" | tee $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# 1. CPU使用率トレンド
echo "## CPU使用率トレンド" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt
grep "CPU:" /var/log/performance/realtime_metrics.log | tail -10080 | \
    cut -d, -f2 | cut -d: -f2 | sed 's/%//' | \
    python3 -c "
import sys
import statistics
data = [float(line.strip()) for line in sys.stdin if line.strip()]
if data:
    print(f'Average: {statistics.mean(data):.2f}%')
    print(f'Max: {max(data):.2f}%')
    print(f'Min: {min(data):.2f}%')
    print(f'95th percentile: {sorted(data)[int(len(data)*0.95)]:.2f}%')
" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# 2. メモリ使用率トレンド
echo "## メモリ使用率トレンド" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt
grep "Memory:" /var/log/performance/realtime_metrics.log | tail -10080 | \
    cut -d, -f3 | cut -d: -f2 | sed 's/%//' | \
    python3 -c "
import sys
import statistics
data = [float(line.strip()) for line in sys.stdin if line.strip()]
if data:
    print(f'Average: {statistics.mean(data):.2f}%')
    print(f'Max: {max(data):.2f}%')
    print(f'Min: {min(data):.2f}%')
    print(f'95th percentile: {sorted(data)[int(len(data)*0.95)]:.2f}%')
" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# 3. 応答時間分析
echo "## 応答時間分析" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt
find /var/log/performance -name "app_performance_*.json" -mtime -7 | \
    xargs python3 -c "
import json
import sys
import statistics

response_times = []
for file_path in sys.argv[1:]:
    try:
        with open(file_path, 'r') as f:
            data = json.load(f)
            for endpoint, metrics in data.get('application_performance', {}).items():
                if 'avg_response_time' in metrics:
                    response_times.append(metrics['avg_response_time'])
    except:
        continue

if response_times:
    print(f'Average response time: {statistics.mean(response_times):.2f}ms')
    print(f'Max response time: {max(response_times):.2f}ms')
    print(f'95th percentile: {sorted(response_times)[int(len(response_times)*0.95)]:.2f}ms')
else:
    print('No response time data available')
" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# 4. エラー率分析
echo "## エラー率分析" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt
awk '$9>=400 {print $9}' /var/log/nginx/access.log | \
    python3 -c "
import sys
from collections import Counter

status_codes = [line.strip() for line in sys.stdin]
error_count = len(status_codes)
total_requests = sum(1 for line in open('/var/log/nginx/access.log'))

if total_requests > 0:
    error_rate = (error_count / total_requests) * 100
    print(f'Error rate: {error_rate:.2f}%')
    print(f'Total errors: {error_count}')
    print(f'Total requests: {total_requests}')
    
    if status_codes:
        error_breakdown = Counter(status_codes)
        print('Error breakdown:')
        for status, count in error_breakdown.most_common(5):
            print(f'  {status}: {count}')
else:
    print('No request data available')
" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# 5. 推奨事項生成
echo "## 推奨事項" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt
python3 -c "
import re

with open('$REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt', 'r') as f:
    content = f.read()

recommendations = []

# CPU使用率チェック
cpu_avg = re.search(r'CPU.*Average: ([0-9.]+)%', content)
if cpu_avg and float(cpu_avg.group(1)) > 70:
    recommendations.append('CPU使用率が高くなっています。スケールアウトを検討してください。')

# メモリ使用率チェック
mem_avg = re.search(r'Memory.*Average: ([0-9.]+)%', content)
if mem_avg and float(mem_avg.group(1)) > 80:
    recommendations.append('メモリ使用率が高くなっています。メモリ増設を検討してください。')

# 応答時間チェック
resp_avg = re.search(r'Average response time: ([0-9.]+)ms', content)
if resp_avg and float(resp_avg.group(1)) > 500:
    recommendations.append('応答時間が遅くなっています。アプリケーションの最適化を検討してください。')

# エラー率チェック
error_rate = re.search(r'Error rate: ([0-9.]+)%', content)
if error_rate and float(error_rate.group(1)) > 1:
    recommendations.append('エラー率が高くなっています。エラーの原因調査を行ってください。')

if recommendations:
    for rec in recommendations:
        print(f'- {rec}')
else:
    print('- 現在、特に問題は検出されていません。')
" | tee -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

# レポート送信
mail -s "Weekly Performance Analysis Report" -a $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt \
    admin@example.com < $REPORT_DIR/weekly_analysis_$ANALYSIS_DATE.txt

echo "$(date): Weekly performance analysis completed" >> /var/log/performance/performance_ops.log
```

## 4. 性能チューニング

### 4.1 自動チューニング

#### 4.1.1 動的設定調整

```python
#!/usr/bin/env python3
# auto_performance_tuning.py

import psutil
import subprocess
import time
import logging
from datetime import datetime

class AutoPerformanceTuner:
    def __init__(self):
        self.logger = self.setup_logging()
        self.tuning_actions = []
        
    def setup_logging(self):
        """ログ設定"""
        logger = logging.getLogger('AutoTuner')
        logger.setLevel(logging.INFO)
        handler = logging.FileHandler('/var/log/performance/auto_tuning.log')
        formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
        handler.setFormatter(formatter)
        logger.addHandler(handler)
        return logger
    
    def get_system_metrics(self):
        """システムメトリクス取得"""
        return {
            'cpu_percent': psutil.cpu_percent(interval=1),
            'memory_percent': psutil.virtual_memory().percent,
            'disk_usage': psutil.disk_usage('/').percent,
            'load_average': psutil.getloadavg()[0],
            'network_io': psutil.net_io_counters(),
            'disk_io': psutil.disk_io_counters()
        }
    
    def tune_nginx_workers(self, metrics):
        """Nginx ワーカープロセス数調整"""
        cpu_count = psutil.cpu_count()
        current_workers = self.get_nginx_workers()
        optimal_workers = cpu_count
        
        if metrics['cpu_percent'] > 80 and current_workers < cpu_count * 2:
            # CPU使用率が高い場合はワーカー数を増やす
            new_workers = min(current_workers + 1, cpu_count * 2)
            self.set_nginx_workers(new_workers)
            self.logger.info(f"Increased nginx workers from {current_workers} to {new_workers}")
            
        elif metrics['cpu_percent'] < 30 and current_workers > cpu_count:
            # CPU使用率が低い場合はワーカー数を減らす
            new_workers = max(current_workers - 1, cpu_count)
            self.set_nginx_workers(new_workers)
            self.logger.info(f"Decreased nginx workers from {current_workers} to {new_workers}")
    
    def get_nginx_workers(self):
        """現在のNginx ワーカー数取得"""
        try:
            result = subprocess.run(['pgrep', '-c', 'nginx:'], capture_output=True, text=True)
            return int(result.stdout.strip()) - 1  # マスタープロセスを除く
        except:
            return 1
    
    def set_nginx_workers(self, worker_count):
        """Nginx ワーカー数設定"""
        try:
            # nginx.confの worker_processes を更新
            subprocess.run([
                'sed', '-i', 
                f's/worker_processes.*/worker_processes {worker_count};/',
                '/etc/nginx/nginx.conf'
            ])
            
            # Nginx リロード
            subprocess.run(['nginx', '-s', 'reload'])
            
        except Exception as e:
            self.logger.error(f"Failed to set nginx workers: {e}")
    
    def tune_postgresql_connections(self, metrics):
        """PostgreSQL 接続数調整"""
        try:
            # 現在の接続数確認
            result = subprocess.run([
                'psql', '-t', '-c', 'SELECT count(*) FROM pg_stat_activity;'
            ], capture_output=True, text=True)
            
            current_connections = int(result.stdout.strip())
            
            # max_connections 確認
            result = subprocess.run([
                'psql', '-t', '-c', 'SHOW max_connections;'
            ], capture_output=True, text=True)
            
            max_connections = int(result.stdout.strip())
            connection_ratio = current_connections / max_connections
            
            if connection_ratio > 0.8:
                # 接続数が上限に近い場合の対処
                self.logger.warning(f"High connection usage: {current_connections}/{max_connections}")
                
                # アイドル接続の終了
                subprocess.run([
                    'psql', '-c', 
                    "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < now() - interval '30 minutes';"
                ])
                
                self.logger.info("Terminated idle connections")
                
        except Exception as e:
            self.logger.error(f"Failed to tune PostgreSQL connections: {e}")
    
    def tune_system_swappiness(self, metrics):
        """システムスワップ設定調整"""
        if metrics['memory_percent'] > 90:
            # メモリ使用率が高い場合はスワップを有効化
            current_swappiness = self.get_swappiness()
            if current_swappiness < 60:
                self.set_swappiness(60)
                self.logger.info(f"Increased swappiness from {current_swappiness} to 60")
                
        elif metrics['memory_percent'] < 50:
            # メモリ使用率が低い場合はスワップを最小化
            current_swappiness = self.get_swappiness()
            if current_swappiness > 10:
                self.set_swappiness(10)
                self.logger.info(f"Decreased swappiness from {current_swappiness} to 10")
    
    def get_swappiness(self):
        """現在のswappiness値取得"""
        try:
            with open('/proc/sys/vm/swappiness', 'r') as f:
                return int(f.read().strip())
        except:
            return 60  # デフォルト値
    
    def set_swappiness(self, value):
        """swappiness値設定"""
        try:
            subprocess.run(['sysctl', f'vm.swappiness={value}'])
        except Exception as e:
            self.logger.error(f"Failed to set swappiness: {e}")
    
    def run_tuning_cycle(self):
        """チューニングサイクル実行"""
        self.logger.info("Starting auto-tuning cycle")
        
        # システムメトリクス取得
        metrics = self.get_system_metrics()
        self.logger.info(f"System metrics: {metrics}")
        
        # 各種チューニング実行
        self.tune_nginx_workers(metrics)
        self.tune_postgresql_connections(metrics)
        self.tune_system_swappiness(metrics)
        
        self.logger.info("Auto-tuning cycle completed")

if __name__ == "__main__":
    tuner = AutoPerformanceTuner()
    tuner.run_tuning_cycle()
```

### 4.2 手動チューニング手順

#### 4.2.1 データベースチューニング

```bash
#!/bin/bash
# database_performance_tuning.sh

echo "=== データベース性能チューニング $(date) ==="

# 1. スロークエリ分析
echo "## スロークエリ分析"
psql -c "
SELECT query, calls, total_time, mean_time, rows
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
" | tee /tmp/slow_queries.txt

# 2. インデックス使用状況確認
echo "## インデックス使用状況"
psql -c "
SELECT schemaname, tablename, indexname, 
       idx_tup_read, idx_tup_fetch,
       idx_tup_read::float / NULLIF(idx_tup_fetch, 0) as read_fetch_ratio
FROM pg_stat_user_indexes 
ORDER BY idx_tup_read DESC;
"

# 3. 未使用インデックス検出
echo "## 未使用インデックス"
psql -c "
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes 
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
"

# 4. テーブルサイズ確認
echo "## テーブルサイズ"
psql -c "
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) as index_size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
"

# 5. VACUUM・ANALYZE推奨
echo "## VACUUM・ANALYZE推奨テーブル"
psql -c "
SELECT schemaname, tablename, n_dead_tup, n_live_tup,
       round(n_dead_tup::float / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables 
WHERE n_dead_tup > 1000
ORDER BY dead_ratio DESC;
"

# 6. 接続プール設定確認
echo "## 接続プール設定"
psql -c "
SELECT setting, unit, context 
FROM pg_settings 
WHERE name IN ('max_connections', 'shared_buffers', 'effective_cache_size', 'work_mem', 'maintenance_work_mem');
"

# 7. チューニング推奨事項出力
echo "## チューニング推奨事項"
python3 - << 'EOF'
import subprocess

# スロークエリ数確認
result = subprocess.run(['psql', '-t', '-c', 'SELECT count(*) FROM pg_stat_statements WHERE mean_time > 1000;'], 
                       capture_output=True, text=True)
slow_query_count = int(result.stdout.strip()) if result.stdout.strip().isdigit() else 0

if slow_query_count > 10:
    print("- スロークエリが多数検出されました。クエリの最適化を検討してください。")

# 未使用インデックス確認
result = subprocess.run(['psql', '-t', '-c', 'SELECT count(*) FROM pg_stat_user_indexes WHERE idx_scan = 0;'], 
                       capture_output=True, text=True)
unused_index_count = int(result.stdout.strip()) if result.stdout.strip().isdigit() else 0

if unused_index_count > 0:
    print(f"- {unused_index_count}個の未使用インデックスが見つかりました。削除を検討してください。")

# デッドタプル確認
result = subprocess.run(['psql', '-t', '-c', 'SELECT count(*) FROM pg_stat_user_tables WHERE n_dead_tup > n_live_tup * 0.1;'], 
                       capture_output=True, text=True)
vacuum_needed_count = int(result.stdout.strip()) if result.stdout.strip().isdigit() else 0

if vacuum_needed_count > 0:
    print(f"- {vacuum_needed_count}個のテーブルでVACUUMが必要です。")

print("- 詳細な分析結果は上記の出力を参照してください。")
EOF

echo "$(date): Database performance tuning analysis completed" >> /var/log/performance/tuning.log
```

#### 4.2.2 Webサーバーチューニング

```bash
#!/bin/bash
# nginx_performance_tuning.sh

echo "=== Nginx 性能チューニング $(date) ==="

# 1. 現在の設定確認
echo "## 現在の設定"
nginx -T | grep -E "(worker_processes|worker_connections|keepalive_timeout|client_max_body_size)"

# 2. 接続状況分析
echo "## 接続状況分析"
echo "アクティブ接続数: $(netstat -an | grep :80 | grep ESTABLISHED | wc -l)"
echo "TIME_WAIT状態の接続数: $(netstat -an | grep :80 | grep TIME_WAIT | wc -l)"
echo "LISTEN状態のポート: $(netstat -ln | grep :80)"

# 3. アクセスログ分析
echo "## アクセスログ分析（過去1時間）"
python3 - << 'EOF'
import re
from collections import Counter
from datetime import datetime, timedelta

# 過去1時間のログを分析
one_hour_ago = datetime.now() - timedelta(hours=1)
log_pattern = r'(\S+) - - \[([^\]]+)\] "(\S+) ([^"]*)" (\d+) (\d+) "([^"]*)" "([^"]*)"'

requests_per_minute = Counter()
status_codes = Counter()
user_agents = Counter()
top_urls = Counter()
response_sizes = []

try:
    with open('/var/log/nginx/access.log', 'r') as f:
        for line in f:
            match = re.match(log_pattern, line)
            if not match:
                continue
                
            ip, timestamp, method, url, status, size, referer, user_agent = match.groups()
            
            # 時間解析（簡易）
            minute = timestamp[:17]  # "DD/Mon/YYYY:HH:MM"
            requests_per_minute[minute] += 1
            
            status_codes[status] += 1
            user_agents[user_agent] += 1
            top_urls[url] += 1
            
            if size.isdigit():
                response_sizes.append(int(size))

    print(f"Total requests analyzed: {sum(requests_per_minute.values())}")
    print(f"Peak requests per minute: {max(requests_per_minute.values()) if requests_per_minute else 0}")
    
    print("\nTop 5 status codes:")
    for status, count in status_codes.most_common(5):
        print(f"  {status}: {count}")
    
    print("\nTop 5 requested URLs:")
    for url, count in top_urls.most_common(5):
        print(f"  {url}: {count}")
        
    if response_sizes:
        avg_size = sum(response_sizes) / len(response_sizes)
        print(f"\nAverage response size: {avg_size:.0f} bytes")

except FileNotFoundError:
    print("Access log file not found")
except Exception as e:
    print(f"Error analyzing logs: {e}")
EOF

# 4. 最適化推奨設定生成
echo "## 最適化推奨設定"
CPU_CORES=$(nproc)
MEMORY_GB=$(free -g | awk 'NR==2{print $2}')

cat << EOF

# 推奨nginx設定 (/etc/nginx/nginx.conf)
worker_processes $CPU_CORES;
worker_connections $(($MEMORY_GB * 1024));
worker_rlimit_nofile $(($CPU_CORES * 2048));

events {
    worker_connections $(($MEMORY_GB * 1024));
    use epoll;
    multi_accept on;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 30;
    keepalive_requests 1000;
    
    client_max_body_size 10M;
    client_body_buffer_size 256k;
    client_header_buffer_size 4k;
    large_client_header_buffers 8 64k;
    
    gzip on;
    gzip_vary on;
    gzip_min_length 1000;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    
    # Connection pooling
    upstream backend {
        server 127.0.0.1:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }
}

EOF

echo "$(date): Nginx performance tuning analysis completed" >> /var/log/performance/tuning.log
```

## 5. キャパシティプランニング

### 5.1 成長予測

#### 5.1.1 リソース使用量予測

```python
#!/usr/bin/env python3
# capacity_planning.py

import pandas as pd
import numpy as np
from sklearn.linear_model import LinearRegression
from datetime import datetime, timedelta
import matplotlib.pyplot as plt
import json

class CapacityPlanner:
    def __init__(self):
        self.data_file = "/var/log/performance/capacity_data.csv"
        self.prediction_months = 12
        
    def collect_historical_data(self):
        """過去のデータ収集"""
        # 簡易実装：実際は既存の監視データから取得
        data = []
        base_date = datetime.now() - timedelta(days=90)
        
        for i in range(90):
            date = base_date + timedelta(days=i)
            # 成長トレンドをシミュレート
            growth_factor = 1 + (i / 90) * 0.3  # 30%成長
            noise = np.random.normal(0, 0.1)
            
            cpu_usage = min(95, 40 * growth_factor + noise * 10)
            memory_usage = min(95, 50 * growth_factor + noise * 15)
            disk_usage = min(95, 30 * growth_factor + noise * 5)
            user_count = int(100 * growth_factor + noise * 20)
            
            data.append({
                'date': date.strftime('%Y-%m-%d'),
                'cpu_usage': max(0, cpu_usage),
                'memory_usage': max(0, memory_usage),
                'disk_usage': max(0, disk_usage),
                'user_count': max(0, user_count)
            })
        
        return pd.DataFrame(data)
    
    def predict_capacity_needs(self, df):
        """容量需要予測"""
        df['date'] = pd.to_datetime(df['date'])
        df['days_from_start'] = (df['date'] - df['date'].min()).dt.days
        
        predictions = {}
        
        for metric in ['cpu_usage', 'memory_usage', 'disk_usage', 'user_count']:
            # 線形回帰モデル
            X = df[['days_from_start']]
            y = df[metric]
            
            model = LinearRegression()
            model.fit(X, y)
            
            # 未来の予測
            future_days = []
            future_predictions = []
            
            start_day = df['days_from_start'].max()
            for month in range(1, self.prediction_months + 1):
                future_day = start_day + (month * 30)
                future_days.append(future_day)
                prediction = model.predict([[future_day]])[0]
                future_predictions.append(max(0, prediction))
            
            predictions[metric] = {
                'current': df[metric].iloc[-1],
                'trend': model.coef_[0],
                'future_values': future_predictions,
                'months': list(range(1, self.prediction_months + 1))
            }
        
        return predictions
    
    def identify_scaling_points(self, predictions):
        """スケーリング必要時期の特定"""
        scaling_recommendations = {}
        
        thresholds = {
            'cpu_usage': 80,
            'memory_usage': 85,
            'disk_usage': 90,
            'user_count': 1000  # 想定最大ユーザー数
        }
        
        for metric, data in predictions.items():
            threshold = thresholds.get(metric, 100)
            scaling_month = None
            
            for i, value in enumerate(data['future_values']):
                if value >= threshold:
                    scaling_month = i + 1
                    break
            
            scaling_recommendations[metric] = {
                'threshold': threshold,
                'scaling_needed_in_months': scaling_month,
                'current_utilization': data['current'],
                'projected_growth_rate': data['trend'] * 30  # 月次成長率
            }
        
        return scaling_recommendations
    
    def generate_capacity_report(self):
        """容量計画レポート生成"""
        # データ収集
        df = self.collect_historical_data()
        
        # 予測実行
        predictions = self.predict_capacity_needs(df)
        
        # スケーリング推奨事項
        scaling_recommendations = self.identify_scaling_points(predictions)
        
        # レポート生成
        report = {
            'generated_at': datetime.now().isoformat(),
            'analysis_period_days': len(df),
            'predictions': predictions,
            'scaling_recommendations': scaling_recommendations,
            'summary': self.generate_summary(scaling_recommendations)
        }
        
        # JSON形式で保存
        report_file = f"/var/log/performance/capacity_report_{datetime.now().strftime('%Y%m%d')}.json"
        with open(report_file, 'w') as f:
            json.dump(report, f, indent=2)
        
        # テキスト形式のサマリー生成
        self.generate_text_report(report, report_file.replace('.json', '.txt'))
        
        return report
    
    def generate_summary(self, scaling_recommendations):
        """サマリー生成"""
        urgent_items = []
        medium_term_items = []
        long_term_items = []
        
        for metric, data in scaling_recommendations.items():
            if data['scaling_needed_in_months']:
                if data['scaling_needed_in_months'] <= 3:
                    urgent_items.append(f"{metric}: {data['scaling_needed_in_months']}ヶ月後")
                elif data['scaling_needed_in_months'] <= 6:
                    medium_term_items.append(f"{metric}: {data['scaling_needed_in_months']}ヶ月後")
                else:
                    long_term_items.append(f"{metric}: {data['scaling_needed_in_months']}ヶ月後")
        
        return {
            'urgent_actions_needed': urgent_items,
            'medium_term_planning': medium_term_items,
            'long_term_monitoring': long_term_items
        }
    
    def generate_text_report(self, report, output_file):
        """テキスト形式レポート生成"""
        with open(output_file, 'w', encoding='utf-8') as f:
            f.write("=== キャパシティプランニング レポート ===\n")
            f.write(f"生成日時: {report['generated_at']}\n")
            f.write(f"分析期間: {report['analysis_period_days']}日間\n\n")
            
            f.write("## 緊急対応が必要な項目\n")
            for item in report['summary']['urgent_actions_needed']:
                f.write(f"- {item}\n")
            if not report['summary']['urgent_actions_needed']:
                f.write("- なし\n")
            
            f.write("\n## 中期的計画が必要な項目\n")
            for item in report['summary']['medium_term_planning']:
                f.write(f"- {item}\n")
            if not report['summary']['medium_term_planning']:
                f.write("- なし\n")
            
            f.write("\n## 長期的監視項目\n")
            for item in report['summary']['long_term_monitoring']:
                f.write(f"- {item}\n")
            if not report['summary']['long_term_monitoring']:
                f.write("- なし\n")
            
            f.write("\n## 詳細予測データ\n")
            for metric, data in report['scaling_recommendations'].items():
                f.write(f"\n### {metric}\n")
                f.write(f"- 現在の使用率: {data['current_utilization']:.1f}%\n")
                f.write(f"- 月次成長率: {data['projected_growth_rate']:.1f}%\n")
                f.write(f"- 閾値: {data['threshold']}%\n")
                if data['scaling_needed_in_months']:
                    f.write(f"- スケーリング必要時期: {data['scaling_needed_in_months']}ヶ月後\n")
                else:
                    f.write("- スケーリング: 当面不要\n")

if __name__ == "__main__":
    planner = CapacityPlanner()
    report = planner.generate_capacity_report()
    print("Capacity planning report generated successfully")
    
    # 緊急対応が必要な場合はアラート送信
    if report['summary']['urgent_actions_needed']:
        print("URGENT: Immediate capacity planning actions required!")
        # メールアラート送信処理
```

---

**注意**: この性能管理手順書は、システムの最適な性能を維持するための重要な手順を定義しています。環境や要件に応じて設定値や閾値を調整してください。