# データベース設計書

**タイトル**: データベース設計書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: system-design.md, functional-requirements.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システムで使用するデータベースの論理設計および物理設計を定義し、データの整合性、性能、保守性を確保することを目的とする。

### 1.2 適用範囲
本データベース設計は、[システム名]で使用する全てのデータベースに適用される。

### 1.3 設計方針
- **正規化**: 第3正規形までの正規化実施
- **整合性**: 参照整合性制約の適切な設定
- **性能**: インデックス設計による検索性能最適化
- **拡張性**: 将来の データ量増加に対応可能な設計
- **セキュリティ**: アクセス制御と暗号化の実装

## 2. データベース全体構成

### 2.1 データベース構成図

```
┌─────────────────────────────────────────────────────────┐
│                Main Database (PostgreSQL)               │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │   User Data   │  │Business Data  │  │Master Data  │ │
│  │               │  │               │  │             │ │
│  │ ・users       │  │ ・transactions│  │ ・codes     │ │
│  │ ・roles       │  │ ・orders      │  │ ・settings  │ │
│  │ ・permissions │  │ ・products    │  │ ・categories│ │
│  └───────────────┘  └───────────────┘  └─────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                 Log Database (MongoDB)                  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │ Access Logs   │  │ Audit Logs    │  │System Logs  │ │
│  │               │  │               │  │             │ │
│  │ ・login_logs  │  │ ・user_actions│  │ ・app_logs  │ │
│  │ ・api_logs    │  │ ・data_changes│  │ ・error_logs│ │
│  │ ・access_hist │  │ ・permission  │  │ ・perf_logs │ │
│  └───────────────┘  └───────────────┘  └─────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                Cache Database (Redis)                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────┐ │
│  │   Sessions    │  │   Cache       │  │  Temporary  │ │
│  │               │  │               │  │             │ │
│  │ ・user_session│  │ ・query_cache │  │ ・temp_data │ │
│  │ ・auth_tokens │  │ ・page_cache  │  │ ・locks     │ │
│  │ ・csrf_tokens │  │ ・api_cache   │  │ ・counters  │ │
│  └───────────────┘  └───────────────┘  └─────────────┘ │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 2.2 データベース分割方針

#### 2.2.1 機能別分割
- **ユーザー管理**: ユーザー、ロール、権限関連
- **業務データ**: 業務固有のトランザクションデータ
- **マスターデータ**: コード値、設定値等の基礎データ
- **ログデータ**: アクセスログ、監査ログ、システムログ

#### 2.2.2 用途別分割
- **OLTP**: オンライン業務処理用（PostgreSQL）
- **ログ蓄積**: ログデータ蓄積用（MongoDB）
- **キャッシュ**: 高速アクセス用（Redis）

## 3. 論理データモデル

### 3.1 ER図（概要）

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│    USERS    │     │    ROLES    │     │ PERMISSIONS │
│             │     │             │     │             │
│ user_id     │◄────┤ role_id     │◄────┤permission_id│
│ username    │     │ role_name   │     │ perm_name   │
│ email       │     │ description │     │ description │
│ created_at  │     │ created_at  │     │ resource    │
└─────────────┘     └─────────────┘     └─────────────┘
       │
       │ 1:N
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ USER_ORDERS │     │   ORDERS    │     │  PRODUCTS   │
│             │     │             │     │             │
│ user_id     │────►│ order_id    │◄────┤ product_id  │
│ order_id    │     │ order_date  │     │ product_name│
│ order_role  │     │ total_amount│     │ price       │
└─────────────┘     │ status      │     │ category_id │
                    └─────────────┘     └─────────────┘
                           │                   │
                           │ N:M               │ N:1
                           ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐
                    │ORDER_DETAILS│     │ CATEGORIES  │
                    │             │     │             │
                    │ order_id    │     │ category_id │
                    │ product_id  │     │ category_nm │
                    │ quantity    │     │ description │
                    │ unit_price  │     │ created_at  │
                    └─────────────┘     └─────────────┘
```

### 3.2 主要エンティティ定義

#### 3.2.1 ユーザー管理系

**USERS（ユーザー）**
| 項目名 | 論理名 | データ型 | 制約 | 説明 |
|---|---|---|---|---|
| user_id | ユーザーID | SERIAL | PK, NOT NULL | ユーザー識別子 |
| username | ユーザー名 | VARCHAR(50) | NOT NULL, UNIQUE | ログインID |
| email | メールアドレス | VARCHAR(255) | NOT NULL, UNIQUE | メールアドレス |
| password_hash | パスワードハッシュ | VARCHAR(255) | NOT NULL | ハッシュ化パスワード |
| first_name | 名 | VARCHAR(50) | NOT NULL | 名前（名） |
| last_name | 姓 | VARCHAR(50) | NOT NULL | 名前（姓） |
| department_id | 部署ID | INTEGER | FK | 所属部署 |
| status | ステータス | VARCHAR(20) | NOT NULL | アクティブ/非アクティブ |
| last_login | 最終ログイン | TIMESTAMP | NULL | 最終ログイン日時 |
| created_at | 作成日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | 更新日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |

**ROLES（ロール）**
| 項目名 | 論理名 | データ型 | 制約 | 説明 |
|---|---|---|---|---|
| role_id | ロールID | SERIAL | PK, NOT NULL | ロール識別子 |
| role_name | ロール名 | VARCHAR(50) | NOT NULL, UNIQUE | ロール名称 |
| description | 説明 | TEXT | NULL | ロール説明 |
| is_system | システムロール | BOOLEAN | NOT NULL, DEFAULT FALSE | システム定義フラグ |
| created_at | 作成日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | 更新日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |

**USER_ROLES（ユーザーロール関連）**
| 項目名 | 論理名 | データ型 | 制約 | 説明 |
|---|---|---|---|---|
| user_id | ユーザーID | INTEGER | PK, FK, NOT NULL | ユーザー識別子 |
| role_id | ロールID | INTEGER | PK, FK, NOT NULL | ロール識別子 |
| assigned_at | 付与日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | ロール付与日時 |
| assigned_by | 付与者 | INTEGER | FK, NOT NULL | 付与者ユーザーID |

#### 3.2.2 業務データ系

**ORDERS（注文）**
| 項目名 | 論理名 | データ型 | 制約 | 説明 |
|---|---|---|---|---|
| order_id | 注文ID | SERIAL | PK, NOT NULL | 注文識別子 |
| customer_id | 顧客ID | INTEGER | FK, NOT NULL | 顧客識別子 |
| order_date | 注文日 | DATE | NOT NULL | 注文日 |
| delivery_date | 配送予定日 | DATE | NULL | 配送予定日 |
| total_amount | 総金額 | DECIMAL(10,2) | NOT NULL | 総金額 |
| tax_amount | 消費税額 | DECIMAL(10,2) | NOT NULL | 消費税額 |
| status | ステータス | VARCHAR(20) | NOT NULL | 注文ステータス |
| created_at | 作成日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 作成日時 |
| updated_at | 更新日時 | TIMESTAMP | NOT NULL, DEFAULT NOW() | 更新日時 |

**ORDER_DETAILS（注文明細）**
| 項目名 | 論理名 | データ型 | 制約 | 説明 |
|---|---|---|---|---|
| detail_id | 明細ID | SERIAL | PK, NOT NULL | 明細識別子 |
| order_id | 注文ID | INTEGER | FK, NOT NULL | 注文識別子 |
| product_id | 商品ID | INTEGER | FK, NOT NULL | 商品識別子 |
| quantity | 数量 | INTEGER | NOT NULL, CHECK > 0 | 注文数量 |
| unit_price | 単価 | DECIMAL(8,2) | NOT NULL | 単価 |
| subtotal | 小計 | DECIMAL(10,2) | NOT NULL | 小計金額 |

### 3.3 制約定義

#### 3.3.1 主キー制約
- 全テーブルで主キー定義
- サロゲートキー（連番）を基本とする
- 複合主キーは関連テーブルでのみ使用

#### 3.3.2 外部キー制約
- 参照整合性を保証する外部キー定義
- CASCADE DELETE は慎重に適用
- インデックスを併用して性能を確保

#### 3.3.3 チェック制約
- データ範囲の妥当性チェック
- ステータス値の制限
- 数値の正負制限

#### 3.3.4 一意制約
- ビジネスキーの一意性保証
- 複合一意制約による業務ルール実装

## 4. 物理データモデル

### 4.1 テーブル設計詳細

#### 4.1.1 ユーザーテーブル（users）

```sql
CREATE TABLE users (
    user_id         SERIAL PRIMARY KEY,
    username        VARCHAR(50) NOT NULL UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    password_hash   VARCHAR(255) NOT NULL,
    first_name      VARCHAR(50) NOT NULL,
    last_name       VARCHAR(50) NOT NULL,
    department_id   INTEGER REFERENCES departments(department_id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active' 
                    CHECK (status IN ('active', 'inactive', 'locked')),
    last_login      TIMESTAMP,
    failed_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until    TIMESTAMP,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- インデックス
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_department ON users(department_id);
CREATE INDEX idx_users_status ON users(status);

-- 更新日時自動更新トリガー
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at BEFORE UPDATE
    ON users FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

#### 4.1.2 注文テーブル（orders）

```sql
CREATE TABLE orders (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INTEGER NOT NULL REFERENCES customers(customer_id),
    order_number    VARCHAR(20) NOT NULL UNIQUE,
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    delivery_date   DATE,
    total_amount    DECIMAL(12,2) NOT NULL CHECK (total_amount >= 0),
    tax_amount      DECIMAL(12,2) NOT NULL CHECK (tax_amount >= 0),
    discount_amount DECIMAL(12,2) NOT NULL DEFAULT 0 CHECK (discount_amount >= 0),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
    notes           TEXT,
    created_by      INTEGER NOT NULL REFERENCES users(user_id),
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- インデックス
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_number ON orders(order_number);

-- パーティショニング（年月別）
CREATE TABLE orders_y2024m01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
```

### 4.2 インデックス設計

#### 4.2.1 基本インデックス方針
- **主キー**: 自動的にインデックス作成
- **外部キー**: 参照性能向上のためインデックス作成
- **検索条件**: WHERE句で頻繁に使用される列
- **ソート条件**: ORDER BY句で使用される列

#### 4.2.2 複合インデックス設計

```sql
-- ユーザー検索用複合インデックス
CREATE INDEX idx_users_dept_status ON users(department_id, status);

-- 注文検索用複合インデックス
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);
CREATE INDEX idx_orders_status_date ON orders(status, order_date);

-- 注文明細集計用インデックス
CREATE INDEX idx_order_details_product_date ON order_details(product_id, created_at);
```

#### 4.2.3 部分インデックス

```sql
-- アクティブユーザーのみのインデックス
CREATE INDEX idx_users_active_username ON users(username) 
    WHERE status = 'active';

-- 未完了注文のみのインデックス
CREATE INDEX idx_orders_pending ON orders(order_date, customer_id)
    WHERE status IN ('pending', 'confirmed');
```

### 4.3 パーティショニング設計

#### 4.3.1 範囲パーティショニング

```sql
-- 注文テーブルの月別パーティショニング
CREATE TABLE orders (
    order_id        SERIAL,
    order_date      DATE NOT NULL,
    -- その他のカラム
    PRIMARY KEY (order_id, order_date)
) PARTITION BY RANGE (order_date);

-- 月別パーティション作成
CREATE TABLE orders_y2024m01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE orders_y2024m02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

#### 4.3.2 ハッシュパーティショニング

```sql
-- ログテーブルのハッシュパーティショニング
CREATE TABLE access_logs (
    log_id          SERIAL,
    user_id         INTEGER,
    -- その他のカラム
    PRIMARY KEY (log_id, user_id)
) PARTITION BY HASH (user_id);

-- ハッシュパーティション作成
CREATE TABLE access_logs_p0 PARTITION OF access_logs
    FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE access_logs_p1 PARTITION OF access_logs
    FOR VALUES WITH (modulus 4, remainder 1);
```

## 5. セキュリティ設計

### 5.1 アクセス制御

#### 5.1.1 データベースユーザー設計

```sql
-- アプリケーション用ユーザー
CREATE USER app_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE main_db TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;

-- 読み取り専用ユーザー（レポート用）
CREATE USER report_user WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE main_db TO report_user;
GRANT USAGE ON SCHEMA public TO report_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_user;

-- バックアップ用ユーザー
CREATE USER backup_user WITH PASSWORD 'secure_password';
GRANT pg_read_all_data TO backup_user;
```

#### 5.1.2 行レベルセキュリティ

```sql
-- 部署別データアクセス制御
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY dept_orders_policy ON orders
    FOR ALL TO app_user
    USING (customer_id IN (
        SELECT customer_id FROM customers 
        WHERE department_id = current_setting('app.current_department_id')::integer
    ));
```

### 5.2 データ暗号化

#### 5.2.1 カラムレベル暗号化

```sql
-- 機密データの暗号化
CREATE EXTENSION pgcrypto;

-- 暗号化関数
CREATE OR REPLACE FUNCTION encrypt_sensitive_data(data TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN encode(encrypt(data::bytea, 'encryption_key', 'aes'), 'base64');
END;
$$ LANGUAGE plpgsql;

-- 復号化関数
CREATE OR REPLACE FUNCTION decrypt_sensitive_data(encrypted_data TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN decrypt(decode(encrypted_data, 'base64'), 'encryption_key', 'aes')::TEXT;
END;
$$ LANGUAGE plpgsql;
```

### 5.3 監査設計

#### 5.3.1 監査ログテーブル

```sql
CREATE TABLE audit_logs (
    audit_id        SERIAL PRIMARY KEY,
    table_name      VARCHAR(50) NOT NULL,
    operation       VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    record_id       INTEGER NOT NULL,
    old_values      JSONB,
    new_values      JSONB,
    user_id         INTEGER,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- 監査トリガー関数
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, new_values, user_id)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, row_to_json(NEW), current_setting('app.current_user_id')::integer);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, old_values, new_values, user_id)
        VALUES (TG_TABLE_NAME, TG_OP, NEW.id, row_to_json(OLD), row_to_json(NEW), current_setting('app.current_user_id')::integer);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_logs (table_name, operation, record_id, old_values, user_id)
        VALUES (TG_TABLE_NAME, TG_OP, OLD.id, row_to_json(OLD), current_setting('app.current_user_id')::integer);
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

## 6. 性能設計

### 6.1 性能要件対応

#### 6.1.1 接続プール設定
- **最大接続数**: 100接続
- **初期接続数**: 10接続
- **接続タイムアウト**: 30秒
- **アイドルタイムアウト**: 300秒

#### 6.1.2 クエリ最適化
- **EXPLAIN ANALYZE**: 実行計画の確認
- **統計情報更新**: 定期的なANALYZE実行
- **インデックスヒント**: 必要に応じたヒント使用

### 6.2 バックアップ・リカバリ設計

#### 6.2.1 バックアップ戦略

```bash
# フルバックアップ（日次）
pg_dump -h localhost -U backup_user -d main_db -f backup_$(date +%Y%m%d).sql

# 差分バックアップ（WALファイル）
# postgresql.conf設定
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'
```

#### 6.2.2 レプリケーション設定

```sql
-- プライマリサーバー設定
-- postgresql.conf
wal_level = replica
max_wal_senders = 3
max_replication_slots = 3

-- レプリケーションユーザー作成
CREATE USER replicator REPLICATION LOGIN PASSWORD 'secure_password';

-- スタンバイサーバー設定
-- recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=primary_server port=5432 user=replicator'
```

## 7. 運用設計

### 7.1 監視項目

#### 7.1.1 データベース監視
- **接続数**: 最大接続数に対する使用率
- **CPU使用率**: データベースプロセスのCPU使用率  
- **メモリ使用率**: 共有バッファ、ワークメモリ使用率
- **ディスク使用率**: データ、WAL、一時ファイル領域
- **ロック状況**: 長時間ロック、デッドロック発生状況

#### 7.1.2 性能監視
- **スロークエリ**: 実行時間の長いクエリ
- **インデックス使用率**: インデックスの利用状況
- **テーブルサイズ**: テーブル・インデックスサイズ増加率
- **統計情報**: クエリ実行統計、キャッシュヒット率

### 7.2 メンテナンス

#### 7.2.1 定期メンテナンス

```sql
-- 統計情報更新（日次）
ANALYZE;

-- 不要データ削除（週次）
VACUUM;

-- 完全な領域回収（月次）
VACUUM FULL;

-- インデックス再構築（月次）
REINDEX DATABASE main_db;
```

#### 7.2.2 データ保持ポリシー

```sql
-- 古いログデータの削除（日次実行）
DELETE FROM access_logs WHERE created_at < CURRENT_DATE - INTERVAL '1 year';
DELETE FROM audit_logs WHERE created_at < CURRENT_DATE - INTERVAL '3 years';

-- パーティション削除
DROP TABLE IF EXISTS orders_y2022m01;
```

---

**注意**: このデータベース設計書は、システムのデータ基盤となる重要な文書です。変更時は性能・整合性への影響を十分に検討してください。