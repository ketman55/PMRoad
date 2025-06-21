# 監視設定書

**タイトル**: 監視設定書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: system-architecture.md, performance-design.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの監視設定、閾値、アラート条件を定義し、プロアクティブなシステム運用を実現することを目的とする。

### 1.2 適用範囲
本監視設定は、システム全体に適用される：
- インフラストラクチャ監視
- アプリケーション監視
- ネットワーク監視
- セキュリティ監視

### 1.3 監視方針
- **階層化監視**: インフラ、ミドルウェア、アプリケーション層での多層監視
- **予防的監視**: 問題発生前の早期検知
- **自動化**: 可能な限りの監視・対応自動化
- **可視化**: ダッシュボードによる状況の可視化

## 2. 監視アーキテクチャ

### 2.1 監視システム構成

```
┌─────────────────────────────────────────────────────────┐
│                    Monitoring Stack                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Grafana    │  │Prometheus   │  │AlertManager │    │
│  │(Dashboard)  │  │(Metrics)    │  │(Alerting)   │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│         │                 │                 │          │
│         └─────────────────┼─────────────────┘          │
│                           │                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   ELK       │  │  Jaeger     │  │   Nagios    │    │
│  │ (Logs)      │  │ (Tracing)   │  │(Legacy)     │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │    Web      │    │    App      │    │  Database   │
   │  Servers    │    │  Servers    │    │  Servers    │
   └─────────────┘    └─────────────┘    └─────────────┘
```

### 2.2 監視ツール選定

| ツール | 用途 | 対象 | 保持期間 |
|---|---|---|---|
| Prometheus | メトリクス収集・保存 | システム・アプリ | 90日 |
| Grafana | ダッシュボード・可視化 | 全般 | - |
| Elasticsearch | ログ収集・検索 | アプリ・システム | 1年 |
| Logstash | ログ処理・変換 | ログデータ | - |
| Kibana | ログ分析・可視化 | ログデータ | - |
| AlertManager | アラート管理・通知 | 全般 | 30日 |
| Jaeger | 分散トレーシング | アプリケーション | 7日 |

## 3. メトリクス監視設定

### 3.1 システムメトリクス

#### 3.1.1 CPU監視

**監視項目**
- CPU使用率（全体・コア別）
- ロードアベレージ（1分・5分・15分）
- CPU待機時間（iowait・steal）

**Prometheus設定**
```yaml
# prometheus.yml
- job_name: 'node-exporter'
  static_configs:
    - targets:
      - 'web-server:9100'
      - 'app-server:9100'
      - 'db-server:9100'
  scrape_interval: 30s
  metrics_path: /metrics
```

**アラートルール**
```yaml
# alerts.yml
groups:
  - name: cpu_alerts
    rules:
      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for more than 5 minutes"

      - alert: CriticalCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 90
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Critical CPU usage detected"
          description: "CPU usage is above 90% for more than 2 minutes"
```

#### 3.1.2 メモリ監視

**監視項目**
- メモリ使用率・使用量
- スワップ使用率・使用量
- メモリキャッシュ・バッファ使用量

**アラートルール**
```yaml
- name: memory_alerts
  rules:
    - alert: HighMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage detected"
        description: "Memory usage is above 80%"

    - alert: CriticalMemoryUsage
      expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 90
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Critical memory usage detected"
        description: "Memory usage is above 90%"
```

#### 3.1.3 ディスク監視

**監視項目**
- ディスク使用率・使用量
- ディスクI/O（読み書き速度・IOPS）
- ディスクレスポンス時間

**アラートルール**
```yaml
- name: disk_alerts
  rules:
    - alert: HighDiskUsage
      expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 85
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High disk usage detected"
        description: "Disk usage is above 85% on {{ $labels.mountpoint }}"

    - alert: DiskIOHigh
      expr: rate(node_disk_io_time_seconds_total[5m]) * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High disk I/O detected"
        description: "Disk I/O utilization is above 80%"
```

### 3.2 ネットワーク監視

#### 3.2.1 ネットワークメトリクス

**監視項目**
- ネットワーク使用率（送受信）
- パケットエラー率
- TCP接続数・接続状態

**アラートルール**
```yaml
- name: network_alerts
  rules:
    - alert: HighNetworkUsage
      expr: rate(node_network_receive_bytes_total[5m]) + rate(node_network_transmit_bytes_total[5m]) > 100000000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High network usage detected"
        description: "Network usage is above 100MB/s"

    - alert: NetworkErrors
      expr: rate(node_network_receive_errs_total[5m]) > 0.01
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Network errors detected"
        description: "Network interface {{ $labels.device }} has errors"
```

### 3.3 アプリケーション監視

#### 3.3.1 アプリケーションメトリクス

**監視項目**
- HTTPリクエスト数・レスポンス時間
- HTTPステータスコード分布
- アプリケーションエラー率
- アクティブユーザー数

**アプリケーション設定例（Spring Boot）**
```java
// メトリクス公開設定
@Component
public class ApplicationMetrics {
    
    private final Counter requestCounter;
    private final Timer responseTimer;
    private final Gauge activeUsers;
    
    public ApplicationMetrics(MeterRegistry meterRegistry) {
        this.requestCounter = Counter.builder("http_requests_total")
                .tag("application", "main-app")
                .register(meterRegistry);
                
        this.responseTimer = Timer.builder("http_request_duration_seconds")
                .tag("application", "main-app")
                .register(meterRegistry);
                
        this.activeUsers = Gauge.builder("active_users_total")
                .tag("application", "main-app")
                .register(meterRegistry, this, ApplicationMetrics::getActiveUserCount);
    }
}
```

**アラートルール**
```yaml
- name: application_alerts
  rules:
    - alert: HighResponseTime
      expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High response time detected"
        description: "95th percentile response time is above 3 seconds"

    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is above 5%"
```

### 3.4 データベース監視

#### 3.4.1 PostgreSQL監視

**監視項目**
- 接続数・アクティブ接続数
- クエリ実行時間・待機時間
- デッドロック・ロック待機
- テーブル・インデックスサイズ

**postgres_exporter設定**
```yaml
# docker-compose.yml
postgres_exporter:
  image: prometheuscommunity/postgres-exporter
  environment:
    DATA_SOURCE_NAME: "postgresql://monitoring_user:password@postgres:5432/database?sslmode=disable"
  ports:
    - "9187:9187"
```

**アラートルール**
```yaml
- name: postgresql_alerts
  rules:
    - alert: PostgreSQLDown
      expr: pg_up == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "PostgreSQL is down"
        description: "PostgreSQL instance {{ $labels.instance }} is down"

    - alert: PostgreSQLHighConnections
      expr: pg_stat_database_numbackends / pg_settings_max_connections * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High PostgreSQL connections"
        description: "PostgreSQL connections are above 80%"

    - alert: PostgreSQLSlowQueries
      expr: rate(pg_stat_activity_max_tx_duration{state="active"}[5m]) > 30
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "PostgreSQL slow queries detected"
        description: "Long running queries detected"
```

## 4. ログ監視設定

### 4.1 ログ収集設定

#### 4.1.1 Logstash設定

```ruby
# /etc/logstash/conf.d/application.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][log_type] == "application" {
    grok {
      match => { 
        "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:thread}\] %{DATA:logger} - %{GREEDYDATA:message}"
      }
    }
    
    date {
      match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
    }
    
    if [level] == "ERROR" {
      mutate {
        add_tag => [ "error" ]
      }
    }
  }
  
  if [fields][log_type] == "nginx" {
    grok {
      match => { 
        "message" => "%{NGINXACCESS}"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

#### 4.1.2 Filebeat設定

```yaml
# /etc/filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/application/*.log
    fields:
      log_type: application
    fields_under_root: true
    multiline.pattern: '^\d{4}-\d{2}-\d{2}'
    multiline.negate: true
    multiline.match: after

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
    fields:
      log_type: nginx_access
    fields_under_root: true

  - type: log
    enabled: true
    paths:
      - /var/log/nginx/error.log
    fields:
      log_type: nginx_error
    fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_host_metadata:
      when.not.contains.tags: forwarded
```

### 4.2 ログアラート設定

#### 4.2.1 ElastAlert設定

```yaml
# /etc/elastalert/rules/application_errors.yml
name: Application Error Alert
type: frequency
index: filebeat-*
num_events: 10
timeframe:
  minutes: 5

filter:
  - terms:
      level: ["ERROR", "FATAL"]

alert:
  - "email"
  - "slack"

email:
  - "admin@example.com"

slack:
webhook_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
slack_channel_override: "#alerts"
slack_username_override: "ElastAlert"

alert_text: |
  Application errors detected:
  {0} errors in the last 5 minutes
  
  Query: {1}
  
alert_text_type: alert_text_only
```

## 5. セキュリティ監視

### 5.1 セキュリティメトリクス

#### 5.1.1 認証・認可監視

**監視項目**
- ログイン失敗回数
- 異常なアクセスパターン
- 権限昇格試行
- セッション異常

**ログアラート設定**
```yaml
# /etc/elastalert/rules/security_alerts.yml
name: Failed Login Alert
type: frequency
index: filebeat-*
num_events: 5
timeframe:
  minutes: 5

filter:
  - query:
      query_string:
        query: 'message:"Failed password" OR message:"authentication failure"'

alert:
  - "email"
  - "slack"

alert_text: |
  Multiple failed login attempts detected:
  {0} failed attempts in the last 5 minutes
  
  This may indicate a brute force attack.
```

#### 5.1.2 ネットワークセキュリティ監視

**監視項目**
- 不正アクセス試行
- DDoS攻撃パターン
- ポートスキャン
- 異常な通信パターン

## 6. ダッシュボード設定

### 6.1 Grafanaダッシュボード

#### 6.1.1 システム概要ダッシュボード

**パネル構成**
- システム稼働状況（UP/DOWN）
- CPU・メモリ・ディスク使用率
- ネットワーク使用量
- アクティブアラート数

**JSON設定例**
```json
{
  "dashboard": {
    "title": "System Overview",
    "panels": [
      {
        "title": "CPU Usage",
        "type": "stat",
        "targets": [
          {
            "expr": "100 - (avg by(instance) (rate(node_cpu_seconds_total{mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "{{instance}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 70},
                {"color": "red", "value": 90}
              ]
            }
          }
        }
      }
    ]
  }
}
```

#### 6.1.2 アプリケーションダッシュボード

**パネル構成**
- リクエスト数・レスポンス時間
- エラー率・ステータスコード分布
- アクティブユーザー数
- データベース接続数

### 6.2 アラート通知設定

#### 6.2.1 通知チャネル設定

**Slack通知**
```yaml
# alertmanager.yml
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
        channel: '#alerts'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          *Instance:* {{ .Labels.instance }}
          {{ end }}
```

**メール通知**
```yaml
receivers:
  - name: 'email'
    email_configs:
      - to: 'admin@example.com'
        from: 'alertmanager@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alertmanager@example.com'
        auth_password: 'password'
        subject: 'Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Severity: {{ .Labels.severity }}
          Instance: {{ .Labels.instance }}
          {{ end }}
```

## 7. 監視運用手順

### 7.1 監視データ管理

#### 7.1.1 データ保持ポリシー

| データ種別 | 保持期間 | 圧縮 | アーカイブ |
|---|---|---|---|
| メトリクス（高解像度） | 7日 | なし | なし |
| メトリクス（中解像度） | 30日 | 5:1 | なし |
| メトリクス（低解像度） | 90日 | 10:1 | あり |
| ログデータ | 365日 | gzip | S3 |
| アラート履歴 | 30日 | なし | なし |

#### 7.1.2 データクリーンアップ

```bash
#!/bin/bash
# cleanup_monitoring_data.sh

# 古いPrometheusデータの削除
find /var/lib/prometheus -name "*.tmp" -mtime +1 -delete

# 古いログの圧縮とアーカイブ
find /var/log -name "*.log" -mtime +7 -exec gzip {} \;
find /var/log -name "*.log.gz" -mtime +30 -exec aws s3 cp {} s3://backup-bucket/logs/ \;
find /var/log -name "*.log.gz" -mtime +30 -delete

# Elasticsearchインデックスの削除
curator --config /etc/curator/curator.yml /etc/curator/delete_indices.yml
```

### 7.2 監視システム監視

#### 7.2.1 監視システム自体の監視

**監視項目**
- Prometheus・Grafana・Elasticsearch の稼働状況
- データ収集の遅延・欠損
- ディスク使用量・性能

```yaml
# 監視システム監視アラート
- name: monitoring_system_alerts
  rules:
    - alert: PrometheusDown
      expr: up{job="prometheus"} == 0
      for: 1m
      labels:
        severity: critical
      annotations:
        summary: "Prometheus is down"
        description: "Prometheus has been down for more than 1 minute"

    - alert: DataCollectionDelay
      expr: prometheus_tsdb_head_max_time - prometheus_tsdb_head_min_time > 300
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Data collection delay detected"
        description: "Data collection is delayed by more than 5 minutes"
```

---

**注意**: この監視設定書は、システムの健全性を保つための重要な文書です。設定変更時は十分なテストを実施してください。