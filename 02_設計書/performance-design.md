# 性能設計書

**タイトル**: 性能設計書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: system-architecture.md, non-functional-requirements.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムの性能要件を満たすための設計方針と具体的な実装方法を定義し、要求される性能水準を達成することを目的とする。

### 1.2 適用範囲
本性能設計は、システム全体に適用される：
- アプリケーション性能
- データベース性能
- ネットワーク性能
- インフラストラクチャ性能

### 1.3 性能設計原則
- **スケーラビリティ**: 負荷増加に対応可能な設計
- **可用性**: 高い稼働率を維持する設計
- **効率性**: リソースを効率的に活用する設計
- **予測可能性**: 性能が予測可能な設計
- **測定可能性**: 性能を測定・監視可能な設計

## 2. 性能要件分析

### 2.1 性能要件マトリックス

| 性能項目 | 現在値 | 目標値 | 限界値 | 測定方法 |
|---|---|---|---|---|
| **応答時間** |
| オンライン検索 | - | 3秒以内 | 5秒 | APM監視 |
| オンライン更新 | - | 5秒以内 | 8秒 | APM監視 |
| 帳票出力 | - | 10秒以内 | 20秒 | アプリログ |
| **スループット** |
| 同時ユーザー数 | - | 100ユーザー | 150ユーザー | ロードテスト |
| API呼び出し/秒 | - | 500 RPS | 750 RPS | API Gateway |
| バッチ処理件数 | - | 10万件/時 | 15万件/時 | バッチログ |
| **リソース使用率** |
| CPU使用率 | - | 70%以下 | 85% | システム監視 |
| メモリ使用率 | - | 80%以下 | 90% | システム監視 |
| ディスクI/O | - | 80%以下 | 90% | システム監視 |

### 2.2 性能目標の根拠

#### 2.2.1 ユーザビリティ観点
- **3秒ルール**: オンライン検索は3秒以内でユーザー満足度維持
- **5秒ルール**: データ更新は5秒以内で業務効率維持
- **10秒ルール**: 帳票出力は10秒以内で待機許容範囲

#### 2.2.2 ビジネス要件観点
- **同時ユーザー**: ピーク時間帯の想定ユーザー数
- **処理件数**: 業務量増加を考慮した処理能力
- **成長対応**: 3年後の業務拡大を見込んだ設計

## 3. アーキテクチャ性能設計

### 3.1 スケーラビリティ設計

#### 3.1.1 水平スケーリング
```
                    Load Balancer
                    (Nginx/HAProxy)
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼────┐      ┌────▼────┐      ┌────▼────┐
   │ Web/App │      │ Web/App │      │ Web/App │
   │ Server1 │      │ Server2 │      │ Server3 │
   └─────────┘      └─────────┘      └─────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                    ┌────▼────┐
                    │Database │
                    │Cluster  │
                    └─────────┘
```

#### 3.1.2 負荷分散設定
```nginx
# Nginx ロードバランサー設定
upstream app_servers {
    least_conn;
    server app1.example.com:8080 weight=3 max_fails=3 fail_timeout=30s;
    server app2.example.com:8080 weight=3 max_fails=3 fail_timeout=30s;
    server app3.example.com:8080 weight=2 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
}
```

### 3.2 キャッシュ戦略

#### 3.2.1 多層キャッシュ設計
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Browser   │    │     CDN     │    │   Reverse   │
│   Cache     │    │   Cache     │    │   Proxy     │
│   (HTTP)    │    │  (Global)   │    │  (Nginx)    │
└─────────────┘    └─────────────┘    └─────────────┘
                                             │
┌─────────────┐    ┌─────────────┐           │
│Application  │    │   Memory    │           │
│   Cache     │◄───┤   Cache     │◄──────────┘
│ (Database)  │    │  (Redis)    │
└─────────────┘    └─────────────┘
```

#### 3.2.2 キャッシュポリシー
| データ種別 | キャッシュ場所 | TTL | 更新方式 | 無効化条件 |
|---|---|---|---|---|
| 静的コンテンツ | CDN, Browser | 1年 | 手動 | デプロイ時 |
| API レスポンス | Redis | 5分 | 自動 | データ更新時 |
| セッション | Redis | 30分 | 自動 | ログアウト時 |
| マスターデータ | アプリケーション | 1時間 | 自動 | マスター更新時 |
| クエリ結果 | データベース | 10分 | 自動 | テーブル更新時 |

### 3.3 非同期処理設計

#### 3.3.1 非同期処理アーキテクチャ
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │    │   Web API   │    │Message Queue│
│             │    │             │    │  (Redis)    │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       │ 1. Request        │                   │
       ├──────────────────►│                   │
       │                   │ 2. Queue Job      │
       │                   ├──────────────────►│
       │ 3. Job ID         │                   │
       │◄──────────────────┤                   │
       │                   │                   │
       │ 4. Poll Status    │                   │
       ├──────────────────►│                   │
                           │                   │
                    ┌─────────────┐           │
                    │  Worker     │           │
                    │ Processes   │◄──────────┘
                    └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │   Result    │
                    │  Storage    │
                    └─────────────┘
```

#### 3.3.2 非同期処理対象
- **重い処理**: レポート生成、データエクスポート
- **外部連携**: API呼び出し、メール送信
- **バッチ処理**: データ集計、クリーンアップ
- **ファイル処理**: アップロード、画像変換

## 4. データベース性能設計

### 4.1 データベース最適化

#### 4.1.1 インデックス戦略
```sql
-- 基本的なインデックス
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- 複合インデックス（カバリングインデックス）
CREATE INDEX idx_orders_covering ON orders(status, order_date) 
INCLUDE (customer_id, total_amount);

-- 部分インデックス
CREATE INDEX idx_orders_active ON orders(order_date) 
WHERE status IN ('pending', 'processing');

-- 関数インデックス
CREATE INDEX idx_users_upper_email ON users(UPPER(email));
```

#### 4.1.2 クエリ最適化
```sql
-- 悪い例：N+1問題
SELECT * FROM orders WHERE customer_id = ?; -- N回実行

-- 良い例：結合による一括取得
SELECT o.*, c.name as customer_name 
FROM orders o 
JOIN customers c ON o.customer_id = c.id 
WHERE o.order_date >= '2024-01-01';

-- 悪い例：サブクエリ
SELECT * FROM orders 
WHERE customer_id IN (
    SELECT id FROM customers WHERE status = 'active'
);

-- 良い例：結合
SELECT o.* FROM orders o 
JOIN customers c ON o.customer_id = c.id 
WHERE c.status = 'active';
```

### 4.2 データベース分散設計

#### 4.2.1 読み取り専用レプリカ
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Primary   │───►│  Replica1   │    │  Replica2   │
│  Database   │    │ (Read Only) │    │ (Read Only) │
│ (Read/Write)│    └─────────────┘    └─────────────┘
└─────────────┘           │                   │
       ▲                  │                   │
       │                  ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Write     │    │   Report    │    │   Batch     │
│ Operations  │    │   Queries   │    │ Processing  │
└─────────────┘    └─────────────┘    └─────────────┘
```

#### 4.2.2 パーティショニング戦略
```sql
-- 日付範囲パーティショニング
CREATE TABLE orders (
    order_id SERIAL,
    order_date DATE NOT NULL,
    -- 他のカラム
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

-- 月別パーティション
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- ハッシュパーティショニング
CREATE TABLE user_activities (
    activity_id SERIAL,
    user_id INTEGER NOT NULL,
    -- 他のカラム
    PRIMARY KEY (activity_id, user_id)
) PARTITION BY HASH (user_id);
```

### 4.3 接続プール設定

#### 4.3.1 HikariCP設定
```yaml
# アプリケーション設定
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # 最大接続数
      minimum-idle: 5            # 最小アイドル接続数
      connection-timeout: 30000  # 接続タイムアウト（ms）
      idle-timeout: 600000       # アイドルタイムアウト（ms）
      max-lifetime: 1800000      # 接続の最大生存時間（ms）
      leak-detection-threshold: 60000  # リーク検出閾値（ms）
```

#### 4.3.2 接続プール監視
```java
// 接続プール監視メトリクス
@Component
public class DatabaseMetrics {
    
    @Autowired
    private HikariDataSource dataSource;
    
    @EventListener
    @Scheduled(fixedRate = 30000)
    public void recordMetrics() {
        HikariPoolMXBean poolBean = dataSource.getHikariPoolMXBean();
        
        // メトリクス記録
        Metrics.gauge("db.connections.active", poolBean.getActiveConnections());
        Metrics.gauge("db.connections.idle", poolBean.getIdleConnections());
        Metrics.gauge("db.connections.total", poolBean.getTotalConnections());
        Metrics.gauge("db.connections.waiting", poolBean.getThreadsAwaitingConnection());
    }
}
```

## 5. アプリケーション性能設計

### 5.1 アプリケーション最適化

#### 5.1.1 レスポンス最適化
```java
// ページネーション実装
@RestController
public class UserController {
    
    @GetMapping("/users")
    public ResponseEntity<PagedResult<User>> getUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sort) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by(sort));
        Page<User> users = userService.findUsers(pageable);
        
        return ResponseEntity.ok(new PagedResult<>(users));
    }
}

// 非同期処理実装
@Service
public class ReportService {
    
    @Async("taskExecutor")
    public CompletableFuture<String> generateReport(ReportRequest request) {
        // 重い処理
        String reportId = processReport(request);
        return CompletableFuture.completedFuture(reportId);
    }
}
```

#### 5.1.2 メモリ管理
```java
// メモリ効率的な実装
@Service
public class DataProcessingService {
    
    // ストリーム処理による省メモリ実装
    public void processLargeDataset(String filePath) {
        try (Stream<String> lines = Files.lines(Paths.get(filePath))) {
            lines.parallel()
                 .filter(line -> !line.isEmpty())
                 .map(this::parseRecord)
                 .forEach(this::processRecord);
        }
    }
    
    // バッチ処理による省メモリ実装
    public void processBatchData() {
        int batchSize = 1000;
        int offset = 0;
        
        List<Record> batch;
        do {
            batch = dataRepository.findBatch(offset, batchSize);
            processBatch(batch);
            offset += batchSize;
            
            // メモリ解放
            batch.clear();
            System.gc(); // 必要に応じて
            
        } while (batch.size() == batchSize);
    }
}
```

### 5.2 並行処理設計

#### 5.2.1 スレッドプール設定
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    
    @Bean(name = "taskExecutor")
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);        // コアスレッド数
        executor.setMaxPoolSize(50);         // 最大スレッド数
        executor.setQueueCapacity(100);      // キュー容量
        executor.setKeepAliveSeconds(60);    // アイドル時間
        executor.setThreadNamePrefix("async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        return executor;
    }
}
```

#### 5.2.2 非同期処理監視
```java
@Component
public class AsyncMonitoring {
    
    @Autowired
    @Qualifier("taskExecutor")
    private ThreadPoolTaskExecutor taskExecutor;
    
    @Scheduled(fixedRate = 30000)
    public void monitorThreadPool() {
        ThreadPoolExecutor executor = taskExecutor.getThreadPoolExecutor();
        
        Metrics.gauge("thread.pool.active", executor.getActiveCount());
        Metrics.gauge("thread.pool.size", executor.getPoolSize());
        Metrics.gauge("thread.pool.queue", executor.getQueue().size());
        Metrics.gauge("thread.pool.completed", executor.getCompletedTaskCount());
    }
}
```

## 6. ネットワーク性能設計

### 6.1 CDN設計

#### 6.1.1 CDN配置戦略
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Tokyo     │    │   Osaka     │    │  Overseas   │
│  CDN Edge   │    │  CDN Edge   │    │  CDN Edge   │
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
       └─────────────────────┼─────────────────┘
                             │
                    ┌─────────────┐
                    │   Origin    │
                    │   Server    │
                    └─────────────┘
```

#### 6.1.2 キャッシュ最適化
```http
# HTTP ヘッダー設定
Cache-Control: public, max-age=31536000, immutable  # 静的リソース
Cache-Control: public, max-age=300, stale-while-revalidate=60  # API
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Last-Modified: Wed, 21 Oct 2015 07:28:00 GMT
```

### 6.2 ネットワーク最適化

#### 6.2.1 HTTP/2設定
```nginx
server {
    listen 443 ssl http2;
    
    # SSL設定
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    
    # HTTP/2 Server Push
    location / {
        http2_push /css/style.css;
        http2_push /js/app.js;
        proxy_pass http://backend;
    }
    
    # 圧縮設定
    gzip on;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript;
}
```

#### 6.2.2 接続プーリング
```java
// HTTP クライアント設定
@Configuration
public class HttpClientConfig {
    
    @Bean
    public CloseableHttpClient httpClient() {
        return HttpClients.custom()
                .setMaxConnTotal(200)           // 最大接続数
                .setMaxConnPerRoute(20)         // ルート別最大接続数
                .setConnectionTimeToLive(60, TimeUnit.SECONDS)
                .build();
    }
}
```

## 7. 性能監視設計

### 7.1 APM実装

#### 7.1.1 メトリクス収集
```java
@RestController
@Timed  // Micrometer アノテーション
public class UserController {
    
    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer responseTimer;
    
    public UserController(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.requestCounter = Counter.builder("api.requests")
                .tag("endpoint", "users")
                .register(meterRegistry);
        this.responseTimer = Timer.builder("api.response.time")
                .tag("endpoint", "users")
                .register(meterRegistry);
    }
    
    @GetMapping("/users/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            requestCounter.increment();
            User user = userService.findById(id);
            return ResponseEntity.ok(user);
        } finally {
            sample.stop(responseTimer);
        }
    }
}
```

#### 7.1.2 カスタムメトリクス
```java
@Component
public class BusinessMetrics {
    
    private final Counter orderCounter;
    private final Gauge activeUserGauge;
    private final Timer orderProcessingTimer;
    
    public BusinessMetrics(MeterRegistry meterRegistry) {
        this.orderCounter = Counter.builder("business.orders.total")
                .register(meterRegistry);
        
        this.activeUserGauge = Gauge.builder("business.users.active")
                .register(meterRegistry, this, BusinessMetrics::getActiveUserCount);
                
        this.orderProcessingTimer = Timer.builder("business.orders.processing.time")
                .register(meterRegistry);
    }
    
    public double getActiveUserCount() {
        return userService.getActiveUserCount();
    }
}
```

### 7.2 性能アラート

#### 7.2.1 アラート設定
```yaml
# Prometheus アラートルール
groups:
  - name: performance
    rules:
      - alert: HighResponseTime
        expr: http_request_duration_seconds{quantile="0.95"} > 3
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "高い応答時間が検出されました"
          
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "エラー率が高くなっています"
          
      - alert: HighCPUUsage
        expr: cpu_usage_percent > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU使用率が高くなっています"
```

## 8. 性能テスト設計

### 8.1 負荷テスト計画

#### 8.1.1 テストシナリオ
| テスト種別 | 目的 | 負荷パターン | 実行時間 | 成功基準 |
|---|---|---|---|---|
| 負荷テスト | 通常負荷での性能確認 | 100ユーザー同時 | 30分 | レスポンス時間＜3秒 |
| ストレステスト | 限界性能の確認 | 段階的増加 | 60分 | システム正常動作 |
| スパイクテスト | 急激な負荷増加対応 | 急激な増加 | 15分 | システム復旧 |
| 持久テスト | 長時間動作の安定性 | 80ユーザー同時 | 8時間 | メモリリーク無し |

#### 8.1.2 JMeterテストプラン
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan testname="API Performance Test">
      <elementProp name="TestPlan.arguments" elementType="Arguments"/>
    </TestPlan>
    <hashTree>
      <ThreadGroup testname="User Load">
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">300</stringProp>
        <stringProp name="ThreadGroup.duration">1800</stringProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy testname="User Login">
          <stringProp name="HTTPSampler.domain">api.example.com</stringProp>
          <stringProp name="HTTPSampler.path">/v1/auth/login</stringProp>
          <stringProp name="HTTPSampler.method">POST</stringProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

### 8.2 性能チューニング

#### 8.2.1 チューニング計画
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Baseline   │───►│  Analysis   │───►│  Tuning     │
│ Performance │    │   & Plan    │    │ & Testing   │
│ Measurement │    └─────────────┘    └─────────────┘
└─────────────┘           │                   │
       ▲                  ▼                   ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Validation │◄───┤ Bottleneck  │    │ Performance │
│     Test    │    │Identification│    │Optimization │
└─────────────┘    └─────────────┘    └─────────────┘
```

#### 8.2.2 チューニングチェックリスト
- [ ] データベースクエリ最適化
- [ ] インデックス追加・見直し
- [ ] アプリケーションキャッシュ実装
- [ ] 不要な処理の削除
- [ ] 並行処理の最適化
- [ ] メモリ使用量最適化
- [ ] ネットワーク設定最適化
- [ ] JVM パラメータ調整

---

**注意**: この性能設計書は、システムの性能要件達成のための重要な文書です。実装時は継続的な性能監視と改善を実施してください。