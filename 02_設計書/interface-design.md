# インターフェース設計書

**タイトル**: インターフェース設計書  
**バージョン**: 1.0.0  
**最終更新日**: YYYY-MM-DD  
**作成者**: [作成者名]  
**レビュアー**: [レビュアー名]  
**関連ドキュメント**: system-design.md, functional-requirements.md  
**ステータス**: draft  

---

## 1. 概要

### 1.1 目的
本文書は、システム内部およびシステム間のインターフェースを定義し、データ交換の仕様、プロトコル、エラーハンドリングを明確にすることを目的とする。

### 1.2 適用範囲
本インターフェース設計は、以下に適用される：
- フロントエンド - バックエンド間API
- 外部システム連携API
- バッチ処理インターフェース
- データベースアクセスインターフェース

### 1.3 設計原則
- **RESTful設計**: リソース指向のAPI設計
- **ステートレス**: セッション状態を持たない設計
- **版数管理**: API バージョニングによる後方互換性
- **エラーハンドリング**: 一貫したエラーレスポンス
- **セキュリティ**: 認証・認可・暗号化の実装

## 2. API全体設計

### 2.1 API設計概要

#### 2.1.1 基本URL構造
```
https://api.example.com/v1/{resource}
```

#### 2.1.2 HTTPメソッド使用方針
| メソッド | 用途 | 冪等性 | 安全性 |
|---|---|---|---|
| GET | リソース取得 | Yes | Yes |
| POST | リソース作成 | No | No |
| PUT | リソース更新（全体） | Yes | No |
| PATCH | リソース更新（部分） | No | No |
| DELETE | リソース削除 | Yes | No |

#### 2.1.3 ステータスコード使用方針
| コード | 用途 | 説明 |
|---|---|---|
| 200 | OK | 正常処理完了 |
| 201 | Created | リソース作成成功 |
| 204 | No Content | 削除成功（レスポンスボディなし） |
| 400 | Bad Request | リクエスト不正 |
| 401 | Unauthorized | 認証エラー |
| 403 | Forbidden | 認可エラー |
| 404 | Not Found | リソース未存在 |
| 409 | Conflict | リソース競合 |
| 422 | Unprocessable Entity | バリデーションエラー |
| 500 | Internal Server Error | サーバー内部エラー |

### 2.2 共通仕様

#### 2.2.1 リクエストヘッダー
```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer {JWT_TOKEN}
X-API-Version: v1
X-Request-ID: {UUID}
User-Agent: {CLIENT_INFO}
```

#### 2.2.2 レスポンスヘッダー
```http
Content-Type: application/json; charset=utf-8
X-Request-ID: {UUID}
X-Rate-Limit-Limit: 1000
X-Rate-Limit-Remaining: 999
X-Rate-Limit-Reset: 1609459200
Cache-Control: no-cache
```

#### 2.2.3 共通レスポンス形式

**成功レスポンス**
```json
{
  "status": "success",
  "data": {
    // 実際のデータ
  },
  "meta": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-01T00:00:00Z"
  }
}
```

**エラーレスポンス**
```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力値に誤りがあります",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      }
    ]
  },
  "meta": {
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-01T00:00:00Z"
  }
}
```

## 3. 認証・認可API

### 3.1 認証API

#### 3.1.1 ログイン
```http
POST /v1/auth/login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "securepassword",
  "remember_me": false
}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "dGhpcyBpcyBmYWtlIHJlZnJlc2ggdG9rZW4",
    "token_type": "Bearer",
    "expires_in": 3600,
    "user": {
      "id": 123,
      "username": "user@example.com",
      "first_name": "太郎",
      "last_name": "山田",
      "roles": ["user"]
    }
  }
}
```

#### 3.1.2 トークン更新
```http
POST /v1/auth/refresh
Content-Type: application/json

{
  "refresh_token": "dGhpcyBpcyBmYWtlIHJlZnJlc2ggdG9rZW4"
}
```

#### 3.1.3 ログアウト
```http
POST /v1/auth/logout
Authorization: Bearer {access_token}
```

### 3.2 ユーザー管理API

#### 3.2.1 ユーザー一覧取得
```http
GET /v1/users?page=1&per_page=20&sort=created_at&order=desc&department_id=1
Authorization: Bearer {access_token}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "users": [
      {
        "id": 123,
        "username": "user@example.com",
        "first_name": "太郎",
        "last_name": "山田",
        "department": {
          "id": 1,
          "name": "営業部"
        },
        "status": "active",
        "last_login": "2024-01-01T10:00:00Z",
        "created_at": "2023-06-01T09:00:00Z"
      }
    ],
    "pagination": {
      "current_page": 1,
      "per_page": 20,
      "total_pages": 5,
      "total_count": 100
    }
  }
}
```

#### 3.2.2 ユーザー詳細取得
```http
GET /v1/users/{user_id}
Authorization: Bearer {access_token}
```

#### 3.2.3 ユーザー作成
```http
POST /v1/users
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "username": "newuser@example.com",
  "email": "newuser@example.com",
  "first_name": "次郎",
  "last_name": "田中",
  "department_id": 2,
  "roles": ["user"]
}
```

#### 3.2.4 ユーザー更新
```http
PUT /v1/users/{user_id}
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "first_name": "次郎",
  "last_name": "田中",
  "department_id": 2
}
```

#### 3.2.5 ユーザー削除
```http
DELETE /v1/users/{user_id}
Authorization: Bearer {access_token}
```

## 4. 業務データAPI

### 4.1 注文管理API

#### 4.1.1 注文一覧取得
```http
GET /v1/orders?status=pending&from_date=2024-01-01&to_date=2024-01-31&customer_id=123
Authorization: Bearer {access_token}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "orders": [
      {
        "id": 1001,
        "order_number": "ORD-20240101-001",
        "customer": {
          "id": 123,
          "name": "株式会社サンプル"
        },
        "order_date": "2024-01-01",
        "delivery_date": "2024-01-05",
        "total_amount": 150000,
        "tax_amount": 15000,
        "status": "pending",
        "items_count": 3,
        "created_at": "2024-01-01T09:00:00Z"
      }
    ],
    "summary": {
      "total_orders": 25,
      "total_amount": 3750000,
      "pending_count": 10,
      "confirmed_count": 15
    }
  }
}
```

#### 4.1.2 注文詳細取得
```http
GET /v1/orders/{order_id}?include=items,customer,shipping
Authorization: Bearer {access_token}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "order": {
      "id": 1001,
      "order_number": "ORD-20240101-001",
      "customer": {
        "id": 123,
        "name": "株式会社サンプル",
        "contact_person": "営業 太郎",
        "phone": "03-1234-5678",
        "email": "contact@sample.co.jp"
      },
      "order_date": "2024-01-01",
      "delivery_date": "2024-01-05",
      "shipping_address": {
        "postal_code": "100-0001",
        "address": "東京都千代田区千代田1-1-1",
        "building": "サンプルビル3F"
      },
      "items": [
        {
          "id": 1,
          "product": {
            "id": 501,
            "name": "商品A",
            "code": "PRD-001"
          },
          "quantity": 10,
          "unit_price": 5000,
          "subtotal": 50000
        }
      ],
      "subtotal": 150000,
      "tax_amount": 15000,
      "total_amount": 165000,
      "status": "pending",
      "notes": "急ぎでお願いします",
      "created_by": {
        "id": 456,
        "name": "営業 太郎"
      },
      "created_at": "2024-01-01T09:00:00Z",
      "updated_at": "2024-01-01T09:00:00Z"
    }
  }
}
```

#### 4.1.3 注文作成
```http
POST /v1/orders
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "customer_id": 123,
  "order_date": "2024-01-01",
  "delivery_date": "2024-01-05",
  "shipping_address": {
    "postal_code": "100-0001",
    "address": "東京都千代田区千代田1-1-1",
    "building": "サンプルビル3F"
  },
  "items": [
    {
      "product_id": 501,
      "quantity": 10,
      "unit_price": 5000
    }
  ],
  "notes": "急ぎでお願いします"
}
```

#### 4.1.4 注文ステータス更新
```http
PATCH /v1/orders/{order_id}/status
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "status": "confirmed",
  "notes": "在庫確認完了"
}
```

### 4.2 商品管理API

#### 4.2.1 商品検索
```http
GET /v1/products/search?q=サンプル&category_id=1&min_price=1000&max_price=10000
Authorization: Bearer {access_token}
```

#### 4.2.2 商品一覧取得
```http
GET /v1/products?category_id=1&status=active&sort=name&order=asc
Authorization: Bearer {access_token}
```

#### 4.2.3 商品詳細取得
```http
GET /v1/products/{product_id}
Authorization: Bearer {access_token}
```

## 5. レポート・分析API

### 5.1 売上レポートAPI

#### 5.1.1 売上サマリー取得
```http
GET /v1/reports/sales/summary?period=monthly&year=2024&month=1
Authorization: Bearer {access_token}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "summary": {
      "period": "2024-01",
      "total_sales": 5000000,
      "total_orders": 120,
      "average_order_value": 41667,
      "growth_rate": 15.5,
      "top_products": [
        {
          "product_id": 501,
          "product_name": "商品A",
          "sales_amount": 800000,
          "sales_count": 25
        }
      ],
      "sales_by_day": [
        {
          "date": "2024-01-01",
          "sales_amount": 200000,
          "order_count": 5
        }
      ]
    }
  }
}
```

#### 5.1.2 詳細売上データ取得
```http
GET /v1/reports/sales/details?from_date=2024-01-01&to_date=2024-01-31&format=json
Authorization: Bearer {access_token}
```

#### 5.1.3 レポートエクスポート
```http
POST /v1/reports/sales/export
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "report_type": "sales_summary",
  "period": "monthly",
  "year": 2024,
  "month": 1,
  "format": "xlsx",
  "email_to": "manager@example.com"
}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "job_id": "export-123456",
    "status": "queued",
    "estimated_completion": "2024-01-01T10:05:00Z"
  }
}
```

## 6. 外部システム連携API

### 6.1 外部決済システム連携

#### 6.1.1 決済処理依頼
```http
POST /v1/external/payment/process
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "order_id": 1001,
  "payment_method": "credit_card",
  "amount": 165000,
  "currency": "JPY",
  "customer_info": {
    "name": "山田太郎",
    "email": "yamada@example.com"
  },
  "card_info": {
    "token": "card_token_123456"
  }
}
```

#### 6.1.2 決済ステータス確認
```http
GET /v1/external/payment/{payment_id}/status
Authorization: Bearer {access_token}
```

### 6.2 在庫システム連携

#### 6.2.1 在庫確認
```http
POST /v1/external/inventory/check
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "items": [
    {
      "product_id": 501,
      "requested_quantity": 10
    }
  ]
}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "items": [
      {
        "product_id": 501,
        "available_quantity": 25,
        "requested_quantity": 10,
        "reserved_quantity": 10,
        "status": "available"
      }
    ]
  }
}
```

#### 6.2.2 在庫引当
```http
POST /v1/external/inventory/reserve
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "order_id": 1001,
  "items": [
    {
      "product_id": 501,
      "quantity": 10
    }
  ],
  "reservation_expires_at": "2024-01-01T12:00:00Z"
}
```

## 7. バッチ処理API

### 7.1 バッチ制御API

#### 7.1.1 バッチジョブ実行
```http
POST /v1/batch/jobs
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "job_type": "daily_summary",
  "parameters": {
    "target_date": "2024-01-01",
    "notification_email": "admin@example.com"
  },
  "priority": "normal"
}
```

#### 7.1.2 バッチジョブステータス確認
```http
GET /v1/batch/jobs/{job_id}
Authorization: Bearer {access_token}
```

**レスポンス**
```json
{
  "status": "success",
  "data": {
    "job": {
      "id": "job-123456",
      "type": "daily_summary",
      "status": "running",
      "progress": 65,
      "started_at": "2024-01-01T02:00:00Z",
      "estimated_completion": "2024-01-01T02:30:00Z",
      "parameters": {
        "target_date": "2024-01-01"
      },
      "results": {
        "processed_records": 6500,
        "total_records": 10000,
        "errors": 0
      }
    }
  }
}
```

## 8. エラーハンドリング

### 8.1 エラーコード体系

| エラーコード | HTTPステータス | 説明 |
|---|---|---|
| INVALID_REQUEST | 400 | リクエスト形式不正 |
| VALIDATION_ERROR | 422 | バリデーションエラー |
| AUTHENTICATION_FAILED | 401 | 認証失敗 |
| ACCESS_DENIED | 403 | アクセス権限なし |
| RESOURCE_NOT_FOUND | 404 | リソース未存在 |
| CONFLICT | 409 | リソース競合 |
| RATE_LIMIT_EXCEEDED | 429 | レート制限超過 |
| INTERNAL_ERROR | 500 | サーバー内部エラー |
| SERVICE_UNAVAILABLE | 503 | サービス利用不可 |

### 8.2 エラーレスポンス詳細

#### 8.2.1 バリデーションエラー
```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力値に誤りがあります",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "有効なメールアドレスを入力してください"
      },
      {
        "field": "password",
        "code": "TOO_SHORT",
        "message": "パスワードは8文字以上で入力してください"
      }
    ]
  }
}
```

#### 8.2.2 業務エラー
```json
{
  "status": "error",
  "error": {
    "code": "INSUFFICIENT_INVENTORY",
    "message": "在庫が不足しています",
    "details": {
      "product_id": 501,
      "requested_quantity": 10,
      "available_quantity": 3
    }
  }
}
```

## 9. API仕様書生成

### 9.1 OpenAPI仕様

#### 9.1.1 基本情報
```yaml
openapi: 3.0.3
info:
  title: システム名 API
  description: システム名のREST API仕様書
  version: 1.0.0
  contact:
    name: 開発チーム
    email: dev-team@example.com
servers:
  - url: https://api.example.com/v1
    description: 本番環境
  - url: https://api-staging.example.com/v1
    description: ステージング環境
```

#### 9.1.2 セキュリティ定義
```yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
security:
  - BearerAuth: []
```

### 9.2 API文書化ツール
- **Swagger UI**: インタラクティブなAPI文書
- **ReDoc**: 美しいAPI文書生成
- **Postman Collection**: テスト用コレクション
- **SDK生成**: 各言語向けSDK自動生成

---

**注意**: このインターフェース設計書は、システム間連携の基準となる重要な文書です。変更時は影響を受ける全システムとの調整を行ってください。