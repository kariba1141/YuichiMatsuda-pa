# ETRCARS 外部設計書

# 1. システムアーキテクチャ図
本システムアーキテクチャ図は、ユーザーインターフェース層から AWS インフラストラクチャまでの全体フローを示す。

## 1.1. クライアント層
- Angular ベースの管理パネル用 Web アプリケーション
  - 本アプリケーションは、管理パネルとして機能する包括的な Web アプリケーションであり、システム管理者、店舗マネージャー、ディーラーマネージャーなど、管理者および運用担当者向けに設計される。
  - システム全体のコアデータ（ユーザーアカウント、商品カテゴリ、顧客情報、車両情報、注文取引履歴、レポート、分析情報など）を統合的に管理するためのインターフェースを提供する。
- iOS iPad アプリ（POS 業務向け）
  - このネイティブアプリは、POS（Point-of-Service）業務向けに設計され、顧客と直接対応する現場スタッフによって使用される。
  - バックエンド側では、この iPad アプリからのリクエストを検知できるため、専用の処理フローを実行できる。
  - 主な機能として、新規サービス注文の作成、車両情報の登録、顧客のデジタル署名の取得など、現場オペレーションに最適化された機能を提供する。

## 1.2. API・アプリケーション層
- Django ベースの REST API サーバ
  - バックエンドは Django と Django REST Framework（DRF）を用いたモノリシック構造のアプリケーションで、リソースベースの RESTful API を提供する。
  - モノリシックながら、認証、商品、注文などビジネスドメインごとに Django の「アプリ（apps）」として分割されており、保守性と責務分離が確保される。
- JWT 認証システム
  - API エンドポイントの保護には JSON Web Token（JWT）認証を使用する。
  - ログイン成功時に、短命のアクセストークンと長命のリフレッシュトークンを発行し、アクセストークンによる stateless かつ安全な認証を実現する。
- トークンライフサイクル管理
  - トークン発行：SimpleJWT を使用して認証成功時にトークンを生成する。
  - トークン有効期限：アクセストークンの有効期限は 7 日。さらに 7 日間のリフレッシュが可能。有効期限切れ後は再ログインが必要。
  - トークン更新と無効化：現時点ではトークン更新および無効化処理は未実装。
- 非同期タスク処理
  - Redis ベースのキューを利用し、長時間処理や高負荷タスクをバックグラウンドで非同期処理します。
  - 大容量の Excel レポート生成や一斉メール送信などの重い処理をワーカーへオフロードすることで、メインアプリケーションの応答性を維持し、ユーザー体験を向上させている。
  
## 1.3. データストレージ層
- PostgreSQL：構造化データベース
  - PostgreSQL はシステムの主要なリレーショナルデータベースとして使用され、ユーザー認証情報、顧客情報、商品情報、注文履歴など、すべての構造化データを保存する。
  - 堅牢性、信頼性および複雑なクエリとデータ整合性の強力なサポートにより PostgreSQL を採用する。
- 暗号化ポリシー
  - 保存時（Data at Rest）：データベースボリュームは AES-256 による暗号化を適用。
  - 通信時（Data in Transit）：SSL/TLS による暗号化通信と証明書検証を実装。
  - 機密情報：パスワードなどのユーザー認証情報は Django の PBKDF2（SHA-256）でハッシュ化。
  - 個人情報（PII）：フィールドレベルで AES-256-GCM を用いた暗号化を実施。
- バックアップポリシー
  - バックアップ頻度：RDS の自動スナップショット機能を用いて毎日フルバックアップを取得。
  - 保持期間：バックアップは 7 日間保持。
  - 災害復旧（DR）：WAL（Write-Ahead Log）アーカイブにより、5 分間隔のポイントインタイムリカバリが可能。
  - テスト：毎年、バックアップ復元テストを実施し、可用性を検証。
- AWS S3：ファイルストレージ（画像、ドキュメント、エクスポート）
  - S3 は非構造化データや大容量ファイルの保存に使用します。
  - 車両画像、顧客の署名データ、その他ユーザーアップロードファイル、さらに Excel 形式のデータエクスポートなどシステム生成ファイルも保存する。
  - これにより、アプリケーションサーバーからストレージを分離し、スケーラビリティとパフォーマンスが向上される。

## 1.4. 高レベルアーキテクチャ概要
- AWS を利用したアプリケーションホスティング
  - インフラはすべて AWS 上に構築され、アプリケーションは Docker コンテナとして統一的にデプロイ可能な形で構成される。
  - Amazon ECS（Elastic Container Service）を用いてアプリケーションコンテナを実行する。
- 負荷分散とマルチ AZ による高可用性
  - Application Load Balancer（ALB）が ECS 上で稼働する複数の Django インスタンスへ HTTP トラフィックを分散する。
  - ECS サービスおよび基盤インフラは複数のアベイラビリティゾーン（Multi-AZ）に跨ってデプロイされ、単一データセンター障害に対する耐障害性を確保する。
- VPC によるセキュリティ構成
  - Amazon VPC を用いて全体のネットワークセキュリティを強化する。
  - 特に PostgreSQL データベースは datasubnet と呼ばれるプライベートサブネットに隔離され、インターネットから直接アクセスできない。
  - Django アプリケーションも appsubnet と呼ばれる別のプライベートサブネット内で稼働する。
- スケーラビリティとパフォーマンス戦略
  - 主なスケール戦略は 水平スケーリングであり、ECS タスク数を需要に応じて動的に増減できます。
  - 現在は最小 3 タスク構成（2 vCPU / 4GB RAM）ですが、ユーザー数増加に応じて最小タスク数を増加させる想定。
  - データベースのボトルネックを回避するため、今後 RDS の **読み取り専用レプリカ（Read Replica）**を導入し、レポート処理や検索処理など読み取り負荷の高い処理を分散させる。
  - Django アプリケーションは読み込み系トラフィックをこれらレプリカへ振り分けるよう構成される。

## 1.5. 外部システム連携
### 1.5.1. ETR-BUY システム連携
本システムでは、外部の部品・商品情報管理システムである ETR との主要な連携を実装。
- コア連携機能
  - ETR システムからリアルタイムの部品および商品データを取得
  - 専用の JWT トークンを用いて ETR API へのリクエストを認証
  - 商品カタログ、仕様、在庫状況などの情報へアクセス
  - 自システムと ETR システム間で商品情報の整合性を維持

- 共通仕様
  - 通信方式：HTTPS のみ
  - コンテンツタイプ：application/json
  - 認証：
    - Authorization: Bearer <ETR_JWT_TOKEN>
    - ETR_JWT_TOKEN：ETR 保護リソース用の Bearer トークン（環境変数／Secrets Manager にて管理）
  - エンコーディング：UTF-8
  - タイムスタンプ：ログおよびエラーペイロードでは ISO-8601（UTC）を使用（内部仕様）
  - レート制限：429 エラー時は Retry-After の秒数に従って再試行
  - 冪等性：ここで記載する GET エンドポイントでは不要

### 1.5.2. エンドポイント仕様
#### 1.5.2.1. 商品一覧取得
- メソッド：GET
- URL：/api/product_lists/
- 目的：カテゴリおよびモデル情報に基づく商品／部品リストの取得
- リクエストヘッダー
  - Content-Type: application/json
  - Authorization: Bearer <ETR_JWT_TOKEN>
- クエリパラメータ：

| 項目 | タイプ | 説明 |
|----|----|----|
| category_id | string | カンマ区切りのカテゴリ ID（例：101,102,103） |
| model_number | string | 車両モデル識別子 |
| cta_shop_id | integer/string | CTA 由来のショップ識別子 |

- 成功レスポンス（例）

```
{
  "product_data": [
    {
      "id": 987,
      "name": "Oil 5W-30",
      "category": [{ "id": 101, "category_name": "OIL" }],
      "catalog_price": 5500,
      "code": "OIL-001"
    }
  ]
}
```

- エラー
  - 400：不正なパラメータ
  - 401/403：認証エラー
  - 429：レート制限超過
  - 5xx：上流システムエラー

#### 1.5.2.2. 商品コードチェック
- メソッド：GET
- URL：/api/check-product-code-cta/
- 目的：指定された CTA ショップに対して、商品コードが有効かどうかを検証する
- リクエストヘッダー
  - Content-Type: application/json
  - Authorization: Bearer <ETR_JWT_TOKEN>
- クエリパラメータ

| 項目 | タイプ | 説明 |
|----|----|----|
| product_id | string | 単一、またはカンマ区切りの複数の商品 ID／商品コード（ETR がサポートする形式） |
| cta_shop_id | integer/string | CTA 由来のショップ識別子 |

- 成功レスポンス（例）

```
[
    {
      "id": 987,
      "name": "Oil 5W-30",
      "category": [{ "id": 101, "category_name": "OIL" }],
      "catalog_price": 5500,
      "code": "OIL-001"
    },
    ...
 ]
```

#### 1.5.2.3. エラーハンドリング処理
- API通信の失敗は検知され、コンテキスト情報とともにログに記録される
- レスポンス検証により、処理前にデータ整合性が確保される
- タイムアウト制御により、ETRサービスの障害時にシステム遅延を防止する
- 特定のエラー種別は分類され、連携問題のトラブルシューティングを支援するためにログへ記録される

##### エラー：400（パラメータ不正）
- 400（不正なリクエスト）

```
{  
    "field_name": ["This field is required."]  
}
```

- 400（非フィールドエラー）

```
{
    "non_field_errors": [
        "The fields alive, number, hiragana_prefix, vehicle_class_number, regional_code, dealer must make a unique set."
    ]
}
```

- 400（カスタム検証エラー）

```
{
    "details": "Customer with that Kanji Name and Phone Number is already existed"
}
```

- 400 (重複エラー)

```
{
    "Maker": ["\"[HONDA] is already existed.\""]
}
```

##### 401/403（認証・権限エラー）
- 401 (未認証)

```
{
  "detail": "Incorrect Username or Password"
}
```

- 403 (禁止)

```
{
  "detail": "Insufficient permissions"
}
```

##### 404（未検出エラー）

```
{
  "detail": "Resource not found"
}
```

##### 429（レート制限）

```
No response
```

##### 5xx （上流/サーバーエラー）
- 500 (カスタムエラー)

```
{
    "error": "touches cannot be 0"
}
```

- 500 (Django デフォルトエラー)

```
Server Error (500)
```

## 1.6. システム構成図
システムはクライアント–サーバモデルを採用しており、フロントエンドクライアントとバックエンドサービス間は RESTful API によって通信される。
すべてのコンポーネントは、スケーラビリティと信頼性を確保するため AWS インフラ上にデプロイされる。

<img width="452" height="378" alt="image" src="https://github.com/user-attachments/assets/6baa23ea-3457-419b-8041-1266c24d3ebf" />

# 2. 非機能要件
## 2.1. 性能要件
- バックグラウンド処理：長時間処理タスクは Redis ベースのキューシステムにより非同期処理され、タイムアウトは 600 秒（DEFAULT_TIMEOUT: 600）に設定されています。
- ページネーション：API 応答はページネーションを実装しており、デフォルトのページサイズは 40 件。
- 本番環境ワーカー：Gunicorn は 3 ワーカー構成で稼働し、各ワーカーの最大リクエスト数は 1000 に設定。
- リクエストタイムアウト：本番サーバーのリクエストタイムアウトは、長時間処理に対応するため 6000 秒に設定。
- データベース性能：PostgreSQL ではコネクションプーリングを利用し、接続効率を最適化。
- API レスポンスタイム：負荷試験の結果に基づき、後日確定予定。
- 同時接続ユーザー数：インフラのスケーリング要件に基づき、後日決定される。

## 2.2. 可用性要件
- デプロイ方式：Docker を用いたコンテナベースのデプロイ方式を採用しています。
- ブルーグリーンデプロイ：本番環境には django および django-blue の 2 系統サービスを構成し、ゼロダウンタイムでのデプロイを実現。
- 負荷分散：AWS Load Balancer により、トラフィックを複数コンテナへ分散。
- データベース：PostgreSQL はバックアップおよびリカバリ機能を備え、高可用性を確保。
- SLA 目標値：ビジネス要件に基づき、後日定義される。
- インフラのスケーリング：AWS ECS のデプロイ設定により、自動スケーリング戦略を適用。

# 3. セキュリティ/認証仕様
## 3.1. セキュリティ要件
- 通信セキュリティ：すべての通信は TLS 1.2 以上を使用（ENABLED_HTTPS=True）。
- 認証方式：JWT ベースの認証方式を採用し、アクセストークンの有効期限は 7 日（7 日間のリフレッシュ可）に設定。
- データ暗号化：
  - 個人データは、django-searchable-encrypted-fields を使用し、AES-256-GCM により暗号化。
  - データベース接続は SSL/TLS による暗号化を使用。
  - パスワードは Django の PBKDF2（SHA-256）アルゴリズムによりハッシュ化される。
- CORS ポリシー：クロスオリジンリクエストに対応する CORS 設定を実装。
- セッションセキュリティ：HTTPS 環境において Secure Cookie を有効化。

## 3.2. 認証（Authentication）
JWT（JSON Web Token）が主要な認証方式であり、フォールバックとしてセッション認証およびベーシック認証が利用可能。ユーザーが認証に成功すると、以降のすべてのリクエストライフサイクルにおいて、呼び出し元はユーザープロファイルを通じて識別される。
- 主要方式：Authorization ヘッダに送信する JWT Bearer トークン
- トークン有効期限：アクセストークン 7 日、さらに 7 日間のリフレッシュが可能
- デフォルトポリシー：すべての API エンドポイントは認証済みユーザーを要求。匿名アクセスはデフォルトで不可。
- アイデンティティ解決：認証後、ユーザープロファイル（割り当てられたロール、ディーラー、ショップ、ブランチ、工場を含む）がすべての後続の認可判断の基盤となる。

## 3.3. 認可とアクセス制御（Authorization & Access Control）
認可はすべての API リクエストに対して順番に適用される4層構成で実施される。ビジネスロジックが実行される前に、リクエストはすべての適用可能な層を通過する必要がある。

**認証 → RBAC → メニュー/権限 → スコープ（BOLA）**

### 3.3.1. 層1：認証（あなたは誰か？）
リクエストが有効な JWT を持ち、呼び出しユーザーを識別することを確認する。有効な認証情報がないリクエストは HTTP 401 で拒否される。セクション 3.2 参照。

### 3.3.2. 層2：RBAC（あなたのロールはこのエンドポイントに許可されているか？）
各エンドポイントはアクセスを許可されるロールを宣言する。システムは2つの補完的なスタイルを使用する。
- 許可リスト（Allowlist、推奨）：明示的にリストされたロールのユーザーのみが許可され、他のロールは HTTP 403 を受け取る。一般的なロールグループには、タブレットユーザーのみ、タブレット・ショップマネージャー・ディーラーマネージャーの組み合わせ、およびそれらに管理者を加えたグループが含まれる。
- 拒否リスト（Blocklist、レガシー）：リストされたロールはアクセスが拒否され、他のすべてのロールは許可される。古いエンドポイントとの後方互換性のために維持。

### 3.3.3. 層3：メニュー/個別権限アクセス（この細粒度の権限を持っているか？）
ロールメンバーシップを超えて、システムは文字列キーで識別される細粒度の権限をサポートする（例：注文管理や販促内の特定メニュー項目、画面、またはオペレーションをカバーする権限）。
- ロール権限：特定のディーラー（ブランチ管理者の場合はブランチ）のスコープ内でロールに付与される。同じロールでも異なるディーラーの2ユーザーは異なる有効権限を持つ可能性がある。
- ユーザー権限：ロールのデフォルトの上に重ねられた、個別ユーザーの明示的な許可または拒否のオーバーライド。
- 有効権限：（ロール権限 ∪ ユーザー許可オーバーライド）− ユーザー拒否オーバーライド として解決される。
- 適用対象：ブランチ管理者、ディーラーマネージャー、ショップマネージャー、ショップ。他のロールはこの層をバイパスする。

### 3.3.4. 層4：スコープ/BOLA（あなたはその特定のオブジェクトを所有しているか？）
最終層は、Broken Object-Level Authorization（BOLA）を防ぐ。ロールと権限チェックを通過しても、ユーザーはアクセスする特定のレコードを所有していなければならない。スコープはユーザープロファイルのディーラーメンバーシップおよび割り当てられたショップに対して評価される。

ロール別のデフォルトスコープルール：

| ロール | ディーラースコープ | ショップスコープ |
|--------|-----------|-----------|
| Tablet User | あり | あり |
| Shop Manager | あり | あり |
| Dealer Manager | あり | なし |
| Dealer Admin | あり | なし |
| Admin | バイパス | バイパス |
| その他のロール | なし | なし |

スコープはリクエストライフサイクルの5つのポイントで適用される：

| ポイント | 保護内容 |
|---------|---------|
| 一覧クエリセット | 一覧レスポンスをフィルタリングし、ユーザーがディーラー/ショップスコープ内のレコードのみ表示できるようにする |
| クエリパラメータ | ビューロジック実行前に ?shop= などのフィルタパラメータを検証する |
| 詳細オブジェクトアクセス | 単一レコードの GET、PATCH、DELETE に対するオブジェクトレベルのガード |
| リクエストペイロード（POST/PATCH） | リクエストボディ内のディーラー、ショップ、スタッフの外部キーがスコープ内であることを検証する |
| サービス/バックグラウンドタスク | サービス、非同期ジョブ、シグナルハンドラーが使用する中央集権的なスコープ付きクエリヘルパー |

## 3.4. ロール（Roles）
システムは以下のロールを定義する。各ユーザープロファイルはちょうど1つのロールを割り当てられ、ディーラー、1つ以上のショップ、および（該当する場合は）ブランチと工場にリンクされる。

| ロール | 説明 |
|--------|------|
| Tablet User | サービスショップで iPad アプリを使用する現場オペレーター |
| Shop Manager | 特定のショップ（またはディーラー内のショップ群）を管理する |
| Dealer Manager | ディーラー内のすべてのショップを管理する |
| Branch Manager | ブランチ（ショップレベルより上のグループ）を管理する |
| Admin | システム管理者。スコープチェックをバイパスし、ディーラー横断の全アクセスを持つ |
| Factory User | 工場ドメイン内で操作する |
| Shop | ショップレベルのアカウント |
| Branch Admin | ブランチにスコープされた管理者ロール |
| Dealer Admin | ディーラーにスコープされた管理者ロール |

# 4. 監視/トラッキング仕様
## 4.1. 運用・保守要件
- ログ管理：アプリケーションログは Django のロギングフレームワークで管理され、Sentry と連携して収集・分析を行う。
- 監視：Sentry を用いてエラー追跡およびパフォーマンス監視を実施。
- バックグラウンドジョブ：Django-RQ と Redis を使用し、非同期タスク処理を実現。
- データベースバックアップ：自動バックアップ戦略を実施し、詳細は運用セクションで定義される。

# 5. API 外部仕様（REST/JSON）
## 5.1. API 一覧（主要エンドポイント）
### 5.1.1. 認証
- POST /api/token/ - JWT token authentication (ios login)
- POST /api/api-token-auth/ - JWT token authentication (web)
- POST /api/logout/

### 5.1.2. 顧客管理
- GET|POST /api/customer_info/
- GET /api/customer_info/customer-info-web/
- GET|PATCH|DELETE /api/customer_info/{id}/
- GET /api/customer_search/
- GET /api/export-customer/

### 5.1.3. ナンバープレート管理
- GET|POST /api/license_plates/
- PATCH|DELETE /api/license_plates/{id}/
- POST /api/license_plates/create_only/
- POST /api/license_plates/create_only_new/
- GET /api/license_plates/{id}/last_purchase_history/

### 5.1.4. 車両管理
- GET|POST /api/car_data/
- PATCH|DELETE /api/car_data/{id}/
- GET /api/car_data/{id}/announced_recalls/
- GET|PUT /api/car_data/{id}/update_check_up_status/
- GET /api/car_search/
- GET|POST /api/car_brand/
- PATCH|DELETE /api/car_brand/{id}

### 5.1.5. 注文管理
- GET|POST /api/orders/
- PATCH|DELETE /api/orders/{id}
- GET /api/orders/count?shop=<id>&status=<int>
- DELETE /api/orders/batch_delete/
- GET /api/orders/tablet-search/
- GET /api/orders/purchase-management/
- GET /api/orders/last-inspection-order/?license_plate=<id>
- GET|POST /api/order_items/
- PATCH|DELETE /api/order_items/{id}
- GET /api/search_orders/
- GET /api/export_order/

### 5.1.6. ショップ予約
- GET|POST /api/shop-bookings/
- PATCH|PUT|DELETE /api/shop-bookings/{id}
- GET /api/shop-bookings/month-view/

### 5.1.7. 工場予約
- GET|POST /api/factory-bookings/
- PATCH|PUT|DELETE /api/factory-bookings/{id}
- PATCH /api/factory-bookings/{id}/patch-mechanic/
- GET /api/factory-bookings/day-view/
- GET /api/factory-bookings/summary/
- GET /api/export_factory_order/
- GET|POST /api/pits/
- PATCH|PUT|POST /api/pits/{id}/

### 5.1.8. 分析・レポート
- GET /api/sales-achievement/first-layer/
- GET /api/sales-achievement/second-layer/
- GET /api/export-sales-achievement/
- GET /api/daily-sales-achievement/first-layer/
- GET /api/daily-sales-achievement/second-layer/
- GET /api/daily-sales-achievement/third-layer/
- GET /api/sales-performance/second-layer/
- GET /api/export-sales-performance/
- GET /api/export_daily_monthly_sales_performances/
- GET /api/repairs/
- GET /api/export_repairs/
- GET /api/registered-car-statistics/
- GET /api/registered-car-statistics/second-layer
- GET /api/registered-car-statistics/daily/
- GET /api/registered-car-statistics/monthly/
- GET /api/export-registered-car-statistics/
- GET /api/product-purchase/first-table/
- GET /api/product-purchase/second-table/
- GET /api/export-product-purchase/

### 5.1.9. 声掛けリスト(リマインダー) 
- GET|POST /api/reminders/
- PATCH|PUT|DELETE /api/reminders/{id}/
- PATCH /api/reminders/bulk-update-status/
- GET /api/license-plate-reminders/
- GET|POST /api/reminder-memos/
- PATCH|PUT|DELETE /api/reminder-memos/{id}/

### 5.1.10. アンケートメニュー
- GET|POST /api/question-menus/
- PATCH|DELETE /api/question-menus/{id}/
- GET /api/question-menus/get_question_history/
- GET|POST /api/questions/
- PATCH|DELETE /api/questions/{id}/
- GET /api/get_question_names/

### 5.1.11. アンケート履歴
- GET /api/question-history/
- GET /api/question-history/export/

### 5.1.12. ユーザー管理
- GET|POST /api/user-profiles/
- PATCH|PUT|DELETE /api/user-profiles/{id}/
- POST /api/user-profiles/{id}/reset-password/
- GET|POST /api/tablet-user-profiles/
- PATCH|PUT|DELETE /api/tablet-user-profiles/{id}
- POST /api/send-initial-setup-emails/
- GET|POST /api/password-setup/
- GET /api/get-all-user-full-name/

### 5.1.13. ディーラー管理
- GET|POST /api/dealers/
- PATCH|PUT|DELETE /api/dealers/{id}/
- GET /api/get-all-dealers/
- GET /api/get-all-dealers-etr/
- POST /api/import-dealers/

### 5.1.14. ショップ管理
- GET|POST /api/shops/
- PATCH|PUT|DELETE /api/shops/{id}/
- GET /api/get-all-dealers/
- GET /api/get-all-dealers-etr/
- POST /api/import-dealers/

### 5.1.15. ログ管理
- GET /api/logs/

### 5.1.16. インポート・エクスポート
- GET /api/import_results/
- GET /api/export_results/

### 5.1.17. 権限管理
- GET /api/permissions/
- GET /api/role-permissions/
- POST /api/role-permissions/
- PATCH /api/role-permissions/{id}/
- DELETE /api/role-permissions/{id}/
- GET /api/user-profiles/{id}/permissions/
- PATCH /api/user-profiles/{id}/update-permissions/
- GET /api/user-profiles/self-permissions/

### 5.1.18. ブランチアカウント管理
- GET|POST /api/branch-account-user-profiles/
- PATCH|PUT|DELETE /api/branch-account-user-profiles/{id}/
- POST /api/branch-account-user-profiles/{id}/reset-password/

### 5.1.19. 車検オーダー
- GET /api/car_checkup_orders/
- GET /api/car_checkup_orders/export/

## 5.2. リクエスト／レスポンス例
### 5.2.1. 認証
#### 5.2.1.1. トークン認証（iOS）
- エンドポイント：/api/token/
- メソッド：POST
- 認証：なし（ログイン用エンドポイント）
- ヘッダー：Content-Type: application/json
##### リクエストボディ（JSON）

```
{
  "username": "string",
  "password": "string"
}
```

##### 成功レスポンス（Success Response 200, JSON）

```
{
    "prefix": "string",
    "token": "string",
    "expiration": number,
"role": number,
"customer_terms": "string",
"etr_token": "string"
}
```

##### エラーレスポンス (problem+json)
- 401 Unauthorized (無効な認証情報)

```
{"detail": "Incorrect Username or Password"}
```

##### セキュリティ & ポリシー
- JWT Bearer アクセストークンの有効期限は 7 日。期限切れ前に同一 API コールでリフレッシュが可能（さらに 7 日延長）。
- 本バージョンではサーバー側でトークン無効化はできず、実質的な無効化は有効期限または署名キーのローテーションにより実施。
- 保護されたすべてのエンドポイントでは、Authorization: Bearer <access_token>
を使用すること。
- 備考

レスポンスには、クライアント側の RBAC（ロールベースアクセス制御）に使用するユーザーロールとスコープが含まれる。

#### 5.2.1.2. トークン認証（Web）
- エンドポイント：/api/api-token-auth/
- メソッド：POST
- 認証：なし（ログイン用エンドポイント）
- ヘッダー：Content-Type: application/json
##### リクエストボディ（JSON）

```
{
"username": "string",
"password": "string"
}
```

##### 成功レスポンス（Success Response 200, JSON）

```
{
    "token": "string",
    "userprofile": {
        "id": number,
        "role_display": "string",
        "branch": null,
        "dealer": {
            "id": number,
            "branch_name": "string",
            "shops": number,
            "sequence": number,
            "name": "string",
            "tax_rate": decimal,
            "logo": "url_string",
            "is_production": boolean,
            "is_approval_function": boolean,
            "dealer_type": number,
            "is_default_sort": boolean,
            "branch": number,
            "default_region": number
        },
        "username": "string",
        "password_notification":boolean,
        "given_name": "string",
        "family_name": "string",
        "phone_number_1": "string",
        "phone_number_2": "string",
        "email": "email_string",
        "profile_picture": null,
        "customer_code": "string",
        "image": null,
        "role": number,
        "attempt": number,
        "lock_time": null,
        "ios_version": "string",
        "email_setup_status": number,
        "filter_config": null,
        "user": number,
        "shops": [],
        "factories": []
    }
}
```

##### エラーレスポンス (problem+json):
- 400 Bad Request (バリデーション)

```
{"field_name": ["This field is required."]}
```

- 401 Unauthorized (無効な認証情報)

```
{"detail": "incorrectUsernameOrPassword"}
```

##### セキュリティ & ポリシー
- JWT Bearer アクセストークンの有効期限は 7 日。期限切れ前にリフレッシュが可能（さらに 7 日延長）。
- 本バージョンではサーバー側でトークン無効化はできず、実質的な無効化は有効期限または署名キーのローテーションにより実施。
- 保護されたすべてのエンドポイントでは、Authorization: Bearer <access_token> を使用すること。
- 備考

レスポンスには、クライアント側の RBAC（ロールベースアクセス制御）に使用するユーザーロールとスコープが含まれる。

#### 5.2.1.3. ログアウト
- エンドポイント：/api/logout/
- メソッド：POST
- 認証：任意（ステートレス）
- ヘッダー：
  - Authorization: Bearer <access_token>
  - Content-Type: application/json
- リクエストボディ
  - Request Body: { }  // empty
##### 成功レスポンス (JSON)

```
{'detail': 'Logout'}
```

##### エラーレスポンス
- 公開／ステートレス動作では特に想定されない。

ただし、保護されている場合は、トークンが無効または欠落している際に 401 が返される可能性がある。

- 動作
  - ステートレスなログアウト：サーバーは JWT を保存したり、無効化したりしない。
  - クライアント側の必須動作：クライアントは保存されている JWT をローカルで 必ず削除しなければならない（MUST delete）。
  - トークンの有効期間：トークンは 有効期限（7 日） に達するか、署名キーがローテーションされる まで有効のまま。

### 5.2.2. 顧客管理 API
#### 5.2.2.1. 顧客関連オペレーション
- ベースリソース：/api/customer_info/
- メソッド：GET（list）、POST（create）、GET /{id}/（retrieve）、PATCH /{id}/（partial update）、DELETE /{id}/（delete）
- 認証：必須（JWT Bearer）
- 権限／スコープ
  - RBAC が適用され、ユーザーは適切な dealer／shop スコープに属していなければならない。
  - SHOP_MANAGER の場合、一覧結果は「そのマネージャーのショップに注文履歴を持つ顧客」に制限される。

##### 顧客一覧
- エンドポイント：/api/customer_info/customer-info-web
- メソッド：GET
- クエリパラメータ（Query Params — フィルターの一部）：
  - dealer: number（ディーラー ID）
  - name, name_kanji: string（漢字名 contains）
  - name_perfect_match: string（漢字フルネーム完全一致、スペース無視）
  - kana_name, kana_name_given: string（カナ名 contains）
  - phones または phone: string（正規化済完全一致）
  - member_number: string
  - shop, shops: string（ショップ名のカンマ区切り）または ids（カンマ区切り）
  - number, hiragana_prefix, vehicle_class_number, regional_code: string（ナンバープレート項目）
  - brand, model: string（車データ）
  - created_from, created_to: YYYY-MM-DD
  - updated_from, updated_to: ISO-8601 datetime
  - latest_order_date: YYYY-MM-DDtoYYYY-MM-DD
  - no_order: "true"（注文を持たない顧客のみ返す）

##### 成功レスポンス（Success Response 200）：顧客のページネーションされたリスト

```
{
    "count": number,
    "next": "url_string",
    "previous": "url_string",
    "results": [
        {
            "id": number,
            "family_name": "string",
            "given_name": "string",
            "kana_name": "string",
            "kana_name_given": "string",
            "line_qr_code": "url_string",
            "address_line_1": "string",
            "address_line_2": "string",
            "house_number": "string",
            "house_phone_number": "string",
            "postal_code": "string",
            "prefecture": "string",
            "city": "string",
            "town": "string",
            "building": "string",
            "mobile_phone_number": "string",
            "normalized_house_phone_number": "string",
            "normalized_mobile_phone_number": "string",
            "shop": [],
            "license_plate_data": [
                "string"
            ],
            "maker": [
                "string"
            ],
            "model": [
                "string"
            ],
            "created_at": "datetime_string",
            "updated_at": "datetime_string",
            "order_number": "string",
            "latest_order_date": "string",
            "is_term_accepted": boolean
        }
    ]
}
```

- 備考
  - 結果は重複排除される。
  - パフォーマンス向上のため、ナンバープレートおよび注文情報は事前読み込みされる。

##### 顧客取得
- エンドポイント: /api/customer_info/{id}/
- メソッド: GET

##### 成功レスポンス（Success Response 200）：Customer オブジェクト

```
{
	"id": number,
	"family_name": "string",
	"given_name": "string",
	"kana_name": "string",
	"kana_name_given": "string",
	"line_qr_code": "url_string",
	"address_line_1": "string",
	"address_line_2": "string",
	"house_number": "string",
	"house_phone_number": "string",
	"postal_code": "string",
	"prefecture": "string",
	"city": "string",
	"town": "string",
	"building": "string",
	"mobile_phone_number": "string",
	"normalized_house_phone_number": "string",
	"normalized_mobile_phone_number": "string",
	"shop": [],
	"license_plate_data": [
		"string"
	],
	"maker": [
		"string"
	],
	"model": [
		"string"
	],
	"created_at": "datetime_string",
	"updated_at": "datetime_string",
	"order_number": "string",
	"latest_order_date": "string",
	"is_term_accepted": boolean
}
```

##### 顧客の作成
- エンドポイント: /api/customer_info/
- メソッド: POST
- ヘッダー: Content-Type: application/json
- リクエストボディ（JSON）：フィールド定義は「Database Entity Definitions（Excel）」に従う。主要なフィールドは以下のとおり：
  - dealer: number（必須）
  - family_name: string（必須）
  - given_name: string
  - kana_name, kana_name_given: string
  - postal_code または map_code: string（相互排他。どちらか一方のみ使用可能）
  - house_number, address_line_1, address_line_2, building, prefecture, city, town
  - house_phone_number, mobile_phone_number, email
  - license_plate_id: integer（任意。新規顧客にナンバープレートを紐付ける）
  - line_qr_code, line_friend_suffix（任意）
- バリデーションルール（サーバー側が authoritative）：
  - 重複保護：同一ディーラー内に同じ漢字氏名（family_name + given_name）、かつ同じ電話番号を持つ顧客が存在する場合、リクエストは拒否される。
  - 住所正規化：postal_code と map_code は相互排他であり、サーバーはどちらか一方を保持し、もう一方を空にする。
  - ナンバープレートの再紐付け（license_plate_id provided の場合）
    - 指定された license_plate_id は新規顧客に紐付けられる。
    - 過去に「プレートのみ」に紐付いていた既存注文は、この顧客に再紐付けされる。
  - 成功レスポンス（Success Response 201）：作成された Customer オブジェクト
##### エラーレスポンス（400, 例）：

```
{
    "details": "Customer with that Kanji Name and Phone Number is already existed"
}
```

##### 顧客の更新（部分更新）
- エンドポイント: /api/customer_info/{id}/
- メソッド: PATCH
- ヘッダー: Content-Type: application/json
- リクエストボディ（JSON）：作成可能フィールドの任意のサブセットを許可。サポートされる追加項目：
  - license_plate_id：追加のナンバープレートを紐付け、以前「未紐付け」であった注文を再紐付けする。
  - line_qr_code, line_friend_suffix
- バリデーションルール：
  - 重複保護：漢字氏名または電話番号フィールドを変更する場合、作成時と同一の重複チェックが適用される。
  - 住所正規化：postal_code と map_code の相互排他ルールは更新時にも適用される。
- 成功レスポンス（Success Response 200）：更新された customer オブジェクト。
- エラーレスポンス（400）：重複／バリデーションエラーに準じる。

##### 顧客削除
- エンドポイント: /api/customer_info/{id}/
- メソッド: DELETE
- 成功レスポンス：204 No Content（許可されている場合）
- 備考：標準の DRF ModelViewSet の delete 動作が適用される。

#### 5.2.2.2. 顧客情報検索
- エンドポイント：/api/customer_search/
- メソッド：GET
- 認証：必須（JWT Bearer）
- 目的：名前、電話番号、ナンバープレート、車両ブランド／モデル、店舗、日付範囲によって顧客を検索する。
- スコープ／権限：結果は、認証されたユーザープロファイルから導出された呼び出し元（caller）のディーラーにスコープされる。
- クエリパラメータ（Query Params — フィルターの一部）：
  - dealer: number（ディーラー ID）
  - name, name_kanji: string（漢字名 contains）
  - name_perfect_match: string（漢字氏名の完全一致、スペース無視）
  - kana_name, kana_name_given: string（カナ名 contains）
  - phones または phone: string（正規化済完全一致）
  - member_number: string
  - shop, shops: string（ショップ名のカンマ区切り）または ID（カンマ区切り）
  - number, hiragana_prefix, vehicle_class_number, regional_code: string（ナンバープレート項目）
  - brand, model: string（車両データ）
  - created_from, created_to: YYYY-MM-DD
  - updated_from, updated_to: ISO-8601 datetime
  - latest_order_date: YYYY-MM-DDtoYYYY-MM-DD
  - no_order: 'true'（注文を持たない顧客のみ返す）
##### 成功レスポンス（Success Response 200）：
- 構造：各アイテムは次の 3 要素の複合オブジェクト
  - 顧客
  - 1 つのナンバープレート
  - そのプレートに紐づく車両

```
{
	"count": <int>, //total plates matched (note: plate-based)
	"next": "url-or-null",
	"previous": "url-or-null",
	"results": [
		{
			"car": {
				/* Car fields; may be null if absent */
			},
			"customer": {
/* Customer objects */
			},
			"license_plate": {
				/* License Plate fields */
			},
			"qr_data": "string-or-null"
		}
	]
}
```

- 備考：
  - Active な LicensePlateCustomerInformation を通じて紐付けられているレコードのみ返される。
  - パフォーマンス向上のため、Car／License Plate データは事前読み込みされる。

#### 5.2.2.3. ナンバープレート管理
- ベースリソース：/api/license_plates/
- メソッド：
	- GET（list）
	- POST（create）
	- GET /{id}/（retrieve）
	- PATCH /{id}/（partial update）
	- DELETE /{id}/（delete）
- 認証：Required（JWT Bearer）
- 目的：顧客リンク、QR データ、および関連車両（car_data）を含むナンバープレートを管理する。

##### スコープ／権限
- 結果はロールによってスコープされる：
	- DEALER_MANAGER / SHOP_MANAGER（割り当てられたショップがある場合）：自身のショップに属するナンバープレートのみ。
	- その他の非管理者ユーザー：自身のディーラーに属するナンバープレート。
	- Admin：明示的な dealer フィルタリングを使用できる。
	
- ナンバープレート一覧・フィルター
	- エンドポイント： /api/license_plates/
	- メソッド： GET
	- クエリパラメータ：
		- id, number, regional_code, hiragana_prefix, vehicle_class_number
		- shop, dealer
		- brand（car_data.brand）, model（car_data.model）
		- customer_kanji_name：スペース区切りのトークンが family_name または given_name にマッチ
		- customer_kana_name：カナ名の 完全一致
		- expired_date_range：YYYY-MM-DD,YYYY-MM-DD（QR の expired_date ウィンドウ；undefined の値は無視される）
		- created_date_range：YYYY-MM-DD,YYYY-MM-DD（包括的範囲。JST 09:00 を境界とする）
		- shop_sub_cars=true + dealer=<id>：ディーラーの sub-cars を一覧表示（ユーザースコープをバイパス）
	- 成功レスポンス（Success Response 200）：LicensePlate の配列で、以下の情報を含む：
		- customer_info：紐付けられた顧客の配列
		- car_data：車両オブジェクト（存在する場合）
		- qr_code：最新の QR レコード（存在する場合）
		- qr_code_snapshot：読み取り専用の QR スナップショットフィールド
		
- ナンバープレート作成
	- エンドポイント：/api/license_plates/
	- メソッド： POST
	- ヘッダー： Content-Type: application/json
	- リクエストボディ（JSON）：エンティティ定義に従う plate フィールド。
		- dealer: ID（required）
		- shop: ID（多くのフローで必須）
		- regional_code: string
		- vehicle_class_number: string
		- hiragana_prefix: string
		- number: string（ハイフンは任意。バックエンドで正規化される）
	- 挙動：
		- モデルフィールドに対する get_or_create により 冪等性が保持される（モデル制約により重複は防止される）。
		- 作成された、または既存の plate を返す。
		- ここでは car_data は作成されない。
	- 成功レスポンス（201）：計算済フィールド（customer_info, car_data, qr_code を含む）を持つ LicensePlate。

- ナンバープレート更新
	- エンドポイント：/api/license_plates/{id}/
	- メソッド： PATCH
	- ヘッダー：Content-Type: application/json
	- リクエストボディ（JSON）：plate フィールドの任意のサブセット。
	- 成功レスポンス（Success Response 200）：更新された LicensePlate。

- ナンバープレート削除
	- エンドポイント：/api/license_plates/{id}/
	- メソッド： DELETE
	- 成功レスポンス：204 No Content（標準 DRF の動作）

- 追加操作
##### ナンバープレートのみ作成・車データなし
- エンドポイント：/api/license_plates/create_only/
- メソッド： POST
- リクエストボディ（JSON）：Create と同じ plate フィールド
- 成功レスポンス（Success Response 201）：LicensePlate を作成

##### ナンバープレート + 車データ作成
- エンドポイント：/api/license_plates/create_only_new/
- メソッド： POST
- リクエストボディ（JSON）：

```
{
	...plate fields...,
	"car_data": [
		{
			"brand": "string",
			"model": "string",
			"color": "string",
			"registered_date": "YYYY-MM",
			"expired_date": "YYYY-MM-DD",
			"mileage": number,
			"front_photo": "data:image/png;base64,...."//optional;base64
		}
	]
}
```

- レスポンス（Response）：201（LicensePlate と組み込みの car_data を含む）

##### 最終購入履歴
- エンドポイント：/api/license_plates/{id}/last_purchase_history/
- メソッド：GET
- 成功レスポンス（Success Response 200）：当該車両の 直近の購入／サービス履歴の集計データ。

#### 5.2.2.4. 顧客データインポート/エクスポート
- 認証：必須（JWT Bearer）

##### 顧客エクスポート（非同期）
- エンドポイント：/api/export-customer/
- メソッド：GET
- 目的：顧客データをダウンロード可能なファイルとしてエクスポートする（非同期ジョブ）。
- クエリパラメーター（任意のサブセット）：
	- dealer: ID
	- shops: IDs（配列）
	- family_name, given_name, kana_name, kana_name_given: string
	- regional_code, hiragana_prefix, vehicle_class_number, number: string
	- no_order: 'true'（注文のない顧客を含める）
	- created_from, created_to: YYYY-MM-DD
	- updated_from, updated_to: YYYY-MM-DD
	- latest_order_date: "YYYY-MM-DDtoYYYY-MM-DD"
	- shop: string（ショップ名）
	- brand, model: string
	- name: string（フリーテキスト；バックエンドのルールに従って処理）
- 挙動：
	- パラメータを検証し、バックグラウンドエクスポートをキューへ投入し、ExportResult レコードを作成する。
	- 即座に 200 を返し、レスポンスは "Exported Successfully"。
	- /api/export_results/ をポーリングしてステータスを追跡し、ファイルが準備できたら取得する。
- 成功レスポンス（Success Response 200）："エクスポートが成功しました"
- エラーレスポンス（400）：不正または不足したパラメータに対するバリデーションエラー。

##### 顧客インポート・非同期
- エンドポイント：/api/import-customer/
- メソッド：POST
- Content-Type: multipart/form-data
- リクエストボディ：file: binary（Excel ファイル）
- 挙動：
	- バックグラウンドインポートをキューへ投入し、ImportResult レコードを作成する。
	- 即座に 200 を返し、レスポンスは "Imported Successfully"。
	- /api/import_results/ をポーリングしてステータスを追跡する。
- 成功レスポンス（Success Response 200）："インポートが成功しました"

### 5.2.3. 車両管理 API
#### 5.2.3.1. 車両データ
- ベースリソース：/api/car_data/
- メソッド：
	- GET（一覧）
	- POST（作成）
	- GET /{id}/（単一取得）
	- PATCH /{id}/（部分更新）
	- DELETE /{id}/（削除）
- 認証：必須（JWT Bearer）
- 目的：ナンバープレートに関連付けられた車両レコードを管理する。

##### 車両データ一覧・フィルター
- エンドポイント：/api/car_data/
- メソッド：GET
- クエリパラメータ：
	- name: string（顧客の漢字フルネーム contains；スペースを考慮）
	- kana_name または name_kana: string（顧客のカナ名；トークンが OR 条件でマッチ）
	- phone: string（顧客電話番号の正規化された完全一致）
	- number, hiragana_prefix, vehicle_class_number, regional_code: string（ナンバープレート項目）
- 成功レスポンス（Success Response 200）：
	- 基本項目：model、color、mileage、registered_date、expired_date、eneos_id、front_photo（存在時は URL）
	- 派生項目：
		- size, type：brand/model マスターから推論される
		- recall_info：リコール番号のリスト（存在する場合）
		- car_inspection_check_up：車検満期までの日数（QR スナップショットから計算）または null
		- oil_check_up, tire_check_up, battery_check_up, car_wash_checkup, coating_checkup：サービスリマインダー指標（整数）
		- air_check_up, eneos_check_up, line_check_up, tightening_check_up：ISO-8601 タイムスタンプ または null

##### 車両データ作成
- エンドポイント：/api/car_data/
- メソッド：POST
- ヘッダー：Content-Type: application/json
- リクエストボディ（JSON）：
	- license_plate: ID（必須）
	- brand: string（推奨）
	- model: string（推奨）
	- color: string
	- mileage: number
	- registered_date: YYYY-MM（文字列）または date
	- expired_date: YYYY-MM-DD
	- eneos_id: string
	- air_check_up_before: number
	- air_check_up_after: number
	- front_photo:ファイルアップロードは通常、複合フロー（例：license plate create_only_new）で処理される
- 挙動：brand + model が既知マスターと一致する場合、model size/type は自動補完される。
- 成功レスポンス（Success Response 201）：CarData オブジェクト（前述の派生フィールドを含む）
- エラーレスポンス（400）：不正形式フィールドに対するバリデーションエラー

##### 車両データ更新
- エンドポイント：/api/car_data/{id}/
- メソッド：PATCH
- ヘッダー：Content-Type: application/json
- リクエストボディ：任意のフィールドサブセット
- 特別な挙動：expired_date を更新した場合、 関連するナンバープレートの QR スナップショットおよび最新 QR レコードが可能な限り同期される。
- 成功レスポンス（Success Response 200）：更新された CarData オブジェクト

##### 車両データ削除
- エンドポイント：/api/car_data/{id}/
- メソッド：DELETE
- 成功レスポンス：204 No Content（標準 DRF の動作）

##### 追加操作
- リコール通知済みとしてマーク（リコール通知を「announced」としてマークし、車両データを返す）
	- エンドポイント：/api/car_data/{id}/announced_recalls/
	- メソッド：GET
- 点検ステータス更新（サービス／コンタクト履歴の更新）
	- エンドポイント：/api/car_data/{id}/update_check_up_status/
	- メソッド：PUT または GET
	- クエリ／ボディ（Query / Body）：update_type in {air_check_up, enoes_check_up, line_check_up, tightening_check_up, oil, tire, battery, car_wash, coating, checkup}
- 挙動：指定された update_type に対してタイムスタンプを設定、または ServiceHistory エントリを作成 する。あわせて 、ToDoListStatus.last_contact_history を更新 する。

#### 5.2.3.2. 車両検索
- エンドポイント：/api/car_search/
- メソッド：GET
- 認証：Required（JWT Bearer）
- 目的：車両（CarData）を検索し、車両・顧客・ナンバープレート・QR データ の複合オブジェクトとして返す。
- クエリパラメータ
	- name: string　顧客の漢字フルネーム。「姓 名」のスペース対応 contains マッチ。
	- kana_name または name_kana: string　顧客のカナ名。姓・名に対して トークン OR 条件。
	- phone: string　正規化された顧客電話番号との完全一致。
	- number: string　ナンバープレート番号。
	- hiragana_prefix: string　vehicle_class_number: string
	- regional_code: string
- スコープおよび並び順
	- 非管理者ユーザー：結果は呼び出し元ユーザーのディーラーに限定される。
	- 管理者ユーザー：すべてのディーラーを閲覧可能。
	- デフォルトの並び順（Default order）：updated_at desc
- レスポンス（200 OK, ページネーションあり）：各項目は複合オブジェクトで構成される。

```
{
	"car": { ...CarData...},
	"customer": { ...CustomerInformation... },
	"license_plate": { ...LicensePlate... },
	"qr_data": { ...QRCode... } | null
}
```

- エラーレスポンス（400）：例：パラメータ型が不正な場合。

#### 5.2.3.3. 車両メーカー/車種
- ベースリソース：/api/car_brands/
- メソッド：
	- GET（一覧）
	- POST（作成）
	- GET /{id}/（単一取得）
	- PATCH /{id}/（更新）
	- DELETE /{id}/（削除）
- 認証（Authentication）：必須（JWT Bearer・7日有効）
- 権限（Permissions）：
	- 読み取り（SAFE methods／GET）： 全ロールが利用可能
	- 書き込み（POST / PATCH / DELETE）： Admin ロールのみ
- ページネーション（Pagination）：なし（全件リストを返す）

##### 車両メーカー一覧・検索（List & Filter Car Brand）
- エンドポイント：/api/car_brands/
- メソッド：GET
- クエリパラメータ：
	- name: string（ブランド名に対する 大文字・小文字を区別しない contains）
- 成功レスポンス（Success Response 200）：
	- Brand オブジェクト（ブランド）：ネストされた model（車種）リストを含む

```
{
	"id": number,
	"name": "TOYOTA",
	"sequence": number|null,
	"models": [
		{
			"id": number,
			"name": "PRIUS",
			"size": "S|M|L|-"|null,
			"type": number|null,
			"maker": "TOYOTA",
			"sub_models": [
				{
					"id": number,
					"name": "Zvw50",
					...
				}
			],
			...
		}
	],
	...
}
```

##### 車両メーカー作成（Create Car Brand）
- エンドポイント：/api/car_brands/
- メソッド：POST
- ヘッダー：Content-Type: application/json
- リクエストボディ：name: string（ブランド名）
- 挙動：
- 成功時、ブランドの下に デフォルトモデル「その他」 が自動生成される（size "-"、type 9）。
- ブランド名の重複がある場合は バリデーションエラー を返す。
- 成功レスポンス（Success Response 201）：ネストされたモデル（「その他」含む）を含む Brand オブジェクト
- エラーレスポンス：400 Bad Request: { "Maker": ["[HONDA] is already existed."] }

##### 車両ブランド更新
- エンドポイント：/api/car_brands/{id}/
- メソッド：PATCH
- ヘッダー：Content-Type: application/json
- リクエストボディ：name: string（ブランド名）
- 挙動：
	- 成功時、ブランドの下に デフォルトモデル「その他」 が自動生成される（size "-"、type 9）。
	- ブランド名の重複がある場合は バリデーションエラー。
- 成功レスポンス（Success Response 200）：更新された Brand オブジェクト
- エラーレスポンス：400 Bad Request（上記と同様の重複名エラー）

##### 車両ブランド削除
- エンドポイント：/api/car_brands/{id}/
- メソッド：DELETE
- 成功レスポンス：204 No Content（標準 DRF の動作）

### 5.2.4. 見積もり管理 API
#### 5.2.4.1. 見積もり関連オペレーション
- ベースリソース：/api/orders/
- メソッド（Methods）：
	- GET（一覧）
	- POST（作成）
	- GET /{id}/（取得）
	- PATCH /{id}/（更新）
	- DELETE /{id}/（削除）
- 認証：必須（JWT Bearer）
- ページネーション：PageNumberPagination page_size = 20
- スコープ：
	- DEALER_MANAGER / SHOP_MANAGER（担当ショップあり）：結果は担当ショップに限定される。
	- iOS ユーザーエージェント：デフォルトで CANCELLED（キャンセル） ステータスは一覧から除外される。

##### 注文一覧・検索
- エンドポイント：/api/orders/
- メソッド：GET
- クエリパラメータ：

| 件名 | フィールド |
|----|----|
| 注文 | order_number（部分一致：icontains）, status（整数）, order_status（カンマ区切りのステータス値） |
| 日付範囲 | from_date / to_date（注文の作成日時：created_at）, sale_datetime_from / sale_datetime_to, updated_at_from / updated_at_to |
| 実際の引き取り日時 | actual_pickup_datetime_from/actual_pickup_datetime_to |
| ディーラー／ショップ | dealer, shop, shops=1,2,3 |
| 顧客 | customer_id, name (漢字：部分一致、分かち書き検索), name_kana（カナ：分かち書き検索）, family_name（文字列）, email, birth_date, postal_code, prefecture, municipality, address, telephone_number / phone (正規化された電話番号に対して完全一致), is_term_accepted（bool）, is_dm（bool）, is_personal（bool） |
| 車両（Vehicle – plate/car_data 経由） | number, hiragana_prefix, vehicle_class_number, regional_code（カンマ区切りリスト）, brand, model, sub_model, register_date_from / register_date_to（車両登録日）, expired_date_from / expired_date_to（車検有効期限）, last_updated_from / last_updated_to |
| サービス／商品 | services=<ids>, service_type=<ids>, level_one / level_two / level_three, custom_level_two（level_two の部分一致：icontains）, fee_items（名称）, product_code（部分一致：icontains）, product_name（部分一致：icontains）, quantity, sales_price, total_sales_price |
| ステータス補助 | status_bar (1-4), status_order_history (0-3) |
| セレクション | selected_only=true (リクエスト内の複雑なアイテムレベルフィルターを適用する特別なフラグ) |

##### 成功レスポンス（Success Response 200）：以下のコアフィールドを含み、ネストされたデータを持つ：
- customer_info、car_info（読み取り専用スナップショット）、license_plate（読み取り専用）、car_data
- order_items（sequence順で並び替え）、inspection_items、car_inspection_items
- car_inspection_order、car_inspection_questionnaires、inspection_order_items
- shop_data、staff_data、checking_staff_data、fixing_staff_data
- pick_up_datetime、dropoff_datetime、sales_datetime
- orderservice_set、purchased_services
- total_price、total_paid、sub_totaltotal_discount、
- signatures：customer_signature、staff_signature、car_rental_customer_signature、car_rental_staff_signature
- created_date、updated_date、shop_booking

##### 注文取得
- エンドポイント：/api/orders/{id}/
- メソッド：GET
- 成功レスポンス（Success Response 200）：リスト項目と同一構造。

##### 注文作成
- エンドポイント：/api/orders/
- メソッド：POST
- ヘッダー：Content-Type: application/json
- リクエストボディ（主なフィールド）：
	- shop（id）、staff（id）、customer_id（id）、car_data_id（id）
	- status（int）※ リペアのみ（全 item が REPAIR） の場合、注文作成は CONFIRMED までのみ許可される。
	- sales_datetime（任意）：これが指定されている場合、created_at / updated_at はこの値で上書きされる。
	- order_items：
 	- car_inspection_items、inspection_items、car_inspection_order、car_inspection_questionnaires（いずれも 任意）
	- shop_booking（任意）

```
[
	{
		"uuid": "uuid4",
		"name": "Engine Oil",
		"quantity": 1,
		"price": 5500,
		"service_id": <int>,
		"service_name": "オイル",
		"service_type": <int>,
		"level_one": "通常",
		"level_two": "5W-30",
		"level_three": "",
		"product_code": "OIL-001",
		"is_complete": true|false,
		"repair": false
	},
	...
]
```

##### バリデーションルール（作成時）：
- リペア項目が1つでもあり、status == CONFIRMED の場合：以下の項目が必須
	- dropoff_datetime
	- pick_up_datetime
	- repair_factory
	- repair_start_date
	- repair_end_date
- リペアのみの の場合：作成時の status は CONFIRMED を超えてはならない

##### 挙動：
- リクエストに 完了（complete）項目 + 保存（saved）項目 または検査項目が混在している場合：
	- サーバーは注文を 2つに分割（split） する場合がある。（例：confirmed 用と saved 用）
	- レスポンスには、unconfirmed_order（id）、unconfirmed_order_contains_pi（bool）が含まれる
- 状態が READY または PICKED_UP の場合：完了した項目について ServiceHistory が生成される。
- status > SAVED の場合：サービス種別ごとに OrderService が自動生成される。
- Tire service（タイヤ）：tightening_check_up をリセットする。
- Oil service（オイル）：first_mileage が設定されることがある。
- shop_booking が与えられた場合：新規作成 / 紐付けが行われ、Phone booking（電話予約）の編集もサポートされる

##### 成功レスポンス（Success Response 201）：メイン注文（confirmed がある場合は confirmed 側）が返され、
加えて以下を含む：

```
{
	...orderfields...,
	"unconfirmed_order_contains_pi": <bool>,
	"unconfirmed_order": <id|null>
}
```

##### 注文更新
- エンドポイント：/api/orders/{id}/
- メソッド：PATCH
- ヘッダー：Content-Type: application/json
- ボディ：作成時に受け付けるフィールドの任意のサブセット（修理項目・検査項目に関するルールは作成時と同じ）
- 挙動：
	- 合計金額を再計算する
	- 作成時と同様に unconfirmed_order とフラグが返される場合がある
- 成功レスポンス（Success Response 200）：作成レスポンスと同じ構造を持つ更新済み注文オブジェクト

##### 注文削除
- エンドポイント：/api/orders/{id}/
- メソッド：DELETE
- 挙動（Behavior）：該当 order_number に紐づく FactoryBooking レコードも削除される
- 成功レスポンス：204 No Content

##### 追加操作
- 店舗およびステータス別カウント
	- エンドポイント：/api/orders/count?shop=<id>&status=<int>
	- メソッド：GET
	- レスポンス：{ "count": <int> }

- バッチ削除
  - エンドポイント：/api/orders/batch_delete/
	- メソッド：DELETE
	- ボディ（Body）：{ "orders": [ { "id": <int> }, ... ] }
	- 挙動（Behavior）：
		- 各注文を ソフトデリート（alive = None） としてマークする
		- 関連する FactoryBooking が存在する場合は、それらも削除される

- 購買管理
	- エンドポイント：/api/orders/purchase-management/
	- メソッド：GET
	- クエリパラメータ（Query）：
		- dealer（必須）
		- product_category（任意）
		- from / to（任意）
		- actual_pickup_datetime_from / actual_pickup_datetime_to（任意）
	- エクスポート：
		- エンドポイント：/api/orders/export-purchase-management/（GET）
		- レスポンス："Exported Successfully"

- ナンバープレートの最新点検注文
	- エンドポイント：/api/orders/last-inspection-order/?license_plate=<id>
	- メソッド：GET
	- レスポンス：指定ナンバープレートに対する 最新の車両点検タイプの注文

#### 5.2.4.2. 見積もり注文検索
- エンドポイント：/api/search_orders/
- メソッド：GET
- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 目的：注文を検索し、iPad／タブレット向けの軽量コンポジットデータを返す。

##### ページネーションと並び順
- ページネーション：ページベース、page_size = 20
- 並び順：-updated_at（更新日の新しい順）

##### クエリパラメータ（Query Parameters / Filters）
- 注文：
	- order_number: string（部分一致・icontains）
	- status: int
	- order_status: コンマ区切りの整数（例：1,2,3）
	
- 日付範囲：
	- from_date / to_date: created_at（YYYY-MM-DD、to_date は 23:59:59 を含む）
	- sale_datetime_from / sale_datetime_to
	- updated_at_from / updated_at_to
	- actual_pickup_datetime_from / actual_pickup_datetime_to
	
- ディーラー／ショップ：
	- dealer: id
	- shop: id
	- shops: コンマ区切りの ID（例：1,2,3）
	
- 顧客（Customer）：
	- customer_id: id
	- name: string（漢字；姓／名トークン化 contains）
	- name_kana: string（カナ；姓／名トークン化）
	- family_name: string（icontains）
	- email: string
	- birth_date: YYYY-MM-DD
	- postal_code: string
	- address: string（住所のどちらかの行と一致）
	- municipality: string（city または town と一致）
	- telephone_number / phone: string（正規化された完全一致）
	- prefecture: string
	- is_term_accepted: bool
	- is_dm: bool
	- is_personal: bool
	- member_number, member_number_email, member_number_eneos, member_number_other
	
- 車両（ナンバープレート・車両データ経由）
	- number, hiragana_prefix, vehicle_class_number, regional_code（コンマ区切りリスト可）
	- brand, model, sub_model（コンマ区切り可）
	- register_date_from / register_date_to（car.registered_date）
	- expired_date_from / expired_date_to（車検満了日）
	- last_updated_from / last_updated_to（car_data.updated_at、to は 23:59:59 を追加）
	- register_month: YYYY-MM
	- eneos: string（car_data.eneos_id）
	
- サービス／商品：
	- services: サービス ID のコンマ区切りリスト
	- service_type: サービスタイプ（car-inspection / checkup を含む）
	- level_one / level_two / level_three: コンマ区切り
	- custom_level_two: icontains（level_two 用）
	- fee_items: 名前のコンマ区切りリスト
	- product_code: icontains
	- product_name: icontains
	- quantity: number
	- sales_price: number
	- total_sales_price: number

##### レスポンス（200 OK, paginated）
- 各アイテム：

```
     {
       "id": number,
       "license_plate": { ... },      // minimal plate summary
       "brand": "TOYOTA" | null,      // from car_data if available
       "model": "PRIUS" | null,
       "color": "white" | null,
       "car_info_color": "white" | null,
       "front_photo": "/media/..." | null,
       "customer_fullname": "山田 太郎",
       "has_repair": true|false,
       "status": number,
       "order_number": "ORD-2025-0001",
       "dropoff_datetime": "2025-10-31T09:00:00Z" | null,
       "pick_up_datetime": "2025-10-31T17:00:00Z" | null,
       "purchased_services": "オイル, 洗車, 車検",
       "updated_date": "2025-10-31T18:45:12+09:00"
     }
```

##### エラーレスポンス（Error Responses）
- 400 Bad Request：パラメータの型または形式が不正な場合。

#### 5.2.4.3. 見積もりデータエクスポート
- エンドポイント：/api/export_order/
- メソッド：GET
- 認証：必須（JWT Bearer、トークン有効期限 7日）
- 目的：注文データまたは注文に紐づく顧客情報を Excel ファイルとしてエクスポートする非同期ジョブをキューに投入 する。

##### 挙動
- エクスポート種別は is_customer によって決定される：
	- is_customer = false → 注文データのエクスポート
	- is_customer が指定されていない、または "false" 以外 →注文顧客情報（Order Customer Info）のエクスポート
- API の動作：
	- API は 即時レスポンス を返す
	- 実際のエクスポート処理は バックグラウンドジョブとしてキューに登録
	- 完了確認とファイル取得は、GET /api/export_results/ にて実施

##### クエリパラメータ
- 基本
	- dealer: id
	- dealer_name: string
	- shop: id
	- shops: コンマ区切りの shop id（例: 1,2,3）
	- staffs: コンマ区切りの staff id
	- lang: string（任意）
	- export_all: int（任意）
	- is_customer:
		- "false" → 注文エクスポート
		- その他 or 未指定 → 注文顧客情報エクスポート

- 注文日付
	- from_date, to_date
	- sale_datetime_from, sale_datetime_to
	- actual_pickup_datetime_from, actual_pickup_datetime_to
	
- 車検満了日
	- inspection_expired_start
	- inspection_expired_end

- 顧客フィルター
	- family_name, given_name, name, name_kana
	- email, birth_date
	- Line
	- house_phone_number, mobile_phone_number, phone
	- postal_code, building, prefecture, municipality, address
	- is_dm, is_personal, is_term_accepted
	- member_number
	
- 車両フィルター
	- regional_code, vehicle_class_number, hiragana_prefix, number
	- brand, model
	- registration_date_from, registration_date_to
	- expired_date_from, expired_date_to
	- last_updated_from, last_updated_to
	- register_month（YYYY-MM）
	- eneos
	
- 商品／サービスフィルター
	- services（idリスト）
	- service_type（int リスト）
	- level_one, level_two, fee_items
	- product_category
	- product_code, product_name
	- quantity, sales_price, total_sales_price
	- purchase_price, total_purchase_price
	
- 集計メタデータ
	- aggregate_dealer_code, aggregate_dealer_name
	- dealer_code, operator_code, operator_name, shop_code
	
- 走行距離
	- mileage_upper, mileage_lower

- アンケート
	- survey_response_start_date
	- survey_response_end_date

##### レスポンス
- 200 OK："エクスポートが成功しました "
- 400 Bad Request：クエリパラメータのバリデーションエラー

##### 結果確認およびダウンロード
- 以下でエクスポートジョブのステータスと生成ファイルを確認可能：
	- /api/export_results/?dealer=<id>
	- id、dealer、export_statusでフィルターが可能
	- ファイル URL はジョブ完了時に公開される、かつ、ロールベースアクセス制御（RBAC） が適用される

#### 5.2.4.4. 車検オーダー履歴（Checkup Order History）
- エンドポイント：
  - /api/car_checkup_orders/（一覧取得）
  - /api/car_checkup_orders/export/（Excelエクスポート、非同期）
- メソッド：GET
- 認証：必須（JWT Bearer）
- 認可：IsAuthenticated + メニューアクセス権限「sales_promotion.order_management.checkup_order_history」
- 目的：Web サイト向けに車検オーダー（3項目・8項目・15項目・安全点検）を一覧表示およびエクスポートする。日付範囲・ディーラー・ショップ・点検タイプでフィルタリング可能。

##### クエリパラメータ
- from_date：YYYY-MM-DD（任意。created_at >= from_date でフィルタ）
- to_date：YYYY-MM-DD（任意。created_at <= to_date 23:59:59 でフィルタ）
- inspection_type：int（0 = 3項目、1 = 8項目、2 = 15項目、3 = 安全点検）
- dealer：int（ディーラー ID）
- shop：int（任意。ショップ ID）
- lang：en | ja（エクスポート時の翻訳言語選択）
- page：int（ページネーション、一覧エンドポイントのみ）

##### リクエスト例（一覧）
```
GET /api/car_checkup_orders/?from_date=2026-01-01&to_date=2026-02-01&inspection_type=0&dealer=12&page=1
```

##### レスポンス（200 OK）— 一覧
```json
{
  "count": 1,
  "results": [
    {
      "id": 501,
      "shop_name": "Tokyo Shop A",
      "license_plate_number": "品川 500 あ 1234",
      "car_brand_model": "Toyota / Prius",
      "status": 3,
      "status_display": "Completed",
      "checkup_datetime": "2026-01-15T10:30:00Z",
      "created_datetime": "2026-01-15T10:00:00Z",
      "display_mileage": 45200,
      "attribute": {
        "inspection_type": "0",
        "inspection_area": "engine",
        "mileage": 45200
      }
    }
  ]
}
```

##### リクエスト例（エクスポート）
```
GET /api/car_checkup_orders/export/?from_date=2026-01-01&to_date=2026-02-01&inspection_type=0&dealer=12&lang=ja
```

##### レスポンス（200 OK）— エクスポート
```
"Exported Successfully"
```

##### エクスポート動作
- ExportResult レコードを「処理中」状態で作成する。
- バックグラウンドワーカーをキューに登録し、Excel ファイル（3項目・8項目・15項目・安全形式）を書き込み、ExportResult を成功または失敗に更新する。
- エラー時はトレースバックが記録され、ExportResult が失敗としてマークされる。
- フロントエンドは既存の /api/export_results/ エンドポイントをポーリングしてファイルが準備できたらダウンロードする。

##### 顧客データ可視性（エクスポート）
- Admin および Branch Admin の呼び出し元：エクスポートされた Excel において顧客フィールド（氏名、カナ、電話番号、住所等）は空白になる。
- ディーラー/ショップスタッフ：顧客詳細は完全に含まれる。

##### ステータスコード
- 200 OK：一覧取得成功 / エクスポートリクエスト受付成功
- 400 Bad Request：パラメータの型または値が不正
- 401 Unauthorized：トークンなし / 無効
- 403 Forbidden：メニュー権限不足

### 5.2.5. 予約・カレンダー API
#### 5.2.5.1. ショップ予約
- ベースリソース：/api/shop-bookings/
- メソッド：GET（一覧／取得）、POST（作成）、PUT / PATCH（更新）、DELETE（削除）
- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 権限：IsAuthenticated
- ページネーション：DRF のデフォルト設定

##### ショップ予約一覧
- エンドポイント：/api/shop-bookings/
- メソッド：GET
- レスポンス形式：
	- booking_slots: [ { id, reservation_sequence, start_time, end_time, services[], pit, slot_date } ]
	- customer_info: { ... }  // from booking or order
	- license_plate: { ... }  // from booking or order
	- reservation_staff: { id, name, ... } | null
	- work_staff: { id, name, ... } | null
	- Other fields (model): order, car_inspection_order, car_inspection_order_type, creation_type, customer_info, shop, license_plate, remarks, dropoff_datetime, pick_up_datetime, free_input_customer, reservation_staff, work_staff, created_at/updated_at, etc.

##### ショップ予約作成／更新
- エンドポイント：作成：/api/shop-bookings/、更新：/api/shop-bookings/{id}/
- メソッド：作成：POST、更新：PATCH
- ヘッダー：Content-Type: application/json
- リクエストボディ：
	- booking_slots（必須配列）
		- 各 slot の形式：{ reservation_sequence?, start_time?, end_time?, services[], pit?, slot_date? }
		- services: string 配列
	- その他必要に応じて指定するフィールド：
		- customer_info (id), shop (id), license_plate (id)
		- car_inspection_order (id), car_inspection_order_type (1=Pre-inspection, 2=Car inspection)
		- creation_type (1=Normal, 2=Manual)
		- remarks (string), dropoff_datetime (ISO 8601), pick_up_datetime (ISO 8601)
		- free_input_customer (JSON: { name, phone_number, car_model, ... })
	- 備考：更新（PATCH）は既存の booking_slots をすべて置き換える

##### 特別ビュー
- エンドポイント：/api/shop-bookings/month-view/
- メソッド：GET
- クエリパラメータ：
	- shop_id（必須）
	- start_date: YYYY-MM-DD
	- end_date: YYYY-MM-DD
- レスポンス：[ { "date": "YYYY-MM-DD", "data": [ ] } ]
- 選択ロジック：
	- shop_id が一致する予約（予約自体または関連する order / car-inspection-order 経由）
	- dropoff_datetime 〜 pick_up_datetime の期間と日付が重複するもの

##### ショップ予約削除
- エンドポイント：/api/shop-bookings/{id}/
- メソッド：DELETE
- 挙動（Behavior）：
	- 予約に order が紐づいている場合：
		- order.status == CONFIRMED のとき→ order.status は SAVED に変更され、予約は ハードデリート（204）
		- それ以外→ 400 Bad Request
	- 予約に order がない場合：予約は ハードデリート（204）

##### ステータスコード（Status Codes）
	- 200 OK：一覧／取得／更新成功
	- 201 Created：作成成功
	- 204 No Content：削除成功
	- 400 Bad Request：削除不可（order が CONFIRMED でない）、または不正な payload
	- 401 Unauthorized：トークン欠如／不正トークン

#### 5.2.5.2. 工場予約
- ベースリソース：/api/factory-bookings/
- メソッド：GET（一覧／取得）、POST（作成）、PUT / PATCH（更新）、DELETE（削除）
- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 権限／スコープ：
	- Admin：フルアクセス
	- DEALER_MANAGER： 自分のディーラーに属する工場（factory）の予約のみアクセス可能
	- その他ユーザー：ユーザープロファイルに割り当てられた工場の予約のみ

##### 工場予約一覧（List Factory Booking）
- エンドポイント：/api/factory-bookings/
- メソッド：GET
- クエリパラメータ（Query Params）：
	- mechanics: number (mechanics__id)
	- plate_number: exact (car_data__plate_number)
	- regional_code: exact (car_data__regional_code)
	- hiragana_prefix: exact (car_data__hiragana_prefix)
	- vehicle_class: exact (car_data__vehicle_class)
	- factory_name: icontains (factory__name)
	- slot_date: date (slots__slot_date)
	- repair_start_date / repair_end_date: gte/lte on start_date/end_date
	- dropoff_start_date / dropoff_end_date: gte/lte on drop_off
	- pickup_start_date / pickup_end_date: gte/lte on pickup
	- Additional supported meta fields: shop_name, status, order_number, factory, reviewer, mechanic_comments, start_date, end_date, product_category_snapshot, alt_color_code, dealer_name, color_code

##### 工場予約作成（Create Factory Booking）
- エンドポイント：/api/factory-bookings/
- メソッド：POST
- ヘッダー：Content-Type: application/json
- リクエストボディ（必須キー）：
	- order_number: string
	- factory: <id>
	- slot_ids:[ <FactoryBookingSlot id>, ... ]// 既存スロットを紐付け、これらは CONFIRMED 状態になる
	- order_items:
 	- 追加フィールド：必要に応じて任意フィールドを指定可能（drop_off、pickup、start_date、end_date、shop_name、dealer_nameなど）

 ```
[
         {
           "part_name": "...",
           "will_be_fixed": true|false,
           "position": "123.4,567.8",   // "x,y"
           "side": <int>,
           "type": <int>,
           "touch": <int>,
           "actual_touch": <int|null>,
           "after_repaired_image": "<url|null>"
         }, ...
]
```

- 挙動
	- 既存予約が存在する場合のルール：同じ order_number の既存予約が存在し、その status が BOOKED の場合：その予約はリセットされ（items・slots が削除され）、今回の内容で置き換えられる。
	- 既存予約の status が CHECKED_IN 以上場合（CHECKED_IN、REPAIRING、READY_FOR_REVIEW、REVIEWED、COMPLETE、SHOP_PICKED_UP、）、変更は不可となり 400 ParseError を返す。
	- 更新時の特別ルール：status が COMPLETE になった場合は、すべてのリンクされた slot が COMPLETED に変更され、関連する OrderService（リペア用） は、status = READY、work_completion = 100%に更新される。

##### 工場予約更新
- エンドポイント：/api/factory-bookings/{id}/
- メソッド：PATCH
- ヘッダー：Content-Type: application/json
- ボディ：FactoryBooking の任意フィールド、order_items および slot_ids は指定した場合、既存リンクを置き換える
- 成功レスポンス：200 OK（更新後の booking オブジェクト）

##### 工場予約削除
- エンドポイント：/api/factory-bookings/{id}/
- メソッド：DELETE
- 成功レスポンス：204 No Content

##### 追加操作
- メカニックの追加／削除（リスト全体を上書きせず、追加または削除のみ行う）
	- エンドポイント：/api/factory-bookings/{id}/patch-mechanic/
	- メソッド：PATCH
	- ボディ：{ "mechanics": [<MechanicStaff id>, ...], "add": true|false }
	- レスポンス：200 OK

- 日別ビュー（工場の1日単位の予約一覧）
	- エンドポイント：/api/factory-bookings/day-view/
	- メソッド：GET
	- クエリ：actory=<id>, start_date=YYYY-MM-DD, end_date=YYYY-MM-DD
	- レスポンス：[ { "date": "YYYY-MM-DD", "data": [] }, ... ]

- サマリー表示（処理台数・能力などの集計）
	- エンドポイント：/api/factory-bookings/summary/
	- メソッド：GET
	- クエリ：
		- start_date=YYYY-MM-DD（デフォルト：当月の開始日）
		- end_date=YYYY-MM-DD（デフォルト：当月の最終日）
		- factory=<id>（デフォルト：1）
	- レスポンス：

```
       {
         "actual_finished_cars": <int>,
         "actual_finished_touches": <int>,
         "remain_cars": <int>,
         "remain_touches": <int>,
         "schedule_cars": <int>,
         "schedule_touches": <int>,
         "capacity_touches": <int>,
         "capacity_cars": <int>
       }
```

##### 工場注文のエクスポート
- エンドポイント：/api/export_factory_order/
- メソッド：GET
- クエリ（任意）：factory_name、shop_name、dealer_name、order_numberstatus
- 挙動：
	- エクスポートジョブをキューに登録
	- 即時 200 OK で "エクスポートが成功しました" を返す
	- 完了したら /api/export_results/ でファイル取得

#### 5.2.5.3. ピット管理
- ベースリソース：/api/pits/
- メソッド：GET（一覧／取得）、POST（作成）、PUT / PATCH（更新）、DELETE（削除）
- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 権限：IsAuthenticated（グローバルデフォルト）
- ページネーション：PageNumber (page, page_size; デフォルト サイズ 40)
- 並び順：id 昇順がデフォルト

##### ピット一覧
- エンドポイント：/api/pits/
- メソッド：GET
- クエリパラメータ：
	- shop: number（完全一致：指定された shop ID に属する pit のみにフィルタリング）
- レスポンス表現：
	- Fields：Id、name、shop、active、sequence

##### ピット作成／更新
- エンドポイント：/api/pits/ or /api/pits/{id}/
- メソッド：POST（作成）、PATCH（更新）
- フィールド（Fields）：
	- name: string（必須）
	- shop: number（Shop ID・任意）
	- active: boolean（デフォルト = true）
	- sequence: integer（任意、表示順／並び順）
- 制約：同一 shop 内で同一 name は重複不可
- 成功レスポンス（201 Created）：id, name, shop, active, sequence

##### ピット削除（Delete Pit）
- エンドポイント：/api/pits/{id}/
- メソッド：DELETE
- 成功レスポンス：204 No Content

##### ステータスコード（Status Codes）
- 200 OK：一覧／取得／更新成功
- 201 Created：作成成功
- 204 No Content：削除成功
- 400 Bad Request：例：name + shop のユニーク制約エラー
- 401 Unauthorized：トークン欠如／不正トークン

### 5.2.6. 分析・レポート API
#### 5.2.6.1. 販売分析
- エンドポイント：
	- /api/sales-achievement/first-layer/
	- /api/sales-achievement/second-layer/
- メソッド：GET
- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 目的：
	- first-layer： 店舗単位（shop）で販売実績を集計
	- second-layer： 店舗内の スタッフ単位（staff） で販売実績を集計

##### クエリパラメータ
- dealer: string（必須）
- shop: string（任意。second-layer では必須）
- start_date: YYYY-MM-DD（任意。売上日時の下限・含む）
- end_date: YYYY-MM-DD（任意。売上日時の上限・含む。API 内部で +1 日される）
- updated_at_from: YYYY-MM-DD（任意。注文 updated_at の下限）
- updated_at_to: YYYY-MM-DD（任意。注文 updated_at の上限。API 内部で +1 日される）
- category: comma-separated ints（任意。service_type ID のリスト、CHECKUP を含む場合、点検注文もカウントされる）
- lang: string（任意。デフォルト en）
- page, month, type：validator により受理されるが 当該エンドポイントでは使用されない

##### 認可／スコープ
- 認証ユーザーのロールに応じて結果が制限される：
	- DEALER_MANAGER / SHOP_MANAGER（担当ショップあり）： 担当ショップに限定
	- その他のユーザー：指定された dealer によって制限

- レスポンス — First Layer（200 OK）
	- 形式：

```
     {
       "overall_data": {
         "qt_quantity": number,
         "qt_amount": number,
         "confirm_quantity": number,
         "confirm_amount": number,
         "qt_number_of_cars": number,
         "confirm_number_of_cars": number,
         "revenue_number_of_cars": number,
         "revenue_quantity": number,
         "revenue_amount": number,
         "order_rate_number_of_cars": number,   // %
         "order_rate_quantity": number,         // %
         "order_rate_amount": number            // %
       },
       "data_items": [
         {
           "id": number,               // shop id
           "name": "Shop Name",
           "qt_quantity": number,
           "qt_amount": number,
           "qt_unit_price": number,
           "confirm_quantity": number,
           "confirm_amount": number,
           "confirm_unit_price": number,
           "revenue_quantity": number,
           "revenue_amount": number,
           "qt_number_of_cars": number,
           "confirm_number_of_cars": number,
           "revenue_number_of_cars": number,
           "order_rate_number_of_cars": number, // %
           "order_rate_quantity": number,       // %
           "order_rate_amount": number          // %
         }
       ]
     }
```

- レスポンス — Second Layer（200 OK）
	- 形式：指定されたショップに対する スタッフ単位の項目配列。

```
     [
       {
         "id": number,               // staff id
         "name": "Staff Name",
         "qt_quantity": number,
         "qt_amount": number,
         "qt_unit_price": number,
         "confirm_quantity": number,
         "confirm_amount": number,
         "confirm_unit_price": number,
         "revenue_quantity": number,
         "revenue_amount": number,
         "qt_number_of_cars": number,
         "confirm_number_of_cars": number,
         "revenue_number_of_cars": number,
         "order_rate_number_of_cars": number, // %
         "order_rate_quantity": number,       // %
         "order_rate_amount": number          // %
       }
     ]
```

##### エクスポート
- エンドポイント：/api/export-sales-achievement/
- メソッド：GET
- パラメータ：前述のエンドポイント（dealer, shop, start_date, end_date, category など）と同一。
- 挙動（Behavior）：
	- 非同期エクスポートジョブをキューに登録する
	- 即座に "Exported Successfully" を返す
	- 生成ファイルは以下で取得可能：/api/export_results/?dealer=<id>

##### 関連エンドポイント：日次／月次販売実績
- 日次（Daily）：
	- /api/daily-sales-achievement/first-layer/
	- /api/daily-sales-achievement/second-layer/
	- /api/daily-sales-achievement/third-layer/
- 月次（Monthly）：
	- /api/monthly-sales-achievement/first-layer/
- 販売パフォーマンス：
	- /api/sales-performance/second-layer/
- エクスポート：
	- /api/export-sales-performance/
	- /api/export_daily_monthly_sales_performances/
- エラーレスポンス
	- 400 Bad Request：無効または不足しているパラメータ（例：dealer が未指定）
	- 401 Unauthorized：トークン欠如または無効トークン

#### 5.2.6.2. リペアレポート
- エンドポイント：/api/repairs/
- メソッド：GET
- 認証：必須（JWT Bearer・トークン有効期間 7 日）
- 目的：指定したディーラーに対して、日別・店舗別に集計された月次のリペア分析データを返す。

##### クエリパラメータ
- dealer: string（必須）
- month: YYYY-MM（必須）：分析対象の月

##### 認可／スコープ
- 認証ユーザーのロールに基づき結果が制限される：
	- DEALER_MANAGER / SHOP_MANAGER（担当ショップあり）：自身に割り当てられたショップに限定
	- 上記以外のユーザー：指定された dealer に基づき制限

- レスポンス（Response 200 OK）
	- 形式：

```
     {
       "days": [
         {
           "date": "01",                 // day-of-month
           "display": "Sat",             // weekday label
           "is_today": false,
           "is_past": true|false,
           "shops": [
             {
               "name": "Shop A",
               "sequence": 10,
               "touches": 5,             // sum of repair touches (quantity for repair items)
               "cars": 4,                // unique cars count
               "discount": 0.0,          // absolute total of negative sales (discounts)
               "sale": 55000.0           // total positive sales for the day
             },
             ...
           ]
         },
         ...
       ],
       "shops": [
         {
           "name": "Shop A",
           "sequence": 10,
           "monthly_total": {
             "touches": 120,
             "cars": 95,                 // unique cars in month
             "discount": 3000.0,
             "sale": 980000.0
           },
           "monthly_passed": {           // up to current day only
             "touches": 90,
             "cars": 70,
             "discount": 2500.0,
             "sale": 720000.0
           }
         },
         {
           "name": "All shops",
           "sequence": 0,
           "monthly_total": { ... },
           "monthly_passed": { ... }
         }
       ]
     }
```

- 補足事項
	- リペア項目のみ が集計対象（service_type = REPAIR）
		- 日別グルーピングはorder.repair_end_dateを使用する（存在しない場合は order.dropoff_datetime.date() にフォールバック、いずれも指定された月内に該当するものが対象）
		- “touches”は修理数量、“cars”は日単位または月単位で重複を除外した車両数をカウント。

- エラーレスポンス
	- 400 Bad Request：month の欠如／形式不正（YYYY-MM）、dealer が欠如（export 時に適用、list は month が必須）
	- 401 Unauthorized：トークン欠如／不正トークン

- エクスポート
	- エンドポイント：/api/export_repairs/
	- メソッド：GET
	- クエリ：dealer（必須）、month=YYYY-MM（必須）、lang（任意、デフォルト = en）
	- 挙動：
		- 非同期エクスポートジョブがキューに登録される
		- 即座に "Exported Successfully" を返す
		- 完了後、/api/export_results/?dealer=<id>にてファイルがダウンロード可能

#### 5.2.6.3. 顧客登録分析
- エンドポイント：
	- /api/registered-car-statistics/：Layer 1 店舗単位での集計
	- /api/registered-car-statistics/second-layer/：Layer 2 店舗内スタッフ単位での集計
	- /api/registered-car-statistics/daily/：日次（Daily）集計
	- /api/registered-car-statistics/monthly/：月次（Monthly）集計
	- /api/export-registered-car-statistics/：エクスポート（非同期）
- メソッド：GET
- 認証：必須（JWT Bearer・7 日間有効）
- 目的：店舗およびスタッフ単位で、顧客登録の完全性（completeness）と QR 取得状況（QR acquisition）を分析 する。

##### クエリパラメータ
- dealer: string（必須）
- start_date: YYYY-MM-DD（任意。 end_date とセットで指定すると created_at の範囲フィルタになる）
- end_date: YYYY-MM-DD（任意）
- shop: string（任意。0 または未指定でdealer 配下の全 shop、second-layer では必須
- month: string（任意。daily/monthly endpoints で使用）
- year: string（任意。daily/monthly endpoints で使用）
- lang: string（任意。デフォルト：en）

##### 備考
- スタッフ集計には、shop、start_date、end_dateが必須

##### 認可／スコープ（Authorization / Scoping）
- 認証ユーザーのロールに応じてデータ範囲が制限される：
	- DEALER_MANAGER / SHOP_MANAGER（担当ショップあり）：自身の担当ショップに限定
	- 上記以外のユーザー：指定 dealer の範囲に限定

##### レスポンス — Layer 1（GET /api/registered-car-statistics/）
- 形式：

```
     {
       "overall_data": {
         "total": number,
         "total_with_plate_number_only": number,
         "total_with_qr_code": number,
         "total_with_qr_acquisition_ratio": "NN%",
         "total_with_name": number,
         "total_with_phone": number,
         "total_with_name_phone": number,
         "total_with_phone_addr": number,
         "total_with_name_phone_addr": number,
         "rate_of_full_info": "NN%",
         "full_registration_ratio": "NN%",
         "total_with_line": number,
         "total_with_mail": number,
         "total_with_eneos": number,
         "total_with_others": number,
         "total_with_eneos_app": number,
         "total_with_ene_app": ""         // placeholder
       },
       "shops": [
         {
           "shop_id": number,
           "shop_name": "Shop A",
           "total": number,
           "total_with_plate_number_only": number,
           "total_with_qr_code": number,
           "total_with_qr_acquisition_ratio": "NN%",
           "total_with_name": number,
           "total_with_phone": number,
           "total_with_name_phone": number,
           "total_with_phone_addr": number,
           "total_with_name_phone_addr": number,
           "rate_of_full_info": "NN%",
           "full_registration_ratio": "NN%",
           "total_with_line": number,
           "total_with_mail": number,
           "total_with_eneos": number,
           "total_with_others": number,
           "total_with_ene_app": ""
         }
       ]
     }
```

##### レスポンス — Layer 2（GET /api/registered-car-statistics/second-layer/）
	- 形式：指定された ショップおよび日付範囲 に対するスタッフ単位（staff-level） の項目配列。

```
     [
       {
         "staff_id": number,
         "staff_name": "Staff Name",
         "total": number,
         "total_with_qr_code": number,
         "total_with_qr_acquisition_ratio": "NN%",
         "total_with_name_phone": number,
         "total_with_name_phone_addr": number,
         "full_registration_ratio": "NN%",
         "total_with_line": number,
         "total_with_eneos": number
       }
     ]
```

##### Daily / Monthly バリアント
- 対象エンドポイント：
	- /api/registered-car-statistics/daily/
	- /api/registered-car-statistics/monthly/
- パラメータ：Layer 1 と同じルールで validator によって検証される
	- dealer
	- shop
	- start_date / end_date
	- または month / year
- レスポンス：
	- 日別または月別にグルーピングされた 登録車統計データ
	- 形式は Layer 1 と同様だが、期間（day / month）ごとにグループ化される

##### エクスポート（Export）
- エンドポイント：/api/export-registered-car-statistics/
- メソッド：GET
- パラメータ：Layer 1 / Layer 2 と同じパラメータセット（dealer, shop, start_date, end_date, month/year など）
- 挙動：
	- 非同期エクスポートジョブをキューに登録
	- 即座に "Exported Successfully" を返す
	- 完了後、/api/export_results/?dealer=<id>でファイル取得可能

##### エラーレスポンス
- 400 Bad Request：無効または不足している必須パラメータ（例：dealer 未指定；second-layer の場合：shop / start_date / end_date が必須）
- 401 Unauthorized：トークン欠如／無効トークン

#### 5.2.6.4. 商品購買分析
- エンドポイント：
	- /api/product-purchase/first-table/
	- /api/product-purchase/second-table/
	- /api/export-product-purchase/
- メソッド：GET
- 認証：必須（JWT Bearer・7 日間有効）
- 目的：
	- 第1テーブル：店舗ごとに、カテゴリ別の購入構成（購入数・比率）および利益を集計。
	- 第2テーブル：店舗ごとに、サービス別の台数および利益内訳を集計。

##### クエリパラメータ
- dealer：string（必須）
- shop：string（任意／特定店舗のみに制限）
- start_date：YYYY-MM-DD（必須）
- end_date：YYYY-MM-DD（必須・API 内部で+1日補正）
- registered_start_date：車両登録日の開始日（任意）
- registered_end_date：車両登録日の終了日（任意・API 内部で+1日補正）
- service_type：カンマ区切り int（任意／Service.type フィルタ）
- services：カンマ区切り int（任意／Service.id フィルタ）
- level_one：カンマ区切り値 or ID（任意／product.level_one）
- level_two：カンマ区切り値 or ID（任意／product.level_two または order_items.name）
- fee_items：カンマ区切り値（任意／order_items.name と一致）
- car_brand：カンマ区切り文字列（任意）
- car_model：カンマ区切り文字列（任意）
- car_sub_model：カンマ区切り文字列（任意）

##### 認可／スコープ制御（Authorization / Scoping）
- 認証ユーザーの役割により、返却データは制限される：
	- DEALER_MANAGER / SHOP_MANAGER（担当店舗が割り当てられている場合）：自分が担当する店舗のみ
	- その他ユーザー（一般スタッフなど）：指定された dealer の範囲のデータのみ

##### レスポンス — 第1テーブル（GET /api/product-purchase/first-table/）
- 形式：

```
     [
       {
         "id": 0,                          // 'All Shops' aggregate row
         "name": "All Shops",
         "purchased_number_of_cars": number,
         "gross_profit": number,
         "all_registered_car": number,
         "number_1category": number,
         "number_2category": number,
         "number_3category": number,
         "number_4category": number,
         "profit_1category": number,
         "profit_2category": number,
         "profit_3category": number,
         "profit_4category": number,
         "number_1category_percentage": number,   // %
         "number_2category_percentage": number,   // %
         "number_3category_percentage": number,   // %
         "number_4category_percentage": number,   // %
         "profit_1category_percentage": number,   // %
         "profit_2category_percentage": number,   // %
         "profit_3category_percentage": number,   // %
         "profit_4category_percentage": number    // %
       },
       {
         "id": number,                     // shop id
         "name": "Shop A",
         "purchased_number_of_cars": number,
         "gross_profit": number,
         "all_registered_car": number,
         "number_1category": number,
         "number_2category": number,
         "number_3category": number,
         "number_4category": number,
         "profit_1category": number,
         "profit_2category": number,
         "profit_3category": number,
         "profit_4category": number,
         "number_1category_percentage": number,   // %
         "number_2category_percentage": number,   // %
         "number_3category_percentage": number,   // %
         "number_4category_percentage": number,   // %
         "profit_1category_percentage": number,   // %
         "profit_2category_percentage": number,   // %
         "profit_3category_percentage": number,   // %
         "profit_4category_percentage": number    // %
       }
     ]
```

##### レスポンス — 第2テーブル（GET /api/product-purchase/second-table/）
- 形式：配列

```
     [
       {
         "id": 0,                          // 'All Shops' aggregate row
         "name": "All Shops",
         "number_of_services": number,
         "all_registered_car": number,
         "purchased_number_of_cars": number,
         "gross_profit": number,
         "services": [
           {
             "name": "Repair",
             "id": number,
             "number_of_car": number,
             "number_of_car_percentage": number, // %
             "profit": number,
             "profit_percentage": number         // %
           },
           ...
         ]
       },
       {
         "id": number,                     // shop id
         "name": "Shop A",
         "number_of_services": number,
         "all_registered_car": number,
         "purchased_number_of_cars": number,
         "gross_profit": number,
         "services": [ ... as above ... ]
       }
     ]
```

##### エクスポート
- エンドポイント：/api/export-product-purchase/
- メソッド：GET
- パラメータ：上記の分析APIと同一（dealer, shop, start_date など）
- 動作：
	- 非同期ジョブをキューに登録
	- 即座に "エクスポートが成功しました" を返す
	- 完了後は、エンドポイント/api/export_results/?dealer=<id>でダウンロード可能 

##### エラーレスポンス（Error Responses）
- 400 Bad Request：無効または必須パラメータ不足（例：dealer / start_date / end_date）
- 401 Unauthorized：トークン欠如／無効

### 5.2.7. インポート・エクスポート API
#### 5.2.7.1. インポート結果
- エンドポイント：/api/import_results/
- メソッド：GET
- フィルター：dealer_id: ID（任意）
- スコープ：
	- ディーラー管理者／ショップ管理者：自分が担当するショップ内で作成されたインポート結果のみ参照可能。
- レスポンス　例：

```
[
	{
		"id": 456,
		"dealer": 12,
		"uploaded_person": "username",
		"file_name": "customers_2025-11-11.xlsx",
		"import_status": "COMPLETED|PENDING|FAILED",
		"created_at": "2025-11-11T03:22:00Z"
	}
]
```

#### 5.2.7.2. エクスポート結果
- エンドポイント：/api/export_results/
- メソッド：GET
- フィルター：id、dealer、export_status
- レスポンス例：

```
[
	{
		"id": 123,
		"export_type": "Export Customers",
		"export_status": "COMPLETED|PENDING|FAILED",
		"status_display": "Completed",
		"file": "https://.../exports/123.xlsx"|null, //may be null if role-restricted or not ready
"created_person": "username",
		"dealer": 12,
		"created_at": "2025-11-11T03:21:00Z"
	}
]
```

- 補足事項
	- ファイル URL は、エクスポートを作成したユーザーのロールと、要求を行ったユーザーのロールが一致している場合、かつエクスポートが完了している場合にのみ返される。

### 5.2.8. 声掛けリスト・アンケート API
#### 5.2.8.1. 声掛けリスト(リマインダー)
- エンドポイント：

| エンドポイント | 説明 |
|----|----|
| /api/reminders/ | リマインダー定義の CRUD（作成・取得・更新・削除） |
| /api/reminders/{id}/ | 単一リマインダーの詳細取得・操作 |
| /api/reminders/bulk-update-status/ | リマインダーの並び順変更および有効／無効フラグの一括更新（配列 Patch） |
| /api/license-plate-reminders/ | 車両に対するリマインダー状態の判定 |
| /api/reminder-memos/ | リマインダーメモ（車両 × リマインダー単位のメモ）の CRUD |

- メソッド
	- /api/reminders/: GET, POST
	- /api/reminders/{id}/: GET, PATCH, PUT, DELETE
	- /api/reminders/bulk-update-status/: PATCH
	- /api/license-plate-reminders/: GET
	- /api/reminder-memos/: GET, POST, PATCH, DELETE
- 認証：必須（JWT Bearer、トークン有効期限：7日）
- 目的：ディーラー単位でリマインダールールを管理し、車両ごとのリマインダー状態や詳細を計算・取得する。

##### クエリパラメータ
- /api/reminders/（GET）：
	- dealer: integer（必須）※ dealer を指定しない場合は 400 エラー
	- is_active: boolean（任意）
- 備考:
	- dealer に一致するリマインダー または グローバル（dealer = null） を含む
	- ページネーションなし（sequence 順で全件返却）
- /api/license-plate-reminders/（GET）：
	- dealer: integer（必須）
	- license_plate: integer（必須）

##### リクエストボディ
- Create Reminders（POST /api/reminders/）："reminders" 配列の中に複数のリマインダーをまとめて登録する形式。

```
     {
       "reminders": [
         {
           "type": int,                  // enum: 0=Coating,1=Car Wash,2=Oil,3=Tire,4=Battery,5=Air Check,6=Repair,7=Expired Date,8=Oil Date,9=Checkup,10=Engine Cleaner
           "dealer": int,
           "is_active": boolean,
           "reminder_conditions": [
             {
               "condition_type": int,    // 0=Red, 1=Yellow
               "condition_unit": int,    // 0=Week,1=Month,2=Year,3=KM
               "condition_value": int    // positive integer
             }
           ]
         }
       ]
     }
```

- 検証
	- is_active = true の場合：同一ディーラー内で、同じ type のアクティブなリマインダーが既に存在してはならない。
	- 作成時（POST）：reminder_conditions（入れ子の条件）が検証され、正しく保存される。

- リクエストボディ — リマインダー更新（PATCH /api/reminders/{id}/）
	- リマインダーのフィールド更新に加えて、既存の reminder_conditions を id を指定して更新 することが可能。

```
     {
       "type": int,
       "is_active": boolean,
       "reminder_conditions": [
         {
           "id": int,                    // required to update this condition
           "condition_type": int,
           "condition_unit": int,
           "condition_value": int
         }
       ]
     }
```

- 備考
	- 更新時に変更されるのは id を持つ条件だけ である。新しい条件は アップデート操作では作成されない。

##### リクエストボディ — ステータスおよび並び順の一括更新（PATCH /api/reminders/bulk-update-status/）
- 配列のインデックス順に基づいて リマインダーの並び順を更新 する。
- 同時に is_active（有効／無効フラグ）をトグル更新 する。

```
     [
       { "id": 12, "type": 2, "is_active": true },
       { "id": 18, "type": 2, "is_active": false }
     ]
```

- ルール
	- 送信された配列の中では、同一 type（種別）につき有効（is_active=true）なリマインダーは 1 件以下 でなければならない。これに違反した場合は 400（Bad Request） を返す。
	- 並び順は、送信された配列のインデックス（0 始まり）で更新 される。

##### レスポンス — リマインダー一覧／詳細（GET）
- レスポンスには、モデルのフィールドに加えて、ネストされた conditions（条件） および読み取り専用ヘルパー項目が含まれる。

```
     {
       "id": number,
       "type": number,                 // see enum above
       "dealer": number,
       "is_active": boolean,
       "sequence": number,
       "is_manual_reset": boolean,     // true for Air Check type, else false
       "reminder_conditions": [
         {
           "id": number,
           "condition_type": number,   // 0=Red, 1=Yellow
           "condition_unit": number,   // 0=Week,1=Month,2=Year,3=KM
           "condition_value": number
         }
       ]
     }
```

##### レスポンス — ナンバープレートリマインダー（GET /api/license-plate-reminders/）
- 指定された車両について、ディーラーに紐づくアクティブなリマインダーを対象に、計算済みのステータスおよび詳細情報を返却する。

```
     [
       {
         "id": number,
         "type": number,
         "dealer": number,
         "is_active": boolean,
         "sequence": number,
         "title": "Oil",               // localized title
         "value": "YYYY-MM-DD" | null, // computed date threshold
         "days": number | null,        // computed days offset used for condition checks
         "status": number,             // -1=White, 0=Red, 1=Yellow, 2=Green
         "is_manual_reset": boolean,   // true for Air Check
         "details": { ... },           // type-specific details
         "reminder_memo": {            // latest memo for this license plate and reminder, or null
           "id": number,
           "reminder": number,
           "license_plate": number,
           "memo": "text"
         } | null
       }
     ]
```

- ステータスロジック
	- サービス履歴に関連するリマインダーで、過去のサービス履歴が存在しない場合、ステータスは White（白） をデフォルトとする。
	- 有効期限（Expired Date）系のリマインダーで、期限値が存在しない場合もステータスは White（白） をデフォルトとする。
	- 上記以外のタイプでは、設定された条件（週／月／年／KM）に対して日付・値を比較し、Red／Yellow／Green を返す。

##### リマインダーメモ（GET / POST / PATCH / DELETE /api/reminder-memos/）
- フィールド
	- id: number
	- reminder: number（必須／リマインダーID）
	- license_plate: number（必須／ナンバープレートID）
	- memo: string
- フィルター
	- license_plate: integer
	- reminder: integer

##### ステータスコード
- 200 OK: GET、PATCH、PUT、DELETE が成功した場合
- 201 Created: POST /api/reminders/ による一括作成の成功
- 400 Bad Request: 不正または不足しているパラメータ（例：GET /api/reminders/ の dealer が不足、一括更新で同一 type のアクティブが複数、存在しない id など）
- 401 Unauthorized: トークンが不正／不足している場合

#### 5.2.8.2. アンケートメニュー
- エンドポイント：

| エンドポイント | 目的 |
|----|----|
| /api/question-menus/ | アンケートメニューの管理（作成・取得・更新・削除） |
| /api/question-menus/{id}/ | 単一アンケートメニューの管理 |
| /api/question-menus/get_question_history/ | 顧客＋車両単位のメニュー別回答履歴取得 |
| /api/questions/ | 個別の質問および選択肢の管理 |
| /api/questions/get_question_names/ | 質問名およびその選択肢一覧の取得 |

- メソッド：

| エンドポイント | 方法 |
|----|----|
| /api/question-menus/ | GET, POST, PATCH, DELETE |
| /api/question-menus/{id}/ | GET, PATCH, DELETE |
| /api/question-menus/get_question_history/ | GET |
| /api/questions/ | GET, POST, PATCH, DELETE |
| /api/questions/get_question_names/ | GET |

- 認証：必須（JWT Bearer／トークン有効期限：7日）
- 目的：
	- ディーラー単位で アンケートメニューを定義 する。
	- メニューは複数の質問を組み合わせて構成される。
	- iPad 用に最適化されたメニュー内容を取得することも可能。

##### クエリパラメータ
- /api/question-menus/（GET）
	- dealer: integer（任意）：指定されたディーラーに属するメニューのみを返す
	- ipad: truthy 値（任意）：指定されている場合、iPad 用に質問詳細を拡張したレスポンス形式を返す
- /api/question-menus/get_question_history/（GET）
	- dealer: integer（任意）：メニュー履歴のディーラーを絞り込み
	- license_plate: integer（任意）：customer_info と組み合わせて、特定車両の履歴を抽出
	- customer_info: integer（任意）：license_plate と組み合わせて対象顧客の履歴を抽出
- /api/questions/（GET）
	- dealer: integer（任意）
	- is_customer: boolean（任意）：true = 顧客向け質問セット、false = スタッフ向け質問セット
- /api/questions/get_question_names/（GET）
	- パラメータはすべて任意：/api/questions/ のフィルターを継承して動作

##### リクエストボディ — アンケートメニュー作成／更新
- POST /api/question-menus/（question IDs を利用して多対多関係を作成）

```
     {
       "name": "Inspection Menu January",
       "dealer": 12,
       "customer_set": [101, 102],   // required, non-empty; list of question IDs for customer
       "staff_set": [201, 202]       // required, non-empty; list of question IDs for staff
     }
```

- PATCH /api/question-menus/{id}/ (質問の割り当てを置き換える)

```
     {
       "name": "Inspection Menu v2",
       "customer_set": [101, 103],
       "staff_set": [201]
     }
```

##### 備考
- 作成（POST）／更新（PATCH）時には、質問は QuestionMenuQuestion を介して紐づけられる。
- PATCH を実行すると、既存の質問リンクはすべてクリアされ、送信された内容に基づいて再作成される。
- POST／PATCH の customer_set および staff_set フィールドは「書き込み専用（write-only）」であり、レスポンスには含まれない。

##### レスポンス — Questionnaire Menu（GET）
- 通常モード（ipad パラメータなし）：質問セットの ID のみ を返す（詳細情報は含まれない）。

```
     {
       "id": number,
       "name": "Inspection Menu",
       "dealer": number|null,
       "staff_set": [1, 2, 3],       // question IDs
       "customer_set": [4, 5, 6]     // question IDs
     }
```

- iPad 最適化（?ipad=1）：質問内容・選択肢・複数選択フラグを展開して返す。

```
     {
       "id": number,
       "name": "Inspection Menu",
       "dealer": number|null,
       "staff_set": [
         { "id": 201, "question": "Staff Q1", "is_multiple_choice": false,
           "choices": [ { "id": 1, "name": "Yes", "is_free_input": false }, ... ] }
       ],
       "customer_set": [
         { "id": 101, "question": "Customer Q1", "is_multiple_choice": true,
           "choices": [ { "id": 10, "name": "A", "is_free_input": false }, ... ] }
       ]
     }
```

##### レスポンス — 質問（GET /api/questions/）
- 質問とその選択肢を返す。

```
     {
       "id": number,
       "dealer": number|null,
       "is_customer": boolean,
       "is_multiple_choice": boolean,
       "question": "Text",
       "choices": [ { "id": number, "name": "Choice", "is_free_input": boolean }, ... ]
     }
```

##### レスポンス — 質問名称（GET /api/questions/get_question_names/）
- メニュー構築用のコンパクトな質問名リストを返す。

```
     [
       {
         "id": number,
         "question": "Text",
         "choices": [ { "id": number, "name": "Choice", "is_free_input": boolean }, ... ],
         "is_multiple_choice": boolean
       }
     ]
```

##### レスポンス — メニュー回答履歴（GET /api/question-menus/get_question_history/）
- 指定された顧客＋車両に対する、最新の回答サマリー付きメニューを返す。

```
     [
       {
         "id": number,
         "name": "Inspection Menu",
         "dealer": number|null,
         "answers": {
           "id": number,
           "created_at": "YYYY-MM-DDTHH:MM:SSZ",
           "updated_at": "YYYY-MM-DDTHH:MM:SSZ",
           "staff": { "id": number, "name": "..." }
         }
       }
     ]
```

##### ステータスコード
- 200 OK：GET／PATCH／DELETE の成功
- 201 Created：POST の成功
- 400 Bad Request：不正なペイロード（例：質問リストが空）、無効な ID
- 401 Unauthorized：トークン欠如／無効

#### 5.2.8.3. アンケート履歴
- エンドポイント
	- /api/question-history/：アンケート回答（履歴）の一覧取得
	- /api/question-history/export/：現在のフィルタ結果を Excel へ同期エクスポート
- メソッド：GET
- 認証：必須（JWT Bearer・有効期限 7 日）
- 目的：顧客および車両に対するアンケート回答履歴の検索およびエクスポート。

##### クエリパラメータ（フィルタ）
- shop：スタッフの所属ショップ ID（任意）
- staff：スタッフ ID（任意）
- dealer：車両のディーラー ID（任意）
- from_date：回答作成日時の下限（answer.created_at >=）
- to_date：回答作成日時の上限（日付終端まで）
- number：ナンバープレート番号
- hiragana_prefix：ナンバープレートのひらがな
- vehicle_class_number：分類番号
- regional_code：地域コード
- name_kanji：顧客の漢字名（姓の部分一致）
- name_kana：顧客のカナ名（部分一致）
- questions：カンマ区切りの質問 ID（任意、該当質問を含む履歴をフィルタ）
- choices：カンマ区切りの選択肢 ID（任意、該当選択肢を含む履歴をフィルタ）

##### 認可・スコープ
- 認証ユーザーのロールに基づいて結果が制限される。
	- DEALER_MANAGER / SHOP_MANAGER（ショップ割当あり）： 割り当てられたショップに属する履歴に限定。
	- 上記以外： 提供されたフィルタに従い、そのままの範囲で返却。

##### レスポンス（200 OK）
- 顧客情報・車両情報が付与された回答履歴行を返す。

```
     [
       {
         "id": number,                            // answer id
         "maker": "Toyota",
         "model": "Prius",
         "plate_number": "品川300 あ 12-34",
         "registered_date": "YYYY-MM-DD",
         "expire_date": "YYYY-MM-DD",
         "mileage": "12345",
         "katashiki": "ZVW30",                    // QR snapshot; may be null
         "engine_katashiki": "2ZR-FXE",           // QR snapshot; may be null
         "specification_number": "12345",         // QR snapshot; may be null
         "classification_number": "6789",         // QR snapshot; may be null
         "customer_family_name": "山田",
         "customer_given_name": "太郎",            // hidden (empty) for admin
         "address": "Tokyo ...",                  // hidden (empty) for admin
         "phone_number": "03-..."                 // hidden (empty) for admin
       }
     ]
```

##### 備考
- 管理者（admin）の場合、customer_given_name、address、phone_number などの個人特定情報はブランクで返される。
- エクスポートエンドポイントは 生成された .xlsx ファイルを即時返却 し、キュー処理は行わない。

##### ステータスコード
- 200 OK：一覧取得またはエクスポート成功
- 401 Unauthorized：トークン欠如／無効
- 400 Bad Request：無効なフィルタ値

### 5.2.9. システム管理 API
#### 5.2.9.1. ユーザー管理
- エンドポイント：

| エンドポイント | 目的 |
|----|----|
| /api/user-profiles/ | Web ユーザー（ディーラー／ショップマネージャー、店舗スタッフ）の管理 |
| /api/user-profiles/{id}/reset-password/ | ユーザーのパスワードをリセット（サーバー側で生成） |
| /api/tablet-user-profiles/ | タブレットユーザー（role = tablet）の管理 |
| /api/send-initial-setup-emails/ | 選択したユーザーへ初期設定メールを送信 |
| /api/password-setup/ | 署名付きトークンを用いたパスワード設定／検証 |
| /api/get-all-user-full-name/ | 指定ディーラーのユーザー氏名一覧（重複除去）取得 |

- メソッド：

| エンドポイント | 方法 |
|----|----|
| /api/user-profiles/ | GET, POST, PATCH, PUT, DELETE |
| /api/user-profiles/{id}/reset-password/ | POST |
| /api/tablet-user-profiles/ | GET, POST, PATCH, PUT, DELETE |
| /api/send-initial-setup-emails/ | POST |
| /api/password-setup/ | GET（検証）, POST（パスワード設定） |
| /api/get-all-user-full-name/ | GET |

- 認証：必須（JWT Bearer　有効期限 7 日）
	- /api/password-setup/（GET/POST）は AllowAny（トークンベースのセットアップ用）
- 目的
	- ユーザープロファイル、ロール、ショップ紐づけの管理
	- セットアップメール送信ワークフローおよびパスワード初期化のサポート

##### クエリパラメータ（filters）
- /api/user-profiles/（GET）
	- dealer：文字列（任意、dealer=null のユーザーも含む）
	- roles：カンマ区切り整数（任意）
	- id：ユーザー ID（任意）
	- role：ロール ID（任意）
- /api/tablet-user-profiles/（GET）：上記と同じフィルタセット
- /api/get-all-user-full-name/（GET）：dealers：カンマ区切りディーラー ID（任意）

##### 認可・スコープ
- 一覧取得
	- SHOP_MANAGER / SHOP ロール：自分自身のプロファイルのみ取得
	- DEALER_MANAGER：管理者が担当するショップの部分集合に属するユーザー（ロール一致）を取得。条件に合わない場合は 自分のプロファイルのみ
	- Tablet user プロファイル一覧：アクセス可能なショップに紐づくタブレットユーザーのみ
- 削除
	- プロファイル削除時、auth_user の is_active も false に更新される。

##### リクエスト／レスポンス — ユーザープロファイルGET / PUT / PATCH（/api/user-profiles/）
- フィールド一覧：

```
     {
       "id": number,
       "username": "string",                 // read-only
       "role": number,
       "role_display": "string",
       "family_name": "string",
       "email": "string|null",
       "email_setup_status": number,
       "shops": [number, ...],
       "branch_id": number|null,
       "dealer_id": number|null,
       "password": "string|null",            // write-only on update; alphanumeric, >= 6 chars, not username/old password
       "confirm_password": "string|null",    // must match password if provided
       "change_password": boolean|null       // when true, applies password change
     }
```

- 挙動
	- shops の更新は、既存セットを完全に置き換える。
	- role が ADMIN の場合、基盤の auth user に対して is_superuser が付与される。
	- change_password=true が指定された場合、パスワードが再設定され、password_at が更新される。

##### リクエスト — パスワードリセット（POST /api/user-profiles/{id}/reset-password/）
- リクエストボディ不要
- 挙動：ランダムパスワードを生成して設定、email_setup_status を EMAIL_NOT_SENT にリセット
- レスポンス：{ "message": "Reset Password Successfully." }

##### 初期セットアップメール送信（POST /api/send-initial-setup-emails/）
- body：

```
     {
       "user_ids": [ number, ... ]  // required; Dealer Manager / Shop roles supported
     }
```

- 挙動
	- セットアップメールを送信する。（対象はディーラーマネージャー、またはユーザーのメールアドレス。もしくは紐づくショップのメールアドレス。）
	- 送信成功時は email_setup_status を EMAIL_SENT に更新。
	- 送信数および警告情報をレスポンスとして返却。
- レスポンス

```
     {
       "success": true,
       "message": "Initial setup emails processed",
       "emails_sent": number,
       "users_processed": number,
       "warnings": [ "text", ... ]   // present when applicable
     }
```

##### トークンによるパスワードセットアップ（/api/password-setup/）
- GET：トークン検証
	- クエリ： token=string（必須）
	- 成功レスポンス：{ valid: true, username, full_name, user_id }
	- エラーレスポンス：無効・期限切れ・再利用済みトークンであることを示す。

- POST：パスワード設定
	- ボディ：{ "token": "string", "password": "string", "confirm_password": "string" }
	- バリデーション内容：最低文字数：6 文字以上、英数字要件、パスワード不一致、再利用禁止
	- 成功レスポンス：{ "success": true, "message": "Password has been set successfully" }

##### ステータスコード
- 200 OK：GET／PUT／PATCH／DELETE、reset-password、setup emails、password-setup の成功
- 201 Created：プロファイル作成系 ViewSet の POST 成功
- 400 Bad Request：不正なペイロード（例：パスワード要件違反、ロール／メール必須、user_ids 欠如）
- 401 Unauthorized：トークン欠如／無効
- 404 Not Found：トークン処理時のユーザー未検出
- 409 Conflict：ユーザー名の重複（作成時）

#### 5.2.9.2. ディーラー管理
- エンドポイント：

| エンドポイント | 目的 |
|----|----|
| /api/dealers/ | ディーラー情報の管理 |
| /api/get-all-dealers/ | ユーティリティ：{id, name} の一覧取得（ロールスコープ適用） |
| /api/get-all-dealers-etr/ | 公開ユーティリティ：{id, name} の一覧取得（AllowAny） |
| /api/import-dealers/ | ファイルからディーラー情報をインポート（POST multipart） |

- メソッド：

| エンドポイント | 方法 |
|----|----|
| /api/dealers/ | GET, POST, PATCH, PUT, DELETE |
| /api/get-all-dealers/ | GET |
| /api/get-all-dealers-etr/ | GET |
| /api/import-dealers/ | POST |

- 認証：必須（JWT Bearer・トークン有効期限 7 日）
- 目的：
	- ディーラーのレコード（名称、支店、税率、ロゴ、タイプ、地域、各種フラグ）の作成・管理
	- 各種ユーティリティ API（一覧取得／インポート）の提供

##### クエリパラメータ
- /api/dealers/（GET）：
	- id：数値（任意）
	- branch：数値（任意、支店 ID）
- /api/get-all-dealers/（GET）：branch_id：数値（任意）
- /api/get-all-dealers-etr/（GET, 公開）：パラメータなし

##### 認可・スコープ
- 一覧取得の挙動（/api/dealers/）
	- Dealer / Shop ロール（Dealer Manager 以下）：現在のユーザーに紐づくディーラーのみ表示
	- Branch Manager：ユーザーが所属する支店（branch）に紐づくディーラーのみ
	- その他のロール：パーミッションの範囲で全ディーラーが対象
- パーミッション
	- IsAuthenticated + DealerPermission

##### リクエストボディ — ディーラー作成／更新（/api/dealers/）
- フィールド：

```
     {
       "name": "string",                     // required, unique per branch (alive)
       "branch": number,                     // required (branch id)
       "tax_rate": "12.34",                  // decimal(4,2)
       "logo": <file>|"path"|null,           // optional (multipart upload supported)
       "is_production": boolean,             // default true
       "default_region": number|null,        // optional (region id)
       "is_approval_function": boolean,      // default false
       "dealer_type": number,                // 0=Regular, 1=ENEOS
       "is_default_sort": boolean,           // default true
       "filter_config": { ... } | null       // JSON; optional
     }
```

##### 備考
- 同一（name, branch）で既存の有効ディーラーが存在する場合、サーバーは 400 を返し、メッセージは “Dealer Name is Duplicated” となる。

##### レスポンス（200 / 201）
- ディーラーオブジェクト（計算済みヘルパー含む）：

```
     {
       "id": number,
       "name": "string",
       "branch": number,
       "branch_name": "string",
       "tax_rate": "12.34",
       "logo": "url|path",
       "is_production": boolean,
       "default_region": number|null,
       "is_approval_function": boolean,
       "dealer_type": number,                // 0=Regular, 1=ENEOS
       "is_default_sort": boolean,
       "filter_config": { ... } | null,
       "shops": number                       // count of shops under this dealer
     }
```

##### ディーラーインポート（POST /api/import-dealers/）
- Content-Type： multipart/form-data
- Body：file：<binary>（必須）
- 挙動：取引単位でディーラーをインポートする。成功時は "Import successful!" を返す。

##### ステータスコード
- 200 OK：GET / PATCH / PUT / DELETE 成功、インポート成功
- 201 Created：作成成功
- 400 Bad Request：バリデーションエラー（例：重複ディーラー名＋支店）、インポート時のファイル欠如
- 401 Unauthorized：トークン欠如／無効
- 403 Forbidden：DealerPermission 不足
- 404 Not Found：無効な dealer id

#### 5.2.9.3. ショップ管理
- エンドポイント：/api/shops/
- メソッド：GET, POST, PUT, DELETE
- 認証：必須（JWT Bearer・7 日有効）
- 目的：ディーラー配下のショップを管理する（住所・連絡先情報・稼働フラグなど）。

##### クエリパラメータ（GET /api/shops/）
- id：数値（任意）
- dealer：数値（任意、ディーラー ID）
- name：文字列（任意）
- email：文字列（任意）
- phone_number：文字列（任意）
- fax_number：文字列（任意）
- is_active：boolean（任意）
- is_production：boolean（任意）
- web：truthy 値（任意、スタッフ・ピットをネストした拡張レスポンスを返す）

##### 認可・スコープ
- 一覧取得
	- Tablet / Shop Manager / Shop / Dealer Manager：現在のユーザーに紐づくアクティブなショップのみ
	- Branch Manager：自分の支店に属するショップ
	- その他：dealer.alive=True のすべてのショップ
- メソッド別パーミッション
	- GET：Tablet User 以上
	- PATCH / PUT：Shop Manager 以上
	- POST / DELETE：Dealer Manager または superuser

##### リクエストボディ — ショップ作成／更新
- 主要フィールド：

```
     {
       "dealer": number,                 // required on create (write-only)
       "name": "string",
       "email": "string|null",
       "phone_number": "string",
       "fax_number": "string",
       "is_active": boolean,
       "is_production": boolean,
       "default_region": "string|null",
       "address_line_1": "string",
       "address_line_2": "string",
       "prefecture": "string",
       "town": "string",
       "city": "string",
       "building": "string",
       "postal_code": "string"
     }
```

##### 備考
- レスポンスには読み取り専用フィールドとしてdealer_info（ディーラー詳細） とinspection_factory（有効な場合）が含まれる。

##### レスポンス — Shop
- 型（例）

```
     {
       "id": number,
       "dealer": number,                  // write-only on input
       "dealer_info": { ... },            // dealer object
       "name": "Shop A",
       "email": "a@example.com",
       "phone_number": "03-...",
       "fax_number": "03-...",
       "is_active": true,
       "is_production": true,
       "default_region": "Kanto",
       "inspection_factory": { ... } | null,
       "address_line_1": "...",
       "address_line_2": "...",
       "prefecture": "...",
       "town": "...",
       "city": "...",
       "building": "...",
       "postal_code": "..."
     }
```

##### レスポンス — Web ヴァリアント（?web=1）
- ネストされたスタッフ情報とピット情報を返却し、インライン編集用に最小限のフィールドのみを含む。

```
     {
       "id": number,
       "name": "Shop A",
       "address_line_1": "...",
       "address_line_2": "...",
       "postal_code": "...",
       "email": "a@example.com",
       "phone_number": "03-...",
       "fax_number": "03-...",
       "is_active": true,
       "staffs": [
         { "id": number|null, "name": "Staff 1", "active": true, "shop_id": number, "sequence": number }
       ],
       "pits": [
         { "id": number|null, "name": "Pit 1", "active": true, "shop_id": number, "sequence": number }
       ]
     }
```

##### ステータスコード
- 200 OK：GET / PATCH / PUT / DELETE 成功
- 201 Created：POST 成功（作成完了）
- 400 Bad Request：バリデーションエラー
- 401 Unauthorized：トークン欠如／無効
- 403 Forbidden：権限不足
- 404 Not Found：ショップ ID が無効

#### 5.2.9.4. ログ管理
- エンドポイント：GET /api/logs/
- 認証：JWT 必須（7日有効）
- 目的：Web / iPad 操作を含むシステム操作ログを取得する。

##### クエリパラメータ（フィルタ）
- names：カンマ区切りの姓（完全一致）：例：names=Yamada,Suzuki

##### レスポンス（200 OK）
- 最新順のログエントリ配列

```
     [
       {
         "id": number,
         "device": "web" | "ipad",
         "ios_version": "string|null",
         "mobile_version": "string|null",
         "ip_address": "IPv4|null",
         "action": "string",
         "action_type": "authentication|register_user|change_password|operation|search|export",
         "action_type_display": "Authentication|Register User|Change Password|Operation|Search|Export",
         "actor": "string|null",
         "family_name": "string|null",
         "target_user": "string|null",
         "detail": "string",
         "created_at": "YYYY-MM-DDTHH:MM:SSZ"
       }
     ]
```

##### 備考
- 結果は id の降順（新しいものが先） で返却される。
- action_type_display は、action_type を人間向けに変換した表示用ラベル。

##### ステータスコード
- 200 OK：取得成功
- 401 Unauthorized：トークンなし／無効

#### 5.2.9.5. エラーハンドリング方針
##### HTTP ステータスコード
- 200 OK：GET／PATCH の成功
- 201 Created：POST の成功
- 400 Bad Request：入力データ不正・リクエスト形式不備
- 401 Unauthorized：認証必須またはトークン不正
- 403 Forbidden：権限不足
- 404 Not Found：対象リソースなし
- 500 Internal Server Error：サーバ内部エラー

##### エラーレスポンス形式
- シンプルエラー例

```
"Error message string"
```

- 詳細エラー例

```
{
  "error": "Specific error description",
  "detail": "Additional error details"
}
```

- バリデーションエラー（Django REST Framework）

```
{
  "field_name": ["Field-specific error message"],
  "non_field_errors": ["General validation errors"]
}
```

- ナンバープレート作成エラー例

```
{
 "non_field_errors": [
        "The fields alive, number, hiragana_prefix, vehicle_class_number, regional_code, dealer must make a unique set."
    ]
}
```

- カスタムエラー例

```
{
  "detail": "License Plate is required"
}

{
  "error": "reminder 123 does not exist"
}

{
 "details": "Customer with that Kanji Name and Phone Number is already existed"
}
```


#### 5.2.9.6. 権限・ブランチアカウント管理

##### 5.2.9.6.1. 権限カタログ
- エンドポイント：/api/permissions/
- メソッド：GET
- 認証：必須（JWT Bearer）
- 目的：利用可能な権限のマスターカタログを返す。

##### レスポンス（200 OK）
```json
[
  {
    "id": 1,
    "major_key": "settings",
    "middle_key": "shop_information_settings",
    "minor_key": "user_profiles",
    "pkey": "settings.shop_information_settings.user_profiles",
    "sequence": 10,
    "is_active": true,
    "description": ""
  }
]
```

##### 5.2.9.6.2. ロール権限管理
- エンドポイント：
  - /api/role-permissions/（一覧取得・新規作成）
  - /api/role-permissions/{id}/（更新・削除）
- メソッド：GET, POST, PATCH, DELETE
- 認証：必須（JWT Bearer）
- 目的：ディーラーごと（Branch Admin の場合はブランチごと）のロールレベル権限ベースラインを管理する。

##### クエリパラメータ（一覧）
- dealer_id：number（任意）
- branch_id：number（任意）

##### バリデーション（作成/更新）
- role = 6（Branch Admin）の場合、サーバーは dealer = null を強制し、(role, branch) の一意性を適用する。
- それ以外の場合、(role, dealer) の一意性を適用する。
- 更新後、同じ role/scope の下のすべてのユーザー単位オーバーライドがリセットされ、ユーザーは新しいベースラインにフォールバックする。

##### リクエストボディ（作成/更新）
```json
{
  "role": 0,
  "dealer": 12,
  "branch": 3,
  "permissions": [1, 2, 3, 5]
}
```

##### レスポンス（200 OK / 201 Created）
```json
{
  "count": 1,
  "results": [
    {
      "id": 7,
      "role": 0,
      "dealer": 12,
      "dealer_name": "Tokyo Dealer",
      "branch": 3,
      "branch_name": "Tokyo Branch",
      "permissions": [1, 2, 3, 5]
    }
  ]
}
```

##### 5.2.9.6.3. ユーザー権限管理
- エンドポイント：
  - /api/user-profiles/{id}/permissions/（ユーザーの有効権限を取得）
  - /api/user-profiles/{id}/update-permissions/（ユーザー単位のオーバーライドを更新）
  - /api/user-profiles/self-permissions/（現在のユーザーがアクセスできる pkey を取得）
- メソッド：GET, PATCH
- 認証：必須（JWT Bearer）
- 目的：ユーザー単位の有効権限を読み取り、ロールベースラインの上にユーザー単位オーバーライドを管理する。

##### 更新動作（PATCH）
- allow = リクエスト − role_baseline、deny = role_baseline − リクエスト として計算される。
- 有効権限 = ロールベースライン + allow − deny

##### リクエストボディ（更新）
```json
{ "permissions": [1, 2, 3, 5, 10, 12] }
```

##### レスポンス（200 OK）— 有効権限
```json
[1, 2, 3, 5, 10, 12]
```

##### レスポンス（200 OK）— Self Permissions（pkeys）
```json
[
  "settings.shop_information_settings.user_profiles",
  "factory_menu.factory_settings.factory_capacity"
]
```

##### 5.2.9.6.4. ブランチアカウント管理（Branch Account Management）
- エンドポイント：
  - /api/branch-account-user-profiles/（Branch Admin ユーザーの一覧取得・作成）
  - /api/branch-account-user-profiles/{id}/（更新・ソフトデリート）
  - /api/branch-account-user-profiles/{id}/reset-password/（パスワードリセットおよびセットアップメール再送）
- メソッド：GET, POST, PATCH, DELETE
- 認証：必須（JWT Bearer）
- 認可：Admin のみ作成・削除が可能。Branch Admin はリード操作のみ。

# 6. プロセスシーケンス図
## 6.1. 認証APIの処理フロー
##### ログインフロー
<img width="429" height="196" alt="image" src="https://github.com/user-attachments/assets/7271fd48-4f8b-4881-be26-7c4756970b09" />

##### ログアウトフロー
<img width="429" height="125" alt="image" src="https://github.com/user-attachments/assets/37252148-7b86-4c57-952e-c4051f15c644" />

## 6.2. 顧客管理APIの処理フロー
##### 顧客検索
<img width="429" height="122" alt="image" src="https://github.com/user-attachments/assets/bbd1a0eb-23bd-4d8e-9637-f4c6dd01c67c" />

##### 顧客の新規作成または取得
<img width="429" height="122" alt="image" src="https://github.com/user-attachments/assets/aabfebcf-1587-4e9c-9fe0-f74fad2a7b1f" />

## 6.3. ナンバープレート管理APIの処理フロー
##### ナンバープレート作成（車両データ付き）
<img width="428" height="257" alt="image" src="https://github.com/user-attachments/assets/f73c83ff-e232-4d5b-85cd-e4613d5254f8" />

##### ナンバープレート作成
<img width="428" height="279" alt="image" src="https://github.com/user-attachments/assets/029be497-3198-425c-9edf-8012f1d62579" />

##### ナンバープレート CRUD 操作
<img width="428" height="596" alt="image" src="https://github.com/user-attachments/assets/7cb8a3e3-b134-4918-85d0-1d0c3ce55a51" />

##### ナンバープレート更新、及び、削除
<img width="440" height="629" alt="image" src="https://github.com/user-attachments/assets/393115a4-86e0-48bd-a581-7b4957925229" />

##### ナンバープレート購入履歴
<img width="440" height="629" alt="image" src="https://github.com/user-attachments/assets/dcfd966c-4ea7-4d66-b6fc-25c09b995feb" />

## 6.4. 車両管理APIの処理フロー
##### 車両データ管理
<img width="440" height="556" alt="image" src="https://github.com/user-attachments/assets/84d9216f-0a4a-4982-bd58-e67aa954ef45" />

##### 車両データの更新、及び、削除
<img width="428" height="639" alt="image" src="https://github.com/user-attachments/assets/a4801726-516c-49e4-8357-760ac271d93d" />

##### 車両検索
<img width="440" height="546" alt="image" src="https://github.com/user-attachments/assets/115b522e-6173-4fa1-a389-5067f49fd950" />

##### 安全点検ステータスの更新
<img width="440" height="492" alt="image" src="https://github.com/user-attachments/assets/b0e6daa3-9e5b-4937-beae-8eb6760dfcff" />

##### 車両リコール
<img width="440" height="513" alt="image" src="https://github.com/user-attachments/assets/c15d1e9f-7c7e-4caf-a920-02a8e66ee791" />

##### 車両メーカー管理
<img width="440" height="549" alt="image" src="https://github.com/user-attachments/assets/a2e08e6c-3cbc-427e-bb39-e451a7834339" />

##### 車両メーカーの更新、及び、削除
<img width="440" height="642" alt="image" src="https://github.com/user-attachments/assets/635504e2-0d61-418a-abd6-fb768c490940" />

## 6.5. 見積管理APIの処理フロー
##### 見積の新規作成、及び、取得
<img width="440" height="542" alt="image" src="https://github.com/user-attachments/assets/67e9b5c8-1c95-433f-8902-03b8d1ce8797" />

##### 見積検索
<img width="440" height="632" alt="image" src="https://github.com/user-attachments/assets/30e78fc6-87b7-4dd4-8cc2-2a70dd09c42c" />

##### 見積の一括削除
<img width="440" height="412" alt="image" src="https://github.com/user-attachments/assets/89e3b71c-2cd2-4e5b-afa3-6b4a11d3f16b" />

##### 見積の購買管理
<img width="440" height="319" alt="image" src="https://github.com/user-attachments/assets/46295d1e-bc25-4d5c-b482-bde903ceb8ba" />

## 6.6. 予約・カレンダーAPIの処理フロー
##### ショップ予約管理
<img width="440" height="548" alt="image" src="https://github.com/user-attachments/assets/e05b8bda-89ac-4a92-acb7-c6ae55d2d253" />

##### ショップ予約の更新、及び、削除
<img width="440" height="559" alt="image" src="https://github.com/user-attachments/assets/39197bd8-3d4f-4e5a-886f-82e7d3a36c70" />

##### 工場予約管理
<img width="440" height="452" alt="image" src="https://github.com/user-attachments/assets/5cf16a53-74d5-4130-8291-16911702b942" />

##### 工場予約の技術者割り当て
<img width="440" height="310" alt="image" src="https://github.com/user-attachments/assets/e5c1bb0c-29ec-440b-b033-0ac9b80fb036" />

## 6.7. 分析・レポートAPIの処理フロー
##### 販売分析
<img width="440" height="354" alt="image" src="https://github.com/user-attachments/assets/dda11c65-2297-471c-a052-e17d428f58de" />

##### 商品購買レポート
<img width="440" height="304" alt="image" src="https://github.com/user-attachments/assets/c2b12b25-8ad0-44c6-addd-47c0694ec64a" />

## 6.8. 声掛けリストAPIの処理フロー
##### 声掛けリスト（リマインダー）管理
<img width="440" height="491" alt="image" src="https://github.com/user-attachments/assets/340b78a4-10dd-435b-a337-54fdc336b18f" />

##### 声掛けリストあ（リマインダー）の更新、及び、削除
<img width="440" height="472" alt="image" src="https://github.com/user-attachments/assets/3049648e-a293-4713-91f8-bd5294490607" />

## 6.9. アンケートメニューAPIの処理フロー
##### アンケートメニュー管理
<img width="440" height="541" alt="image" src="https://github.com/user-attachments/assets/00bc35cd-1dbb-4fbe-b308-34368f556ba5" />

##### 質問項目管理
<img width="440" height="559" alt="image" src="https://github.com/user-attachments/assets/cbb0d1c2-159f-4348-9d30-745624f4985d" />

## 6.10. アンケート履歴APIの処理フロー
##### アンケート履歴
<img width="440" height="476" alt="image" src="https://github.com/user-attachments/assets/b1ae9cfb-0141-42ff-bba4-2f33c364cf70" />

## 6.11. ユーザー管理APIの処理フロー
##### ユーザー管理
<img width="440" height="561" alt="image" src="https://github.com/user-attachments/assets/119a7023-0443-4701-bab4-7cd25f459012" />

##### ユーザーの更新、及び、削除
<img width="440" height="576" alt="image" src="https://github.com/user-attachments/assets/99ce1b42-4cc7-4720-a04b-c8518332dd3c" />

##### タブレットユーザー管理
<img width="440" height="488" alt="image" src="https://github.com/user-attachments/assets/b65007ae-f300-45b1-b3d9-020ecc6c3cb6" />

## 6.12. ディーラー管理APIの処理フロー
##### ディーラー管理
<img width="440" height="572" alt="image" src="https://github.com/user-attachments/assets/9166da1c-a309-4630-95ae-d37568a4bf6f" />

## 6.13. ショップ管理APIの処理フロー
##### ショップ管理
<img width="440" height="593" alt="image" src="https://github.com/user-attachments/assets/51103ae5-fffe-4499-913c-9272922b5788" />

##### ショップの更新、及び、削除
<img width="440" height="621" alt="image" src="https://github.com/user-attachments/assets/62175412-9407-43fc-aca3-1860cb4ba313" />

## 6.14. ログ管理APIの処理フロー
<img width="440" height="487" alt="image" src="https://github.com/user-attachments/assets/214acbea-c4f6-42c4-8b13-e120f8f49bcc" />

## 6.15. インポート・エクスポートAPIの処理フロー
##### インポート結果
<img width="440" height="498" alt="image" src="https://github.com/user-attachments/assets/f07c7354-f166-41db-bedf-e2762d5c1e55" />

##### エクスポート結果
<img width="440" height="456" alt="image" src="https://github.com/user-attachments/assets/1523fc46-a0d1-4ba9-a7f0-dd124abdc07c" />

## 6.16. 権限管理 & Branch Admin 処理フロー
本セクションは権限管理および Branch Admin に関する4つのシーケンスフローを定義する。

##### 管理者によるロール権限設定（Admin Configures Role Permissions）
<img src="https://raw.githubusercontent.com/kariba1141/YuichiMatsuda-pa/main/images/6.16a_admin_configures_role_permissions.png" alt="Admin Configures Role Permissions" style="max-width:100%;" />

##### ユーザー単位のサイドバーオーバーライド & Branch Admin ログイン
<img src="https://raw.githubusercontent.com/kariba1141/YuichiMatsuda-pa/main/images/6.16b_per_user_sidebar_branch_admin_login.png" alt="Per-User Sidebar Override and Branch Admin Login" style="max-width:100%;" />

##### フィーチャーページアクセスチェック（Feature Page Access Check）
<img src="https://raw.githubusercontent.com/kariba1141/YuichiMatsuda-pa/main/images/6.16c_feature_page_access_check.png" alt="Feature Page Access Check" style="max-width:100%;" />


# 7. 運用・保守設計
## 7.1. 環境構成
本システムは3つの環境で稼働する。本番・ステージング・開発環境はすべて同一のアーキテクチャを共有し、規模・データ・アクセスポリシーのみが異なる。

##### 本番環境
- AWS ECS（Fargate）：Django アプリケーションを Gunicorn（3 ワーカー、6000 秒タイムアウト）構成で稼働
- RDS（PostgreSQL）：Multi-AZ 構成、SSL 通信を有効化
- Redis：バックグラウンドジョブ処理およびキャッシュ用途
- S3：画像・ドキュメント・エクスポートファイルのストレージ
- ロードバランサ：AWS Application Load Balancer（ALB）によるトラフィック分散

##### ステージング環境
- 本番アーキテクチャと同一構成（コンポーネント・トポロジー同一）。リリース前検証および UAT に使用。
- AWS ECS（Fargate）：Django アプリケーションを Gunicorn 構成で稼働
- RDS（PostgreSQL）：Multi-AZ 構成、SSL 通信を有効化
- Redis：バックグラウンドジョブ処理およびキャッシュ用途
- S3：画像・ドキュメント・エクスポートファイルのストレージ
- ロードバランサ：AWS Application Load Balancer（ALB）によるトラフィック分散

##### 開発環境
- 本番アーキテクチャのミラー構成。インテグレーションテストおよびフィーチャー開発に使用。
- AWS ECS（Fargate）：Django アプリケーションを Gunicorn 構成で稼働
- RDS（PostgreSQL）：Multi-AZ 構成、SSL 通信を有効化
- Redis：バックグラウンドジョブ処理およびキャッシュ用途
- S3：画像・ドキュメント・エクスポートファイルのストレージ
- ロードバランサ：AWS Application Load Balancer（ALB）によるトラフィック分散

## 7.2. バックアップ方針
##### RDSスナップショット（RDS Snapshots）
- 自動バックアップ：1 日 1 回の自動スナップショットを取得し、保持期間は 7 日間とする。
- 手動スナップショット：主要リリースに合わせ、毎週手動スナップショットを取得。
- リージョン間バックアップ：スナップショットはセカンダリリージョンへ複製し、災害対策を強化。
- ポイントインタイムリカバリ：直近 7 日間について、任意時点への復元が可能。

##### S3 Glacierアーカイブ
- ライフサイクルポリシー：7 年以上経過したログファイルを Glacier にアーカイブ。
- 取得戦略：事業継続性の確保を目的として、Standard Retrieval を採用。
- コスト最適化：頻繁にアクセスされるファイルに対しては Intelligent Tiering を利用し、ストレージコストを最適化。

## 7.3. 監視設計
##### CloudWatchメトリクス
- アプリケーションメトリクス：CPU 使用率、メモリ使用量、ディスク使用量を監視
- データベースメトリクス：接続数、クエリ性能、ストレージ使用量を監視
- カスタムメトリクス：API レスポンスタイム、バックグラウンドジョブのキュー長など
- ログ集約：アプリケーションログを CloudWatch Logs に集約

##### SNS 通知
- 重大アラート：データベース障害、アプリケーションクラッシュ
- パフォーマンスアラート：高 CPU 使用率（50％以上）、メモリ問題（75％以上）
- セキュリティアラート：認証失敗、疑わしいアクセス
- メンテナンス通知：定期メンテナンス実施に関する通知

# 8. 外部システム連携
##### ETR-BUYシステム連携APIエンドポイント
- 認証：POST /api/token-auth/
- 商品一覧取得：GET /api/product_lists/
- 商品コード検証：GET /api/check-product-code-cta/

##### 認証方式
- トークンタイプ：環境変数に保存される静的 JWT トークン
- トークン保存場所：ETR_JWT_TOKEN 環境変数
- トークン形式：Authorization ヘッダに Bearer トークンとして送信
- 現状：トークンには有効期限が設定されている
- トークン更新ポリシー：更新は手動で実施する必要がある
##### 連携機能
- リアルタイム商品データ同期
- 商品コード検証
- 在庫ステータス確認
- リトライロジックを伴うエラーハンドリング（最大 3 回再試行）
- テスト設計の概要（Test Design Overview）

# 9. テスト設計概要
##### ユニットテスト
- フレームワーク：Django 標準の TestCase / APITestCase を使用
- カバレッジ：テストは Django アプリ構成（apps/*/tests.py）に基づいて整理
- データベース：PostgreSQL を利用した専用のテストデータベース構成
- iOS：Swift 単体テストでは XCTest および Mockingbird を使用

##### 結合テスト
- API テスト：Django REST Framework のテストクライアントでエンドポイントを検証
- iPad 連携：iOS クライアントとの API 通信の動作確認
- ETR 連携：外部システム（ETR）との連携テスト

##### システムテスト
- エンドツーエンドテスト：AWS 環境上でワークフロー全体を検証
- パフォーマンステスト：同時接続ユーザー数を想定した負荷試験
- セキュリティテスト：認証・認可の検証

# 10. システム機能概要
## 10.1. ユーザー管理
ユーザー管理システムは、アクセス制御およびユーザーデータ管理を安全かつ効率的に行うことを目的として設計される。
- 認証

最新かつ安全な認証方式を採用しており、Web ユーザーとタブレットユーザーはそれぞれ専用のログインプロセスを使用します。

さらに、ブルートフォース攻撃対策として、ログイン失敗が 10 回続いた場合にアカウントを一時ロックし、30 分間ログインを無効化する保護機能を備えています。

- ユーザープロファイル管理

システムは基本情報に加えて、ユーザーごとの詳細かつ専門的なプロファイル情報を管理可能です。

また、ユーザーデータの作成・取得・更新・削除（CRUD）に対応した API が提供されており、管理者はアカウント管理を全面的に制御できます。パスワードリセット機能も含まれています。

- ロールとアクセス権限

データアクセスは RBAC（Role-Based Access Control）により厳格に制御されます。

ロールには、管理者、ディーラーマネージャー、ショップマネージャー、ショップスタッフが含まれ、各ロールには明確な権限が割り当てられています。

これにより、ユーザーは自身の業務に必要な情報のみにアクセスできるよう制御されています。

- タブレットユーザー作成フロー
<img width="224" height="213" alt="image" src="https://github.com/user-attachments/assets/851fbedd-9807-4df1-9f76-6d158796b36a" />

- Webユーザー作成フロー
<img width="315" height="596" alt="image" src="https://github.com/user-attachments/assets/4bca387e-873b-4233-82fd-b334ae867c9c" />

- ユーザー管理範囲
本システムは、ロールベースアクセス制御（RBAC）に基づく包括的なユーザー管理機能を提供される。

	- タブレットユーザー（TABLET_USER = -1）
	
	店舗におけるタブレット業務に限定されたアクセス権を持つユーザー。

	- ショップマネージャー（SHOP_MANAGER = 0）
	
	店舗単位の業務およびスタッフ管理を担当。
	
	- ディーラーマネージャー（DEALER_MANAGER = 1）
	
	複数店舗に跨るディーラー全体の業務管理を担当。

	- ブランチマネージャー（BRANCH_MANAGER = 2）

	地域拠点（支社）レベルの業務管理を担当。

	- 管理者（ADMIN = 3）

	システム全体のアクセス権限を持ち、すべての管理機能を操作可能。

	- ファクトリーユーザー（FACTORY_USER = 4）
	
	工場業務および関連ワークフローにアクセスできるロール。

	- ショップ（SHOP = 5）
	
	店舗エンティティ用のロールで、店舗固有の業務操作を行うために用いられる。

## 10.2. 商品管理

商品管理システムは高い柔軟性を備えており、特に自動車関連の幅広い商品およびサービスに対応できるよう設計。

- 商品データ構造
	- サービスレベルの分類：商品はサービス種別（車検、リペア、洗車、オイル、タイヤ、バッテリー など）ごとに分類。
	- 商品階層構造：各サービスカテゴリ内の商品は、サービス → 商品グループ → 商品の 3 層構造で管理。
	- 商品タイプ：本システムでは、ビジネス要件に応じて 通常商品、工賃、工賃グループの 3 種類の商品タイプを区別して管理。
- ETR-BUY システムとの連携

システムは ETR-BUY と連携し、車検注文に必要な部品・費用（Part Fee）の商品データを取得します。車種ごとの部品情報は カテゴリ ID と型式番号を用いて取得。

また、車検カテゴリで取り込む ETR 商品を識別するため、車検 の “部品・工賃アイテム” に専用の ETR-BUY 商品タイプを設定し、ETR-BUY 由来であることを明確化。

- データ管理

商品データ管理のための包括的な API を提供し、Excel ファイルによる一括インポートおよびエクスポート機能もサポート。

- 見積もりシステムでの利用

見積もり作成時、商品情報はその時点の「スナップショット」として保存。

これにより、注文確定後にマスターデータが変更されても、注文内部のデータは変わらず保持される。

- 商品管理フロー
<img width="440" height="427" alt="image" src="https://github.com/user-attachments/assets/085711e0-eb51-4828-ae91-e302ff1742af" />

## 10.3. 見積もり管理

見積もり管理システムは、一般サービスから専門的な業務まで幅広いビジネスプロセスをサポートするよう設計されています。

- 見積もりタイプ

以下のような多様な注文タイプをサポートします：

- 
	- 通常見積もり
	- 車検見積もり
	- 車両預かり証
	- 携行缶詰替え販売
	- 安全点検見積もり

- 見積もりステータスサイクル

注文は以下の明確なステータスを経て進行します：

1. 保存済
2. 予約済
3. 作業開始
4. 作業終了
5. 納車済
6. キャンセル

車検注文は特化した 6 ステッププロセス に従い、各ステップで特定の点検データと承認を記録します。

- 見積もりデータ構造

注文内には以下のデータが「スナップショット」として保存されます：

- 
	- 顧客情報
	- 車両情報
	- 商品情報（Order Items 内の product フィールド）

- 他モジュールとの連携
	- 工場連携：リペア見積りを外部工場へ提出可能
	- 予約システム：自店舗予約カレンダーと連携してスケジューリングを管理
 	- 声掛けリスト(リマインダー)：見積もり完了時にサービス履歴を自動作成

- 見積もり管理フロー
<img width="440" height="384" alt="image" src="https://github.com/user-attachments/assets/c968c434-e848-40eb-b4b1-3810806c50b4" />

## 10.4. 車両・顧客管理

本システムは、顧客・車両・サービス履歴の関係性を管理する、アプリケーションの基盤となる重要なモジュール。

- 顧客データ管理
システムは以下の包括的な顧客情報を管理：

- 
	- 基本連絡先情報（氏名、電話番号、メール）
	- サービス・見積書に利用される住所情報
 	- 適切なセキュリティ対策を施した個人情報
  	- 顧客の嗜好や履歴情報

個人情報は暗号化され検索可能であり、重複顧客を防ぐバリデーション機能も備える。

- 車両データ管理

管理する車両情報は以下を含む：

- 
	- メーカー、車種名、年式
	- ボディーカラーおよびボディータイプ
	- 来店ごとの走行距離履歴
	- 必要に応じたリース/任意保険情報

また、各ディーラーにおいて完全なナンバープレート番号が、顧客・車両・注文を紐づける主キーとして利用されます。

- データリレーションシップ

柔軟な関係性管理を実現するデータ構造を採用。

- 
	- 1 人の顧客が複数の車を所有可能（1 対 多）
	- 1 台の車は所有者が変わってもサービス履歴は維持される
	- サービス記録は、所有者変更に関わらず該当車両に恒久的に紐づく

- 車両・顧客管理フロー
<img width="440" height="383" alt="image" src="https://github.com/user-attachments/assets/657d8e58-34bb-4248-9d96-f73e5ec1e464" />

# 11. データフロー設計

データフロー図は、ETRCARS システム内で情報が入力から保存までどのように流れるかを示しています。

## 11.1. ユーザーフロー
- ユーザー作成
	- 管理者（Admin）またはディーラーマネージャーは、Angular の Webユーザー／タブレットユーザー機能から新規ユーザーを作成します。
	- システムはロールベースの作成権限を強制します：
		- 管理者：すべてのユーザー種別を作成可能
		- ディーラーマネージャー：自分の担当店舗に対して ショップマネージャー と ショップ ユーザー のみ作成可能
	- ユーザー作成フォームには、クライアント側／サーバー側の両方でのバリデーションが実装されています：
		- 必須項目の入力チェック（ユーザー名、氏名など）
		- メールアドレスを必要とするロールでのメール形式チェック
		- パスワード要件（6 文字以上の英数字）
		- ユーザー名の重複チェック
- ロール割り当てとバリデーション
	- ユーザーアカウントは、適切なロールを割り当てて作成されます：
		- 管理者（最上位権限）
		- ディーラーマネージャー（ディーラー単位のアクセス権）
		- ショップマネージャー（ショップ単位の単位のアクセス権）
		- ショップ（ショップ単位でアクセスが制限されています）
		- タブレットユーザー（iPad デバイス向けアクセス）
		- ファクトリーユーザー（工場向けアクセス）
	- システムはロールに応じた適切な関連付けを強制します：
		- 各ユーザーは、担当するディーラーまたはショップに紐づけられる
		- ショップマネージャー および ショップ ユーザーは 必ず 1 つのショップにのみ 割り当てられる
		- ディーラーマネージャー は、自分が所属するディーラー内の複数店舗に紐づけ可能
		- 管理者はショップやディーラーへの割り当て不要

- セキュリティ制御
	- ロールベースの権限チェックにより、ユーザーは自分より高い権限を持つユーザーを作成できない。
	- ショップ割り当てバリデーションによって、アクセス権のない店舗への割り当てを防止する。
	- システムは、ユーザー作成操作に対して IP アドレス、タイムスタンプを含む監査ログ を保持する。
	- ブルートフォース対策により、同一ソースからの連続した登録試行を遮断する。

- アカウント確認
	- システムは、新規ユーザーのアクティベーション用にセットアップメールを送信する機能を持つ。（安全なパスワード設定リンクを含む）
	- メールに含まれるセットアップリンクには有効期限が設定されており、1 回のみ使用可能。
	- パスワードリセット機能は、登録済みメールアドレスによる本人確認を必須とする。
	- 初期パスワードは安全に生成され、適切に伝達される。

## 11.2. 車両およびナンバープレート情報のフロー
- コアプロセスフロー
	- タブレットユーザーは iPad のインターフェースからナンバープレート番号を入力。
	- 車両情報を素早く取得するために、QR コードによる読み取りにも対応しています。
		- 車種に応じて複数の QR コード形式をサポート：
			- 軽自動車：3 つの QR コード
			- 普通車：5 つの QR コード
			- 超小型車：2 つの QR コード
		- QR コードは AVFoundation フレームワークを用いて専用ロジックで解析
		- デコードされた情報は、日本の車両登録情報形式に基づいて車両データとして抽出されます
	- システムはナンバープレートおよび車両データ（QR入力があれば QR データも含む）をデータベースに登録します。
	- ナンバープレートは LicensePlateCustomerInformation リンクテーブルを介して顧客に紐づけられます。
	- 1人の顧客に複数の車両を登録可能です。
	- Web 画面からのインポートにも対応しています。

- エラーハンドリング
	- ナンバープレートバリデーション
		- 必須項目チェック（地域名、分類番号、ひらがな、ナンバー）
		- ナンバープレート形式の整合性チェック（先頭が「0」のプレートは禁止など）
		- 無効な入力に対する適切なエラーメッセージ
		- データベース側で重複プレート登録を防止
	- QR コード関連のエラー処理
		- QR コード形式と構造の事前バリデーション
		- 部分的・傾いた QR の読み取りに対するリカバリ処理
		- 車種ごとの QR コード形式に対応
		- スキャン時の可視的フィードバック（ハイライト表示など）
	- ネットワークエラー処理
		- ナンバープレート登録 API の再試行処理（リトライロジック）
		- 通信失敗時の適切なユーザーメッセージ
	- データバリデーション
		- バックエンド送信前の必須項目チェック
		- 日本語文字（漢字・かな）を含むライセンスデータの適切なエンコード
		- 全角数字 → 半角数字の変換処理（例：「１２３４」→「1234」）

- テストケース
	- ナンバープレートテストケース
		- 地域名の抽出精度テスト（例：「岐阜」「東京」「大阪」）
		- 数字＋日本語混在プレートの解析（例：「３C1ろ」）
		- ひらがな接頭辞の認識テスト（例：「あ」「ろ」「し」）
		- カタカナ対応（例：「カ」）
		- 分類番号の桁数バリエーションの扱い（例：「５B2」「３A」）
		- スペース揺らぎへの対応
		- 全角数字から半角数字への変換（例：「１２３」「３５８」）
		- 完全数値分類番号の解析（例：「３３６」）
	- 車両データテスト
		- 車両データ初期化処理（有効・無効入力への対応）
		- 必須項目の欠落チェック
		- ネットワーク送信用ディクショナリへのシリアライズ精度
		- 車両色名の日本語翻訳
		- メーカー／モデル表示形式の整形
		- 車種カテゴリとローカライズ表示の検証
		- 外部ソースからの画像読み込み／処理
		- 特殊文字・極端値・任意項目欠落などのエッジケース
		- 各モジュール間でのデータ整合性の維持
		- プレート情報を含む車両情報リストの整形表示

## 11.3. 顧客情報フロー
- タブレットユーザーは iPad から顧客情報を入力、Web ユーザーは Web インターフェースからインポート可能
- 個人情報（名前・電話番号・住所など）は以下の方式で暗号化：
	- 暗号化アルゴリズム：AES-256-GCM
	- 実装：Django encrypted fields
	- キー管理：AWS Secrets Manager でキーを分離管理し、ローテーションポリシーを適用
	- フィールドレベル暗号化：各機密フィールドを個別に暗号化
- 顧客情報は車両ナンバープレートと紐づく

## 11.4. 商品・サービスフロー
- サービスは種類別（修理、洗車、オイル交換など）に分類され、Web 画面からインポート可能
- 商品は価格情報とともにインポートされ、適切なサービスカテゴリに紐づけ
- 探索しやすいカテゴリ構造で整理
- サービス／商品は各ディーラー単位で管理

## 11.5. 見積もりフロー
- 見積もり作成時の処理フロー：
	- タブレットユーザーは顧客・車両情報を選択(※顧客・車両なしでも注文作成可能)
	- 商品/サービスを数量と価格付きでカートへ追加
	- 見積もりは、以下のステータスを順に進行：
	- 保存済 → 予約済 → 作業開始 → 作業終了 → 納車済
- スタッフ割り当て（ステータス別）
	- 保存済：見積担当スタッフ
	- 予約済：予約担当スタッフ
	- 作業開始：作業開始スタッフ
	- 作業終了/納車済：作業完了スタッフ
- 連携する 2 種類の予約システム
	- 自店舗予約カレンダー：ピット割当・時間帯管理
	- 工場連携：外部修理工場とのスケジューリング
- その他見積もりタイプ
	- 車検：多段階プロセス＋検査結果管理
	- 安全点検：3種類の安全点検項目＋結果記録
	- 車両預かり証：車両状態を顧客署名付きで記録
	- 代車貸出証：レンタル契約＋車両状態記録
	- 携行缶詰替え販売：燃料缶販売（法令準拠）
	- 部品注文：特別部品の注文・納期管理

すべてのデータフローは Django REST Framework による API エンドポイントを使用し、Angular および iOS クライアントは標準化された HTTP リクエストを通じてこれらのエンドポイントを利用します。

## 11.6. エンドポイント権限表

凡例：O＝許可、X＝禁止

| エンドポイント | メソッド | 管理者 | ディーラーマネージャー | ショップマネージャー | ショップユーザー | タブレットユーザー |
|----|----|----|----|----|----|----|
| /api/user-profiles/ | GET | O | O | X | X | X |
| /api/user-profiles/ | POST | O | O | X | X | X |
| /api/user-profiles/{id}/ | PATCH | O | O | X | X | X |
| /api/user-profiles/{id}/ | PUT | O | O | X | X | X |
| /api/user-profiles/{id}/ | DELETE | O | O | X | X | X |
| /api/user-profiles/{id}/reset-password/ | POST | O | O | X | X | X |
| /api/tablet-user-profiles/ | GET | O | O | O | X | X |
| /api/tablet-user-profiles/ | POST | O | O | O | X | X |
| /api/tablet-user-profiles/{id} | PATCH | O | O | O | X | X |
| /api/tablet-user-profiles/{id} | PUT | O | O | O | X | X |
| /api/tablet-user-profiles/{id} | DELETE | O | O | O | X | X |
| /api/send-initial-setup-emails/ | POST | O | O | O | X | X |
| /api/password-setup/ | GET | O | O | O | O | O |
| /api/password-setup/ | POST | O | O | O | O | O |
| /api/get-all-user-full-name/ | GET | O | O | O | X | X |
| /api/license_plates/ | GET | O | O | O | O | O |
| /api/license_plates/ | POST | O | O | O | X | O |
| /api/license_plates/{id}/ | PATCH | O | O | O | X | X |
| /api/license_plates/{id}/ | DELETE | O | O | X | X | X |
| /api/license_plates/create_only/ | POST | O | O | O | X | O |
| /api/license_plates/create_only_new/ | POST | O | O | O | X | O |
| /api/license_plates/{id}/last_purchase_history/ | GET | O | O | O | O | O |
| /api/car_data/ | GET | O | O | O | O | O |
| /api/car_data/ | POST | O | O | O | X | O |
| /api/car_data/{id}/ | PATCH | O | O | O | X | O |
| /api/car_data/{id}/ | DELETE | O | O | X | X | X |
| /api/car_data/{id}/announced_recalls/ | GET | O | O | O | O | O |
| /api/car_data/{id}/update_check_up_status/ | GET | O | O | O | O | O |
| /api/car_data/{id}/update_check_up_status/ | PUT | O | O | O | X | O |
| /api/car_search/ | GET | O | O | O | O | O |
| /api/car_brand/ | GET | O | O | O | O | O |
| /api/car_brand/ | POST | O | X | X | X | X |
| /api/car_brand/{id} | PATCH | O | X | X | X | X |
| /api/car_brand/{id} | DELETE | O | X | X | X | X |
| /api/customer_info/ | GET | O | O | O | O | O |
| /api/customer_info/ | POST | O | O | O | X | O |
| /api/customer_info/customer-info-web/ | GET | O | O | O | O | X |
| /api/customer_info/{id}/ | GET | O | O | O | O | O |
| /api/customer_info/{id}/ | PATCH | O | O | O | X | O |
| /api/customer_info/{id}/ | DELETE | O | O | X | X | X |
| /api/customer_search/ | GET | O | O | O | O | O |
| /api/export-customer/ | GET | O | O | O | X | X |
| /api/orders/ | GET | O | O | O | O | O |
| /api/orders/ | POST | O | O | O | X | O |
| /api/orders/{id} | PATCH | O | O | O | X | O |
| /api/orders/{id} | DELETE | O | O | X | X | X |
| /api/orders/count?shop=&status= | GET | O | O | O | X | X |
| /api/orders/batch_delete/ | DELETE | O | O | X | X | X |
| /api/orders/tablet-search/ | GET | O | O | O | O | O |
| /api/orders/purchase-management/ | GET | O | O | O | X | X |
| /api/orders/last-inspection-order/?license_plate= | GET | O | O | O | O | O |
| /api/order_items/ | GET | O | O | O | O | O |
| /api/order_items/ | POST | O | O | O | X | O |
| /api/order_items/{id} | PATCH | O | O | O | X | O |
| /api/order_items/{id} | DELETE | O | O | X | X | X |
| /api/search_orders/ | GET | O | O | O | O | X |
| /api/export_order/ | GET | O | O | O | X | X |

# 12. ハードウェアおよびインフラ仕様

ETRCARS システムは AWS 上にデプロイされており、以下のハードウェアおよびインフラ構成を使用。

## 12.1. Web／API サーバー

アプリケーションのバックエンドロジック（Django API および Nginx リバースプロキシ）は Docker によりコンテナ化され、AWS のコンテナオーケストレーションサービスによって管理される。

- コンピュート（ECS with Fargate）
	- AWS Elastic Container Service（ECS）を採用し、起動タイプとして AWS Fargate を使用しています。Fargate はサーバーレスのコンテナ実行基盤であり、サーバー管理を AWS が自動的に行うため、アプリケーションに専念。
	- サービスは一定数の稼働タスクを維持するよう設定され、高可用性を確保。
	- 各タスクのスペックは 2 vCPU / 4GB メモリ。
- スケーラビリティ（Auto Scaling）。
	- ECS サービスにはオートスケーリングが設定されており、CPU 使用率やメモリ使用率などのリアルタイム指標に基づいてタスク数を自動的に調整。
	- CPU 使用率が 50％超、またはメモリ使用率が 75％超 の場合、自動的にスケールアウト。
	- トラフィック高負荷時のパフォーマンスを維持し、低負荷時はコストを最適化。
- 負荷分散（ALB）
	- ALB は Web クライアントおよび iPad アプリからの全トラフィックのエントリーポイントとして機能。
	- ALB はリクエストを複数の ECS タスクに分散し、ヘルスチェックによって正常なインスタンスのみにルーティング。
	- SSL/TLS 終端（HTTPS 通信の終端処理）も ALB で行い、安全な通信を提供。

## 12.2. データベースサーバー

システムの主要データベースとして、高信頼性のマネージドリレーショナルデータベースサービスを利用。

- Amazon RDS for PostgreSQL
	- Amazon RDS を使用して PostgreSQL を稼働。
	- RDS はハードウェアプロビジョニング、DB セットアップ、パッチ適用、バックアップなどの運用作業を自動化。
	- DB インスタンスのスペックは 2 vCPU / 8GB メモリ。
- 高可用性
	- RDS は Multi-AZ 構成でデプロイされ、別 AZ に同期スタンバイレプリカを保持。
	- 障害発生時には自動フェイルオーバーが実行され、ダウンタイムを最小限に抑える。
- 災害復旧
	- 毎日の自動スナップショットを取得しており、7 日間の保持期間内でポイントインタイムリカバリが可能。

## 12.3. キャッシュ層

アプリケーションのレスポンス改善およびデータベース負荷軽減のため、キャッシュ層を導入。

- Amazon ElastiCache for Redis
	- Redis を使用したマネージドインメモリデータストアを採用。
	- 主な用途：
		- 非同期タスクキュー（Django-RQ）
			- Redis は Django-RQ のメッセージブローカーとして動作。
			- アプリケーションが投入したジョブは Redis 上のキューに保持され、ワーカーが取り出してバックグラウンド処理を実行。
		- アプリケーションキャッシュ
			- 高負荷クエリ結果や事前計算済レスポンスをキャッシュし、レスポンス高速化を実現。
- ワークフロー概要（Workflow Overview）
1. アプリケーションがジョブ（例：インポート／エクスポート処理）を Redis に投入
2. RQ ワーカーがキューをポーリングし、ジョブを取得してバックグラウンドで実行
3. 進捗やステータスは Redis に保存
4. 処理完了後、結果を記録し、一時キーはワーカーのルールに従いクリーンアップされる

## 12.4. ストレージ
- Amazon S3
	- S3 は、画像やドキュメント、システム生成ファイル（Excel など）を保存するための主要ストレージとして使用。また、システムログや DB バックアップの保存先としても利用し。
	- 7 年以上経過したログファイルは S3 Glacier にアーカイブし、耐久性とコスト効率を両立。

## 12.5. ネットワーキング

ETRCARS のインフラは安全性と分離性を確保したネットワーク構成で提供。

- ネットワーク分離（Amazon VPC）
	- システムは Amazon VPC 内で稼働し、パブリックサブネットとプライベートサブネットを組み合わせた安全な構成。
- 外部接続
1. NAT Gateway：

プライベートサブネット（Django アプリなど）からインターネットへのアウトバウンド通信が可能。ETRBUY や Google Vision API など外部 API へのアクセスが可能。

2. VPC Endpoints：

Amazon ECRや Secrets Managerへの通信を、インターネットを経由せず AWS ネットワーク内で完結。

- セキュリティ
	- 公開されるのは Application Load Balancer のみで、パブリックサブネットに配置。
	- ECS タスクおよび RDS などのバックエンドコンポーネントは、Application Subnet（アプリ用プライベートサブネット）、Data Subnet（DB 用プライベートサブネット）に隔離され、外部から直接アクセスできないよう保護される。

# 13. ソフトウェアコンポーネント

ETRCARS システムは、以下のソフトウェアコンポーネントを利用。

## 13.1. バックエンドAPI サーバー(Django)
##### 目的

Django アプリケーションは、プラットフォーム全体の「中枢神経系」として機能。

主な役割は、Angular Web アプリおよび iOS iPad アプリが利用する、安全で堅牢かつスケーラブルな RESTful API を提供すること。

ビジネスロジックの実行、データ処理、セキュリティポリシーの適用、データストレージおよび外部サービスとの連携など、あらゆる中核処理を担当。

##### 主な特徴
- RESTful アーキテクチャ

Django REST Framework（DRF）により構築されており、クリーンで標準準拠、拡張性の高い API を提供。

これにより、多様なクライアントアプリケーションが容易に統合。

- モジュール設計

モノリシック構成でありながら、users、orders、products などビジネスドメインごとの Django アプリに分割され、高い可読性と管理性を保つ。

- ORM を利用したデータアクセス
- 
PostgreSQL との通信は Django ORM を使用し、複雑な SQL を Python コードとして安全かつ簡潔に記述できます。これにより開発効率向上と SQL インジェクションリスクの低減につながる。

- セキュリティ重視の設計
	- JWT を用いた認証。
	- RBAC（ロールベースアクセス制御）による詳細な権限管理。
	- 重要な顧客データの暗号化など、セキュリティを最優先に設計。

- 非同期処理の統合

Django-RQ と Redis を用いて、レポート生成などの重い処理をバックグラウンドへオフロード。

これにより API の応答性を保ち、より快適なユーザー体験を提供。

## 13.2. Webクライアントアプリケーション(Angular)

##### 目的

Angular アプリケーションは、管理スタッフおよびマネージャー向けの Web ベース管理ポータル。

##### 主な特徴
- SPA

ページ全体のリロードなしに高速な画面遷移が可能で、デスクトップアプリのような滑らかな操作性を提供。

- コンポーネントベース設計

UI は独立かつ再利用可能なコンポーネントで構成され、コードの拡張性と保守性を大幅に向上。

- データ駆動・API 依存

ビジネスロジックは一切持たず、すべて Django API を通じてデータ取得・登録・更新・削除を行う。フロントとバックエンドの責務を明確に分離した設計。

- ロールベース UI（RBAC 連動）

ログインユーザーのロールに応じて表示されるメニューや操作権限が変化。

不要な情報は隠蔽され、業務に必要な機能のみが表示される。

## 13.3. iOSクライアントアプリケーション(Swift/UIKit)
##### 目的
iPad 上で動作する iOS アプリケーションは、店舗のフロント業務においてスタッフが使用する主要ツール。

このアプリの主な目的は、モバイル POS（Point-of-Service）および工場作業管理ツールとして機能し、顧客および車両と直接対応するスタッフの業務を支援すること。

車両の預かりから引き渡しまで、顧客サービス全体のワークフローをデジタル化し、効率化するよう設計。

アプリが対応する主な業務は以下のとおり：

- 顧客・車両チェックイン

既存顧客の検索、新規顧客登録、車両との紐付けを迅速に行う。

- 注文作成

その場で新規サービスを作成し、商品やサービスを選択して即時見積もりを提示できる。

- エビデンス取得
iPad のカメラを使用して車両の損傷、修理箇所、関連書類などの画像を撮影し、工場作業オーダーに直接添付。

- 顧客署名取得

作業の承認やサービス完了の確認のため、顧客のデジタル署名を取得。

- 利用環境

本アプリは オフラインモードには対応しておらず、利用にはインターネット接続が必須。

##### 主な特徴
- ネイティブパフォーマンスとユーザー体験

Swift/UIKit を用いたネイティブアプリケーションとして構築されており、最高レベルのパフォーマンス、応答性、信頼性を提供。

ユーザーインターフェースは iPad の大画面タッチ操作に最適化されており、店舗スタッフが直感的かつ効率的に操作できるよう設計。

- ハードウェア統合

本アプリは iPad に搭載されたハードウェア機能を最大限に活用。

特に、車両損傷や修理箇所などの撮影に使用するカメラ、作業承認や完了確認に必要なデジタル署名取得のためのタッチスクリーンとの連携は、ペーパーレス化のワークフローを支える重要な要素。

- タスク特化・ロール特化型インターフェース

ユーザーインターフェースは、店舗スタッフの業務に特化したデザインとなっています。店内オペレーションに必要な情報と機能のみを表示し、管理ポータル全体の複雑さは UI から排除される。

バックエンドの API も iPad 用に最適化された専用シリアライザーを使用し、必要な形式でデータを返すよう設計。

##### 限定的なオフライン機能
- 長時間のオフライン利用には非対応

本アプリケーションは、在庫情報、価格情報、顧客情報、予約情報などの中央データがリアルタイムで必要となる業務特性上、長時間のオフライン運用を前提として設計されていない。

この設計方針は、安定したネットワーク環境が前提となる一般的な店舗運用環境と整合している。

- 一時的な通信断への最低限の対応

アプリはオンライン環境に最適化されていますが、短時間の通信途切れに対しては以下の基本的な処理を提供：
1. ネットワークエラー処理
2. API 呼び出し失敗時の簡易的なリトライ機能
3. 現在のセッション内に限った一時的なメモリキャッシュ

# 14. インターフェース設計仕様

ETRCARS システムでは、iOS アプリケーションおよび Web アプリケーションそれぞれに対して包括的な UI 設計を提供。

## 14.1. iOSアプリケーション UI
### 14.1.1. ログイン画面
- ロゴの明確な表示
画面上部に ETRCARS のロゴを大きく表示し、ブランドアイデンティティを強調。
これにより、ユーザーは正しいアプリケーションにアクセスしていることを即座に認識できる。

- ユーザー認証フィールド

ユーザー名とパスワードの入力フィールドを明確に配置し、ラベルも適切に設定されています。入力形式の検証（バリデーション）を備えており、送信前に正しい形式で入力されているかを確認することで、ユーザーが必要情報を正確に入力できるよう支援。

- エラーメッセージ表示

ログインに失敗した場合、明確かつ簡潔なエラー文を表示。
これにより、誤ったユーザー名、無効なパスワード、その他の問題が原因であるかをユーザーに知らせ、混乱することなく修正できるよう案内。
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/d1d138b7-d566-49a8-b392-160762af2db2" />

| No. | フィールド名 | 説明 |
|----|----|----|
| 1 | ログイン SS | ログイン ID：ユーザーのログイン ID（スタッフ ID）を入力するフィールド |
| 2 | パスワード | パスワード：パスワードを入力するフィールド |
| 3 | ログイン | ログインボタン：入力した認証情報を確認し、ログインを実行するボタン |

### 14.1.2. ホーム画面
- ナンバープレート検索のための数値キーパッド

ホーム画面は大きな数値キーパッドを中心としており、これは車両検索を開始するための主要なツールとして機能しますこのデザインは最も一般的なユーザータスクである「ナンバープレートによる車両検索」を優先し、アプリ起動時に即座かつ効率的なデータ入力を可能とする。

- クイックアクションボタン

画面上の戦略的に配置された複数のクイックアクションボタンにより、新しい修理オーダーの作成、予約カレンダーへのアクセス、顧客履歴の表示など、その他の主要機能へワンタップでアクセスできますこれらのショートカットは、ナビゲーション手順を最小化することでユーザーのワークフローを効率化するよう設計。

- ナンバープレートおよび QR コードスキャンオプション

車両識別プロセスをさらに高速化するため、インターフェースにはデバイスのカメラを使用したスキャンオプションが含まれている。

ユーザーは車両のナンバープレートを直接スキャンすることも、QR コードをスキャンすることもでき、手動入力に対する柔軟でエラーを減らす代替手段を提供。
<img width="356" height="247" alt="image" src="https://github.com/user-attachments/assets/5f46b39f-0379-4483-969c-897d11b1727c" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両 | 現在選択中の車両を表示 |
| 2 | 車両を選択してください | ヘッダーの案内テキスト |
| 3 | ショッピングカートアイコン | 販売カートへのアクセス |
| 左メニュー |  |  |
| 4 | 顧客検索 | 顧客検索画面を開くボタン |
| 5 | 車両詳細検索 | 車両詳細検索画面を開くボタン |
| 6 | 携行缶詰替え販売 | 携行缶・詰替販売のボタン |
| 中央（入力） |  |  |
| 7 | ひらがな（任意） | 入力ラベル |
| 8 | 青丸表示 | 入力中の文字・数字を表示 |
| 9 | QRスキャンアイコン | QRコード読み取りボタン |
| 10 | B-12スキャンアイコン | 車両ナンバープレートスキャンボタン |
| 11 | Keypad 0–9 | 4桁車番入力ボタン |
| 12 | CLEAR | 入力を全消去するボタン |
| 13 | → DEL | 最後の1文字を削除（バックスペース） |
| 14 | 検索 | 検索を実行するボタン |
| 右メニュー |  |  |
| 15 | カーメンテ販売 | ナンバー未登録車向けのメンテ販売ボタン |
| 16 | 安全点検 | ナンバー未登録車向けの安全点検ボタン |
| 17 | 預かり・代車・パーツ | 預かり、代車、パーツ発注ボタン |
| フッターツールバー |  |  |
| 18 | 自店舗カレンダー | 自店舗カレンダーを表示 |
| 19 | リペアカレンダー | リペアカレンダーを表示 |
| 20 | 自店舗アクションリスト | 自店舗のアクションリストを表示 |
| 21 | お知らせ | お知らせ画面を表示するボタン |
| 22 | （ギアアイコン） / 環境 | 設定・環境画面を開くボタン |

### 14.1.3. 顧客/車両情報検索画面
- QR スキャンによる多用途検索

システムは、ナンバープレートの手動入力と、統合された QR コードスキャナーの両方をサポートする強力な検索機能を提供します。この二重の機能により柔軟性が確保され、効率性が向上し、ユーザーは車両の記録を迅速かつ正確に特定でき、手動入力による潜在的なエラーを最小限に抑える。

- 統合された検索結果

検索が成功すると、アプリケーションは包括的な結果画面を表示し、顧客プロファイルと対応する車両情報を明確に関連付けて表示。

この統一されたビューにより、ユーザーは顧客名や連絡先情報といった基本情報に加え、車両のメーカー、モデル、サービス履歴といった重要な詳細にも即座にアクセスでき、必要情報を一目で把握。

- 新規車両登録

インターフェースには、システムに新しい車両を登録するためのシンプルで直感的なワークフローが含まれる。

この機能は、新規顧客のオンボーディングや既存顧客のプロファイルに追加車両を登録する際に不可欠であり、データベースが最新かつ完全な状態に保たれることを保証。
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/f74638c5-4bf3-4972-9a85-b6888ce8fec5" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両 | 現在選択中の車両を表示する |
| 2 | 車両を選択してください | ヘッダーの案内テキスト |
| 3 | ショッピングカートアイコン | 販売カートにアクセスする |
| 4 | 戻るアイコン | 前の画面へ戻る |
| 5 | ホームアイコン | ホーム画面へ移動する |
| 検索パネル（左側） |  |  |
| 6 | 絞り込み検索 | 検索フィルタパネルのタイトル |
| 7 | 顧客氏名 | 顧客の氏名入力フィールド |
| 8 | 顧客カナ　| 顧客名（カタカナ）入力フィールド |
| 9 | 地名 | 住所／ナンバー地名などの入力フィールド |
| 10 | ひらがな／アルファベット | 文字入力フィールド |
| 11 | 4桁車番 | ナンバー4桁の入力フィールド |
| 12 | メーカー | 車両メーカー入力フィールド |
| 13 | クリア | 検索条件を全てクリアするボタン |
| 検索結果テーブル（メインエリア） |  |  |
| 14 | メーカー | 車両メーカー（列） |
| 15 | 車種名 | 車両モデル名（列） |
| 16 | 色 | 車両カラー（列） |
| 17 | ナンバープレート | ナンバープレート番号（列） |
| 18 | 顧客名 | 顧客名（列） |
| 19 | 電話番号 | 顧客電話番号（列） |
| 20 | QR | QRコード識別子（列） |
| 21 | ステータス | 車両／顧客ステータス（列） |
| フッター（メインエリア） |  |  |
| 22 | 車両の登録 | 新規車両を登録するボタン |
| フッター・ツールバー |  |  |
| 23 | 自店舗カレンダー | 自店舗カレンダーを表示 |
| 24 | リペアカレンダー | リペアカレンダーを表示 |
| 25 | 自店舗アクションリスト | 自店舗のアクションリストを表示 |
| 26 | お知らせ | お知らせ一覧を表示 |
| 27 | 環境（ギアアイコン） | 設定・環境画面にアクセス |

### 14.1.4. ステータス画面
- 統合された To-Do リスト管理

この画面には、車両メンテナンス専用のタスク管理システムが統合される。

サービススタッフは、車両に必要な作業の To-Do リストを作成、追跡、管理でき、すべてのメンテナンス作業が完了し、サービス記録の一部として適切に文書化されることを保証。

- リマインダー作成および追跡

システムには、将来のフォローアップサービスのためのリマインダーを設定および監視する機能が含まれる。

このプロアクティブなツールは、車両が定期メンテナンスや予定点検の時期に到達した際に自動的にフラグを立て、車両所有者とのタイムリーな連絡を可能にすることで、顧客維持に貢献。

- 顧客情報の編集

使いやすさを最大化するため、このインターフェースでは履歴画面から直接顧客情報を編集できる。

スタッフは別画面に移動することなく、連絡先情報やその他の個人データを迅速に更新でき、顧客記録が常に正確かつ最新の状態に保たれる。

- 車両データの編集

この画面では車両データの完全な管理が可能であり、走行距離、登録情報、技術仕様などの主要情報を変更・更新できます。これにより、車両プロファイルがサービス期間を通じて常に最新の状態で維持される。

- 包括的なサービス履歴の参照

この画面の中核要素は、詳細なサービス履歴ログ。

過去に実施されたすべての作業が網羅的に記録されており、交換部品や作業項目の一覧、技術者の記録、サービス日付などを含む。

この容易にアクセス可能な情報は、現在の問題の診断や一貫したサービス品質の維持に極めて重要。
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/6273a3eb-b3d2-430d-b347-991455edb63b" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両（三菱 300…） | 選択中の車両(ナンバー・メーカー等)を表示する |
| 2 | e-NV200 バネット | 車両のメーカー／車種名を表示する |
| 3 | ショッピングカートアイコン | 販売カートにアクセスする |
| 4 | 戻るアイコン | 前の画面へ戻る |
| 5 | ホームアイコン | ホーム画面へ移動する |
| メインヘッダー・サイドナビ |  |  |
| 6 | 車両／顧客ステータス | 画面タイトル |
| 7 | ・三菱 300・わ 56-70 | 選択車両のナンバープレートを大きく表示 |
| 8 | 写真：○　| 写真有無の表示 |
| 9 | To Do リスト | To Do リストを表示するタブ（選択中） |
| 10 | 声掛けリスト（商材別） | 商材別の声掛けリスト表示タブ |
| 11 | アクション履歴（商材別） | 商材別アクション履歴タブ |
| 12 | 購入履歴 | 購入履歴表示タブ |
| 13 | アンケート履歴 | アンケート履歴タブ |
| 情報パネル（上部） |  |  |
| 14 | 車両情報（パネル） | 車両に関する基本情報パネル |
| 15 | メーカー（e-NV200…） | 車両メーカー／車種名を表示 |
| 16 | 車検満了日(2025年9月02日) | 車検満了日を表示 |
| 17 | 車検証（○） | 車検証の確認状況（チェック表示） |
| 18 | 顧客情報（パネル） | 顧客情報パネル |
| 19 | 顧客名（ゼ テスト…） | 顧客氏名を表示 |
| 20 | 情報／連絡ラジオボタン | 住所・電話番号・支払方法の切替表示 |
| 21 | 予約状況（パネル） | 直近の予約情報を表示 |
| 22 | 接点履歴（パネル） | 直近の接点（コンタクト）履歴を表示 |
| 23 | 詳細（ボタン） | パネル内容の詳細を表示 |
| To Do リスト（メインテーブル |  |  |
| 24 | 項目（列） | To Do 項目名を表示する列 |
| 25 | 進捗（列） | 進捗ステータス（×／○／△等）を表示 |
| 26 | メモ（列） | 関連メモ・日付・計測値を表示 |
| 27 | 更新（ボタン） | 対象アイテムの進捗を更新するボタン |
| 28 | 声掛けリスト（行例） | 声掛け項目の一例 |
| 29 | エアチェック（行例） | 空気圧チェック項目の一例 |
| 30 | 安全点検（行例） | 安全点検項目の一例 |
| 下部アクションバー |  |  |
| 31 | 安全点検（ボタン） | 新規安全点検を開始するボタン |
| 32 | カーメンテ販売（ボタン） | メンテナンス販売を開始するボタン |
| 33 | 車検（ボタン） | 車検サービス開始ボタン |
| 34 | 適合商品リスト（ボタン） | 適合する商品一覧を表示 |
| 35 | 預かり・代車・パーツ（ボタン） | 預かり・代車・パーツ発注のボタン |
| 36 | 履歴登録（ボタン） | 新規履歴・接点情報を登録するボタン |
| フッター・ツールバー |  |  |
| 37 | 自店舗カレンダー | 自店舗カレンダーを表示 |
| 38 | リペアカレンダー | リペアカレンダーを表示 |
| 39 | 自店舗アクションリスト | 自店舗のアクションリストを表示 |
| 40 | お知らせ | お知らせ一覧を表示 |
| 41 | 環境（ギアアイコン） | 設定・環境画面にアクセス |

### 14.1.5. 商品閲覧・選択画面
- 分類された商品ブラウジング

この画面には、論理的なカテゴリにグループ化された、よく整理された商品カタログが表示される。

この構造化されたアプローチにより、ユーザーは在庫を効率的にナビゲートし、特定の部品や商品を迅速に見つけることができる。

これは販売プロセスの速度と容易さを大幅に向上させる。

さらに、検索を絞り込むためのフィルタリングシステムも備える。

- 商品詳細

リスト内の各商品は選択可能で、詳細ページを表示できる。

このページには、高品質の画像、明確な価格情報、およびその他の関連仕様が含まれる。

これにより、サービススタッフは必要な情報を迅速に確認でき、顧客に対して正確な商品説明を行うための十分な情報を得ることができる。

- 統合されたカート追加機能

インターフェースには、商品の希望数量を選択し、それを現在の注文または販売カートに直接追加するための直感的な仕組みが用意されている。

この POS（Point-of-Sale）ワークフローへのシームレスな統合により、商品選択から最終的なチェックアウトまで、顧客の請求書作成プロセスが効率化される。
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/c0c7e727-8113-4a70-802a-f0de02340aa7" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両 | 現在選択中の車両を表示する |
| 2 | 車両を選択してください | ヘッダーの案内タイトル |
| 3 | ショッピングカートアイコン | 販売カートにアクセスする |
| 4 | 戻るアイコン | 前の画面へ戻る |
| 5 | ホームアイコン | ホーム画面へ移動する |
| 商品・サービスグリッド |  |  |
| 6 | オイル関連 | オイル商品・サービスのボタン |
| 7 | タイヤ | タイヤ関連サービスのボタン |
| 8 | バッテリー　| バッテリー商品・サービスのボタン |
| 9 | 洗車 | 洗車サービスのボタン |
| 10 | コーティング | コーティングサービスのボタン |
| 11 | エンジン清浄剤 | エンジン清浄剤のボタン |
| 12 | ワイパー | ワイパーブレードのボタン |
| 13 | ACフィルター | エアコンフィルターのボタン |
| 14 | その他商品 | その他の商品ボタン |
| 15 | その他車検部品 | その他の車検部品ボタン |
| 16 | リペア | リペア（傷・凹み等）サービスのボタン |
| 17 | 車両販売 | 車両販売メニューのボタン |
| フッター（メインエリア） |  |  |
| 18 | カートへ進む | ショッピングカート画面へ進むボタン |
| フッター・ツールバー |  |  |
| 19 | 自店舗カレンダー | 自店舗カレンダーを表示 |
| 20 | リペアカレンダー | リペアカレンダーを表示 |
| 21 | 自店舗アクションリスト | 自店舗のアクションリストを表示 |
| 22 | お知らせ | お知らせ一覧を表示 |
| 23 | 環境（ギアアイコン） | 設定・環境画面にアクセス |

### 14.1.6. 車検画面
- ガイド付きのマルチステップ点検ワークフロー

アプリケーションには、技術者を車両点検プロセス全体へ案内する構造化された複数ステップのワークフローが備わっており、事前点検（Pre-inspection）から重大な問題に対する詳細評価（Heavy Repair）まで、必要なすべてのチェックポイントを網羅し、一貫性のある徹底した点検が実施されることが保証される。

- ステータスインジケーター付きインタラクティブチェックリスト

この画面の中心となるのは、点検項目のインタラクティブなチェックリストです各項目には「合格」「不合格」「要注意」などの明確なステータスインジケーターが付いており、技術者は車両のさまざまなコンポーネントの状態を視覚的かつ効率的に記録できますこの仕組みにより、車両全体の状態を一目で把握できる概要が提供される。

- 自動化された部品およびサービスの推奨

点検結果に基づき、システムは必要な部品およびサービスの推奨リストを自動生成する。

「要注意」とマークされた項目について、アプリケーションは適切な交換部品または修理内容を提案することができ、顧客向けの正確なサービス見積作成を効率化し、診断から解決までのプロセスをスムーズに橋渡しする。
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/cdaa0727-8d2e-40b6-b3a9-fc1fb591638a" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両（例：三菱 300…） | 現在選択中の車両の主要情報（ナンバー・メーカー等）を表示する |
| 2 | e-NV200 バネット | 選択車両の車種名を表示するテキスト |
| 3 | ショッピングカートアイコン | 販売カートにアクセスする |
| 4 | インフォアイコン | 追加情報を表示するボタン |
| 5 | ホームアイコン | ホーム画面へ移動する |
| ページタイトル・タブ |  |  |
| 6 | 車検 受付 | 車検受付画面のメインタイトル |
| 7 | Step 2 事前点検の実施 | 現在のステップを示すサブタイトル |
| 8 | 室内・外廻り（タブ）　| 室内／外装点検タブ（選択状態） |
| 9 | タイヤ（タブ） | タイヤ点検タブ |
| 10 | 足廻り（タブ） | 足廻り点検タブ |
| 11 | 下廻り（タブ） | 下廻り点検タブ |
| 12 | エンジンルーム（タブ） | エンジンルーム点検タブ |
| 13 | その他整備（タブ） | その他整備タブ |
| 左テーブル (点検チェックリスト) |  |  |
| 14 | 場所（列） | 点検箇所の大分類を表示する列 |
| 15 | 総合判定（列） | 点検箇所の総合判定（OK／NG）を表示する列 |
| 16 | 点検項目一覧 | 警告灯・ランプ・ホーン・ワイパー等の点検項目 |
| 右テーブル (詳細パネル) |  |  |
| 17 | メモを入力（ボタン） | メモ入力画面を開くボタン |
| 18 | 全て「―」 | サブ項目の結果をすべて「―（対象外）」に設定する |
| 19 | 全て「〇」 | サブ項目の結果をすべて「〇（良好）」に設定する |
| 20 | 点検内容（列） | 点検対象内容（例：エラーメッセージ）を表示 |
| 21 | 結果（列） | 点検結果（〇／不適 等）を表示 |
| 下部アクションバー |  |  |
| 22 | 戻る | 前の画面へ戻る |
| 23 | 保存（車検TOPへ） | 入力内容を保存し、車検TOPへ戻る |
| 24 | 「事前点検報告書」発行 | 事前点検報告書を発行する |
| 25 | Step 3 へ進む | 次のステップへ進む |
| フッターツールバー |  |  |
| 26 | 自店舗カレンダー | 自店舗カレンダーを表示する |
| 27 | リペアカレンダー | リペアカレンダーを表示する |
| 28 | 自店舗アクションリスト | 自店舗アクションリストを表示する |
| 29 | お知らせ | お知らせ一覧を表示する |
| 30 | 環境（ギアアイコン） | 設定／環境画面を表示する |

### 14.1.7. カート画面
- 詳細なラインアイテム内訳

この画面は、注文に含まれるすべての商品およびサービスの明確で項目化された一覧を提示。

各ラインアイテムには、名称、選択数量、個別価格、およびその行の小計がはっきりと記載されており、スタッフと顧客の双方に対して完全な透明性を提供する。

- 税を含む自動合計計算

システムは、すべてのラインアイテムを合計することで最終コストを自動的に計算する。

その後、該当する税金や Add On Fee といった追加料金を適用し、総合計を算出しますこの自動化された処理により、手動計算によるエラーのリスクが最小化され、請求すべき最終金額が正確に提供される。

- 注文確定および書類生成

サマリーを確認し承認した後、ユーザーは注文を確定でき、これはシステム上でトランザクションを完了させる。

その後、インターフェースは必要な書類を生成するためのオプションを提供。

例：
- 顧客向けに物理的なレシートを印刷
- 会計や台帳用途のため、PDF などのデジタル記録を作成
<img width="359" height="249" alt="image" src="https://github.com/user-attachments/assets/d906b850-e10a-4e5a-af77-08c709ca3505" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 選択中の車両 | 現在選択中の車両を表示する |
| 2 | 車両を選択してください | 車両未選択時のヘッダータイトル |
| 3 | ショッピングカートアイコン | 販売カートにアクセスする |
| 4 | インフォアイコン | 追加情報を表示するボタン |
| 5 | ホームアイコン | ホーム画面へ移動する |
| メインエリア (左：合計・商品一覧) |  |  |
| 6 | 見積作成中 | ページのステータス／タイトル |
| 7 | 進捗状況 | 現在の進捗状況ラベル |
| 8 | 合計（税込）　| 税込の合計金額を表示 |
| 9 | （内）消費税 | 合計に含まれる消費税額を表示 |
| 10 | 全てのチェックを外す | 全商品のチェックを解除するチェックボックス |
| 11 | 商品テーブル | 商品名／単価／数量／金額などの一覧 |
| 12 | 小計 | 商品小計金額を表示 |
| メインエリア (右：情報・操作) |  |  |
| 13 | 車両／顧客情報パネル | 顧客名・電話番号・住所・走行距離などを表示 |
| 14 | 予約カレンダーパネル | 来店日・納車日の予約情報を表示 |
| 15 | 来店日／納車日 | 予約日ラベル |
| 16 | 選択ボタン | カレンダーを開いて日付を選択する |
| 17 | 見積り商品／工賃の追加パネル | 追加項目の種類をまとめたパネル |
| 18 | 登録商品ボタン | 登録済み商品を追加する |
| 19 | フリー入力ボタン | 任意の商品・工賃を手入力で追加する |
| 20 | 点検商品修正ボタン | 点検関連商品の修正を行う |
| 21 | その他パネル | 備考・割引などの追加操作 |
| 22 | 備考追加ボタン | 備考を追加する |
| 23 | 合計金額値引きボタン | 合計金額に対する値引きを適用する |
| 24 | アクションパネル | 見積保存・予約確定などの操作領域 |
| 25 | 見積保存（御見積書） | 見積を保存し御見積書を発行する |
| 26 | 予約確定（御見積書） | 予約を確定し御見積書を発行する |
| 27 | 作業開始（作業依頼書） | 作業を開始し作業依頼書を発行する |
| 28 | 作業完了（納品請求書） | 作業完了後納品請求書を発行する |
| フッターツールバー |  |  |
| 26 | 自店舗カレンダー | 自店舗カレンダーを表示する |
| 27 | リペアカレンダー | リペアカレンダーを表示する |
| 28 | 自店舗アクションリスト | 自店舗アクションリストを表示する |
| 29 | お知らせ | お知らせ一覧を表示する |
| 30 | 環境（ギアアイコン） | 設定／環境画面を表示する |

### 14.1.8. 車検オーダー タイプ選択画面
- 4種類の点検タイプカード：画面は4つの大きなタップ可能なカードを表示し、3項目安全点検・8項目安全点検・15項目安全点検・リース点検の各タイプを選択できる。各カードには点検スコープを視覚的に伝えるアイコンが付いており、担当技術者が一目で正しい選択肢を識別できる。
- ヘッダーに車両コンテキストを表示：現在選択中の車両がトップヘッダーバー（車両を選択してください）に表示され、どの車両に対して新しい車検オーダーが紐付けられるかをユーザーに確認させる。ショッピングカートアイコンもクロスフローナビゲーション用にアクセス可能。
- ワンタップタイプ選択：4つのカードのいずれかをタップすると即座にそのタイプの車検オーダーが作成され、対応する点検フォーム（セクション 14.1.9）へ遷移する。画面下部の「戻る」ボタンはオーダーを作成せずに前の画面に戻る。

| No. | フィールド名（原文） | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | ETR-CARS / ENEOS 達 navi | ブランディング：ヘッダー左上に表示されるロゴ |
| 2 | 選択中の車両 | 選択中の車両ラベル：車検オーダーを作成する対象車両を示す |
| 3 | 車両を選択してください | 車両未選択時のプロンプト：タップで車両選択を開く |
| 4 | （ショッピングカートアイコン） | ショッピングカート：現在の車両の販売カートにアクセスする |
| 5 | （ホームアイコン） | ホームボタン：iPad ホーム画面へ移動する |
| 点検タイプカード |  |  |
| 6 | 3項目安全点検 | 3項目安全点検カード：新規3項目点検オーダーを開始する |
| 7 | 8項目安全点検 | 8項目安全点検カード：新規8項目点検オーダーを開始する |
| 8 | 15項目安全点検 | 15項目安全点検カード：新規15項目点検オーダーを開始する |
| 9 | リース点検 | リース点検カード：新規リース点検オーダーを開始する |
| ボトムアクション |  |  |
| 10 | 戻る | 戻るボタン：オーダーを作成せずに前の画面に戻る |
| フッターツールバー |  |  |
| 11 | 自店舗カレンダー | フッターボタン：ショップのカレンダーを開く |
| 12 | リペアカレンダー | フッターボタン：リペアカレンダーを開く |
| 13 | 自店舗アクションリスト（未成約・予約済・作業中・納車済） | フッターボタン：未成約・予約済・作業中・納車済オーダー一覧を表示する |
| 14 | お知らせ | フッターボタン：システムのお知らせを表示する |
| 15 | 環境 | フッターボタン：環境設定画面にアクセスする |

### 14.1.9. 車検 点検フォーム画面
- 判定凡例ヘッダー：フォーム上部にすべての点検項目に対する4種類の判定結果を表示する。○（OK）は現時点で交換・補充不要、△（注意）は早めの交換・補充が必要、×（NG）は安全のため即時交換・補充が必要、—（未実施）は今回点検を実施しなかった項目を表す。凡例はフォーム上部に固定されており、技術者が記入中も常に判定基準を確認できる。
- 点検項目一覧（事前定義）：凡例の下に、選択した点検タイプ（3項目・8項目・15項目・リース）に応じた事前定義の点検項目リストを表示する。項目例：ブレーキパッド、ブレーキライニング、各種ブーツ、マフラー、タイヤ、ブレーキホース、エンジンオイル、オイルフィルターなど。各項目の左列に名称、右の点検結果列に基準値（例：残厚・走行距離閾値・年数閾値）を表示し、技術者が判断の基準を確認できる。
- 項目ごとのワンタップ判定選択：各行に4つの判定アイコン（○ / △ / × / —）がタップ対象として表示される。技術者は各項目につき1つを選択する。備考欄（備考）には追加情報（観察された損傷や推奨するフォローアップサービスなど）を自由テキストで入力できる。
- 保存アクション：右下の「保存」ボタンにより点検結果を現在の車検オーダーに確定する。技術者が項目リストをスクロールしている間も常に表示され、いつでも途中保存できる。

| No. | フィールド名（原文） | 説明 |
|----|----|----|
| ヘッダー |  |  |
| 1 | 15項目安全点検（例） | フォームタイトル：現在記入中の点検タイプを示す |
| 判定凡例 |  |  |
| 2 | 結果の見方 | 判定凡例ラベル：評価記号の凡例を示す見出し |
| 3 | ○ / OK! | 判定記号 OK：現時点で交換・補充の必要なし |
| 4 | △ / 注意 | 判定記号 注意：早めの交換・補充を推奨 |
| 5 | × / NG | 判定記号 NG：安全のため即時交換・補充が必要 |
| 6 | — / 未実施 | 判定記号 未実施：今回は点検を実施していない |
| 点検項目テーブル |  |  |
| 7 | 点検項目 | 点検項目列ヘッダー：点検対象の項目名 |
| 8 | 点検結果 | 点検結果列ヘッダー：基準値と4つの選択可能な判定アイコン |
| 9 | 備考 | 備考列ヘッダー：メモや観察事項などの自由入力欄 |
| 10 | ブレーキパッド（ディスクブレーキ） | 点検項目：残厚（例：3 mm）が基準値として表示される |
| 11 | ブレーキライニング（ドラムブレーキ） | 点検項目：ドラムブレーキのライニング残厚を確認する |
| ボトムアクション |  |  |
| 12 | 保存 | 保存ボタン：点検結果を現在の車検オーダーに確定する |

## 14.2. Webアプリケーション UI
### 14.2.1. ダッシュボード
- 権限コントロール

ロールベースのモジュールアクセスユーザーは、割り当てられたロールに基づいて異なるモジュールやデータを閲覧できる。

例えば、管理者（Administrator）はすべてを閲覧できるが、一般スタッフは職務に関連する機能のみを閲覧できる。

- 主要機能へのクイックアクセスボタン

よく使用される機能へのショートカットを提供し、ワークフローを高速化。

- サービス、タブレットユーザー、店舗、注文履歴へのナビゲーション

サービス管理、タブレットユーザー管理、店舗管理、および注文履歴の閲覧を行うためのナビゲーションリンクを含む。

権限を持つロールのみが顧客管理へアクセス顧客管理セクションには、適切な権限を持つユーザーのみアクセスできる。
<img width="438" height="251" alt="image" src="https://github.com/user-attachments/assets/5941c38f-bccc-4d86-b133-0f9ec16ca14d" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダーバー |  |  |
| 1 | 日本語（ドロップダウン） | 表示言語を変更するドロップダウン |
| 2 | エネオス / ENEOSトレーディング株式会社 | 現在の会社／グループ名を表示 |
| 3 | cta-admin / ログアウト | ログインユーザー名とログアウトリンクを表示 |
| サイドナビメニュー |  |  |
| 4 | 管理メニュー（カテゴリ） | 管理系機能のカテゴリ（折りたたみ可能） |
| 5 | 販促・データ抽出（カテゴリ） | 販促・データ抽出系カテゴリ（折りたたみ可能） |
| 6 | 分析（カテゴリ） | 分析機能カテゴリ（折りたたみ可能） |
| 主な内容(ダッシュボードアイコン) |  |  |
| 7 | 商品一括編集 | タブレットアプリで表示される商品の一括管理 |
| 8 | アプリユーザー　| タブレットアプリ用ユーザーの作成・更新・削除 |
| 9 | 台数 | 車両／ユニット管理画面への遷移（推定） |
| 10 | SS基本情報設定 | SS（サービスステーション）の基本情報管理 |
| 11 | 受注履歴 | 特定注文の状況や詳細を確認 |
| 12 | 顧客 | 顧客管理画面へのアクセス |

### 14.2.2. ユーザー管理インターフェース
- フィルタリングと検索を備えたユーザー一覧

すべてのユーザーの一覧を表示し、検索およびフィルタリングのオプションを提供。

- ロール割り当て（管理者、ディーラーマネージャー、店舗マネージャー、店舗ユーザー）

管理者は、ユーザーに管理者、ディーラーマネージャー、支店（店舗）マネージャー、店舗ユーザーなどの特定のロールおよび権限を割り当てることができる。

- アカウント作成時のメールフィールド検証

新規アカウント作成時、メールアドレス欄には形式が正しいかどうかの検証が含まれる。

- 権限コントロールを伴うユーザープロファイル編集

ユーザープロファイルの編集が可能であるが、ログイン中のユーザーが必要な権限を持っている場合に限られる。
<img width="438" height="251" alt="image" src="https://github.com/user-attachments/assets/263df62c-20dc-48a6-80f2-93699a034f92" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダーバー |  |  |
| 1 | 日本語（ドロップダウン） | 表示言語を変更するドロップダウン |
| 2 | エネオス / ENEOSトレーディング株式会社 | 現在の会社／グループ名を表示 |
| 3 | cta-admin / ログアウト | ログインユーザー名とログアウトリンクを表示 |
| サイドナビメニュー |  |  |
| 4 | 設定 | メインカテゴリ（折りたたみ可能） |
| 5 | SS情報設定 | SS情報設定メニュー |
| 6 | アプリユーザー | アプリユーザー管理メニュー |
| 7 | WEBユーザー（選択中） | WEBユーザー管理メニュー |
| 8 | SS基本情報設定　| SS基本情報設定メニュー |
| 9 | アンケート作成 | アンケート作成メニュー |
| 10 | アンケートメニュー | アンケート管理メニュー |
| 11 | ToDoリスト | ToDoリストメニュー |
| 12 | 声掛けリスト | 声掛けリストメニュー |
| 13 | その他メニュー | データ取込、マスター設定、商品情報などの各種メニュー |
| メインコンテンツエリア |  |  |
| 14 | WEBユーザー（タイトル） | 画面タイトル |
| 15 | メールを送信（ボタン） | メール送信ボタン |
| 16 | ユーザーを追加（ボタン） | ユーザー追加ボタン |
| ユーザー一覧テーブル列 |  |  |
| 17 | ID | ユーザーの一意のID |
| 18 | ユーザー名 | ログインユーザー名 |
| 19 | 役職 | ユーザーの役職・権限（例：店長） |
| 20 | 特約店 | 所属する特約店 |
| 21 | SS | 担当するサービスステーション |
| 22 | 編集 | ユーザー情報編集操作 |
| 23 | PWリセット | パスワードリセット操作 |
| 24 | 削除 | ユーザー削除操作 |
| ユーザーテーブル行アクション |  |  |
| 25 | 編集アイコン | 対象ユーザーの編集ボタン |
| 26 | 削除アイコン | 対象ユーザーの削除ボタン |

### 14.2.3. 顧客管理インターフェース
- 複数フィルターによる高度な検索

名前、日付、店舗、車のメーカー／モデルなど、さまざまな条件で顧客をフィルタリングできる強力な検索機能を備える。

- 完全な履歴および連絡先情報を含む顧客プロファイル

各顧客の詳細ビューを提供し、完全なサービス履歴および連絡先情報を確認できる。

- 顧客データのインポート／エクスポート機能

ファイルからの顧客データの一括インポート、およびすべての顧客データのファイルへのエクスポートをサポート。
<img width="438" height="251" alt="image" src="https://github.com/user-attachments/assets/518a7522-111b-4e13-bc56-6d4b2a0e1b2f" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダーバー | | |
| 1 | 日本語（ドロップダウン） | 表示言語を変更するドロップダウン |
| 2 | エネオス / ENEOSトレーディング株式会社 | 現在の会社／グループ名を表示 |
| 3 | cta-admin / ログアウト | ログインユーザー（cta-admin）表示およびログアウトリンク |
| サイドナビメニュー | | |
| 4 | マスター設定（カテゴリ） | 選択中メニューの親カテゴリ |
| 5 | 商品一括編集 | 商品一括管理メニュー |
| 6 | 商品個別編集 | 商品個別編集メニュー |
| 7 | 顧客（選択中） | 現在の選択メニュー |
| メインコンテンツエリア | | |
| 8 | 顧客（タイトル） | 画面タイトル |
| 9 | エクスポート（ボタン） | 顧客データをエクスポートするボタン |
| 10 | 検索（パネル） | 検索フィルタパネル |
| 検索フィルタ | | |
| 11 | 漢字氏名（全角） | 顧客の漢字氏名で絞り込み |
| 12 | カナ氏名（半角） | 顧客のカナ氏名で絞り込み |
| 13 | 受注なし | 受注なし顧客のみを表示 |
| 14 | 顧客作成日 | 作成日の範囲（From / To） |
| 15 | 最新更新日 | 更新日の範囲（From / To） |
| 16 | 最新受注日 | 受注日の範囲（From / To） |
| 17 | ショップ | ショップ選択フィルタ |
| 18 | 車両メーカー | メーカー選択フィルタ |
| 19 | 車種名 | 車種フィルタ |
| 20 | 4桁車番（セクション） | ナンバー4桁入力セクション |
| 21 | 陸運局名 | 地域（運輸局）選択 |
| 22 | 分類番号 | 分類番号入力 |
| 23 | ひらがな | ナンバーのひらがな入力 |
| 24 | 4桁車番 | 4桁ナンバー入力 |
| 25 | リセット（ボタン） | すべてのフィルタをクリア |
| 26 | 検索（ボタン） | フィルタを適用して検索実行 |
| 検索結果テーブル | | |
| 27 | 顧客名 | 顧客名 |
| 28 | 顧客氏名 | 顧客フルネーム |
| 29 | QR | QRコード情報 |
| 30 | プライバシーポリシー同意済 | お客様情報等の取り扱いの同意状況 |
| 31 | 都道府県 | 住所の都道府県 |
| 32 | 市区町村 | 住所の市区町村 |

### 14.2.4. カーメンテ管理インターフェース
- 種類別のサービスカタログ構成

サービスがカテゴリごとに整理されており、検索および管理が容易となる。

- サービスデータのインポート／エクスポート機能

ファイルを通じてサービス情報を一括インポートおよびエクスポートできる。

- ステータス追跡付きの非同期エクスポート

大量データのエクスポートはバックグラウンドプロセスとして実行されるため、ユーザーは待つ必要がありませんユーザーはエクスポートジョブの進行状況を確認できる。
<img width="438" height="251" alt="image" src="https://github.com/user-attachments/assets/98cc7b05-ec18-4744-bc1b-2cc356c5eeed" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダーバー | | |
| 1 | 日本語（ドロップダウン） | 表示言語を変更するドロップダウン |
| 2 | エネオス / ENEOSトレーディング株式会社 | 現在の会社／グループ名を表示 |
| 3 | cta-admin / ログアウト | ログインユーザー（cta-admin）とログアウトリンク |
| サイドナビメニュー | | |
| 4 | マスター設定（カテゴリ） | 選択項目の親カテゴリ |
| 5 | 商品一括編集（選択中） | 現在開いているメニュー |
| 6 | 商品個別編集 | 商品個別編集メニュー |
| 7 | 顧客 | 顧客管理メニュー |
| 8 | その他メニュー | インポート/エクスポート、工場メニュー等 |
| メインエリア | | |
| 9 | 商品一括編集（タイトル） | 画面タイトル |
| 10 | ショップを選択（Dropdown） | ショップを選択するドロップダウンフィルタ |
| 11 | 「商品・サービス」を追加（ボタン） | 新規商品・サービスを追加するボタン |
| 12 | 商品マスターをエクスポート（ボタン） | 商品マスター一覧をエクスポートするボタン |
| 13 | 商品／サービス一覧（グリッド） | 管理対象の商品・サービス（例：オイル、点検、バッテリー、洗車）を表示 |

### 14.2.5. 分析インターフェース
- フィルタリングオプション付きの販売実績レポート

ユーザーは販売レポートを閲覧し、さまざまなフィルターを適用してパフォーマンスを分析できる。

- サービス利用統計

各サービスがどの程度の頻度で利用されているかを示す統計情報を表示。

- すべてのレポートのエクスポート機能

生成されたすべてのレポートをファイルとしてダウンロードできる。
<img width="438" height="251" alt="image" src="https://github.com/user-attachments/assets/00b857c6-1e47-4ec1-af44-d4b0969e396e" />

| No. | フィールド名 | 説明 |
|----|----|----|
| ヘッダーバー | | |
| 1 | 日本語（Dropdown） | 表示言語を変更するドロップダウン |
| 2 | エネオス / ENEOSトレーディング株式会社 | 現在の会社／グループを表示 |
| 3 | cta-admin（Dropdown） | ログイン中の管理者を表示するユーザーメニュー（ドロップダウン） |
| サイドナビメニュー | | |
| 4 | 顧客実績分析（カテゴリ） | 選択中メニューの親カテゴリ |
| 5 | 顧客データ（累計実績確認）（メニュー） | 現在選択中のメニュー |
| メインエリア | | |
| 6 | 見積データ（累計実績確認）（タイトル） | メインコンテンツのタイトル |
| 7 | 検索（パネル） | 折りたたみ可能な検索条件パネル |
| 8 | 登録日（フィルタ） | 登録日の期間指定（From / To） |
| 9 | SS名（フィルタ） | SS（サービスステーション）選択ドロップダウン（例：All SS） |
| 10 | 商品カテゴリー（フィルタ） | 商品カテゴリの絞り込み（例：オイル関連） |
| 11 | リセット（ボタン） | 検索条件を全てクリアする |
| 12 | 検索（ボタン） | 条件を適用して検索を実行する |
| 13 | エクスポート（ボタン） | 表示結果データをエクスポートする |
| データテーブル表 | | |
| 14 | 見積数（列グループ） | 作成された見積の集計指標 |
| 15 | 成約（列グループ） | 成約（成立）した指標の集計 |
| 16 | 成約率（列グループ） | 成約率（割合）の指標 |
| サブ列（各グループ内の共通サブ列） | | |
| 17 | 台数（サブ列） | 車両の台数 |
| 18 | 件数（サブ列） | 件数（アイテム／ケース数） |
| 19 | 単価/1件（サブ列） | 1件あたりの平均単価 |
| 20 | 金額（サブ列） | 合計金額 |

## 14.3. 画面遷移図
ユーザーがアプリケーション内をどのように操作・遷移するかを示す参考資料として、ユーザーインターフェースの画面遷移図を以下。
<img width="452" height="596" alt="image" src="https://github.com/user-attachments/assets/1dde95b4-13d0-49a9-9605-bf9d20ccaf41" />

# 15. データベース構造
ETRCARS システムでは、PostgreSQL をデータベースとして採用しており、主要なデータ構造は以下のとおり。：

## 15.1. テーブル構成
システムで使用するデータベースのテーブル構成について定義する。
<img width="451" height="387" alt="image" src="https://github.com/user-attachments/assets/a95ea259-13fd-4722-89f2-e0e732d66fa6" />

## 15.2. テーブル構造
| テーブルNo. | テーブル名(物理名) | テーブル名(論理名) |
|----|----|----|
| 1 | branches\_branch | 支店 |
| 2 | dealers\_dealer | ディーラー |
| 3 | services\_service | 商品カテゴリー |
| 4 | cars\_cardata | 車両情報 |
| 5 | car\_brands\_brand | 車両\_メーカー |
| 6 | car\_models\_model | 車両\_車種 |
| 7 | car\_models\_submodel | 車両\_車種(サブ) |
| 8 | license\_plates\_licenseplate | ナンバープレート |
| 9 | license\_plates\_todoliststatus | ステータス\_Todoリスト |
| 10 | service\_histories\_servicehistory | 声掛けリスト/最終見積もりデータ |
| 11 | customers\_customerinformation | 顧客情報 |
| 12 | customers\_licenseplatecustomerinformation | 顧客・ナンバープレートのマッピングテーブル |
| 13 | customers\_memo | 車両/顧客メモ |
| 14 | customers\_phone | 使用していないテーブル |
| 15 | dealers\_defaultregion | 地域名一覧 |
| 16 | shops\_shop | 店舗(SS) |
| 17 | shops\_staff | 店舗(SS)スタッフ |
| 18 | qr\_codes\_qrcode | 車検証QRデータ |
| 19 | qr\_codes\_qrcodelicenseplate | ナンバープレート・車検証QRデータのマッピングテーブル |
| 20 | reminders\_reminder | 声掛けリスト/ディーラー別 |
| 21 | reminders\_remindercondition | 声掛けリスト設定 |
| 22 | products\_product | 商品 |
| 23 | product\_views\_productview | 商品グループ1・商品グループ2 |
| 24 | factories\_factory | 工場 |
| 25 | factory\_bookings\_factorybookingorderitem | 工場予約の見積もりデータ |
| 26 | factory\_bookings\_factorybooking | 工場予約カレンダー |
| 27 | factory\_bookings\_factorybooking\_mechanics | 工場予約カレンダー/工場スタッフ |
| 28 | factory\_capacities\_capacity | 工場予約カレンダー/キャパシティー設定 |
| 29 | orders\_order | 見積もりデータ |
| 30 | order\_items\_orderitem | 保存されたアイテムデータ/カーメンテ見積 |
| 31 | order\_items\_subgroup | 保存されたアイテムデータ/カーメンテ見積 |
| 32 | order\_items\_repairimage | リペアキズ画像 |
| 33 | order\_items\_subgroupproduct | 保存されたアイテムデータ/車検見積 |
| 34 | user\_profiles\_userprofile | ログインユーザーに関する追加情報 |
| 35 | user\_profiles\_userprofile\_shops | ログインユーザー・店舗のマッピングテーブル |
| 36 | auth\_user | 認証ユーザー |
| 37 | order\_items\_carinspectionitem | 車検見積もりに紐づくPIグループ項目 |
| 38 | recommendations\_recommendation | 基本６製品適合 |
| 39 | factories\_factory\_shop | 工場・店舗のマッピングテーブル |
| 40 | products\_carinpsectioncheckup | 車検Step2/事前点検項目 |
| 41 | products\_carinpsectionpartfeeitem | 車検Step2/部品・作業料(PI番号別) |
| 42 | products\_pigroupitem | 事前点検紐づけ番号(PI番号) |
| 43 | car\_checkups\_carcheckupitem | 使用されていないテーブル |
| 44 | booking\_slots\_factorybookingslot | リペア予約カレンダー/予約情報 |
| 45 | stages\_stage | リペア予約カレンダー/工程進捗情報 |
| 46 | stage\_logs\_stagelog | リペア予約カレンダー/工程進捗ログ |
| 47 | customers\_zipcodeaddress | 郵便番号データ |
| 48 | customers\_mapcodeaddress | マップコードデータ |
| 49 | car\_checkup\_orders\_carcheckuporder | 安全点検作成データ |
| 50 | car\_checkup\_order\_items\_carcheckuporderitem | 安全点検作成詳細データ |
| 51 | ios\_deploy\_iosapp | iOS/アプリ情報 |
| 52 | product\_views\_displayimage | 商品グループ1・2の画像データ |
| 53 | product\_views\_alacartecondition | 使用されていないテーブル |
| 54 | car\_brands\_carprice | 車両価格 |
| 55 | export\_results\_exportresult | エクスポート結果 |
| 56 | import\_results\_importresult | インポート結果 |
| 57 | announcements\_announcement | お知らせ |
| 58 | orders\_orderservice | 見積データに含まれるカテゴリーデータ |
| 59 | order\_items\_inspectionitem | オイル・タイヤ用作業完了確認項目データ |
| 60 | order\_items\_checkitem | 車検見積データ紐づくStep2項目のデータ |
| 61 | questionnaires\_question | アンケート質問情報 |
| 62 | questionnaires\_questionmenu | アンケートメニュー |
| 63 | questionnaires\_questionmenuquestion | アンケートメニュー・アンケート情報のマッピングテーブル |
| 64 | questionnaires\_questionchoice | アンケートメニュー・アンケート情報のマッピングテーブル |
| 65 | questionnaires\_answeritem | アンケート回答情報 |
| 66 | questionnaires\_answer | アンケート/ユーザー回答情報 |
| 67 | user\_profiles\_mechanicstaff | 工場スタッフ |
| 68 | car\_checkups\_carcheckup | 安全点検 |
| 69 | user\_profiles\_mechanicstafffactoryuser | ログインユーザー・工場のマッピングテーブル |
| 70 | car\_checkups\_carcheckupcomment | 安全点検備考情報 |
| 71 | recommendations\_recommendationremarks | 基本６製品適合備考情報 |
| 72 | order\_items\_portablecanorderitem | 携行缶詰替え販売 |
| 73 | order\_items\_carinspectionorder | 車検見積で選択・入力されたデータ |
| 74 | orders\_custodyorder | 預かり・代車・パーツ |
| 75 | order\_items\_custodyorderitem | パーツ発注で作成された商品データ |
| 76 | booking\_slots\_shopbooking | 見積データに紐づく予約データ |
| 77 | booking\_slots\_shopbookingslot | 自店店舗カレンダーで作成された予約データ |
| 78 | booking\_slots\_pit | 予約カレンダーのPIT情報 |
| 79 | orders\_orderimage | 見積データに含まれる画像(署名)データ |
| 80 | order\_items\_carinspectionquestionnaire | 車検Step1問診 |
| 81 | reminders\_remindermemo | 声掛けリスト/備考データ |
| 82 | shops\_alertfilter | 使用されていないテーブル |
| 83 | shops\_alertfilteroption | 使用されていないテーブル |
| 84 | custody\_loan\_lenders\_compensationdetail | 代車貸出証発行/補償詳細 |
| 85 | custody\_loan\_lenders\_ledger | 預かり・代車/帳票 |
| 86 | factories\_inspectionfactory | 車検実施店 |
| 87 | license\_plates\_todolistsetting | Todoリスト設定 |

### テーブル No.1

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | 支店名称 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.2

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | ディーラー名称 | `varchar(255)` |
|  | `tax_rate` | 税率 | `numeric(4,2)` |
|  | `logo` | ロゴ | `varchar(100)` |
|  | `is_production` | 本番環境 | `boolean` |
|  | `is_approval_function` | 承認機能 | `boolean` |
|  | `dealer_type` | ディーラー種別 | `smallint` |
| FK | `branch_id` | 支店ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `default_region_id` | 地域名ID(初期値) | `integer` |
|  | `is_default_sort` | 並び順(初期値)有効フラグ | `boolean` |
|  | `filter_config` | Webサイト上のフィルター設定を保存 | `jsonb` |

### テーブル No.3

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `type` | カテゴリーの種類を保存 | `integer` |
|  | `name` | 商品カテゴリー名称 | `varchar(50)` |
|  | `image` | 画像ファイルパス | `varchar(100)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
|  | `repair_service_type` | 使用されていない列 | `boolean` |

### テーブル No.4

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `data_type` | 使用されていない列 | `integer` |
|  | `front_photo` | 車両画像ファイルパス | `varchar(100)` |
|  | `brand` | メーカー名 | `varchar(255)` |
|  | `model` | 車種名 | `varchar(255)` |
|  | `color` | ボディーカラー | `varchar(255)` |
|  | `color_code` | ボディーカラーコード | `varchar(255)` |
|  | `registered_date` | 初年度登録年月 | `date` |
|  | `expired_date` | 車検満了日 | `date` |
|  | `mileage` | 現在の走行距離 | `integer` |
|  | `oil_check_up` | 使用されていない列 | `date` |
|  | `tire_check_up` | 使用されていない列 | `date` |
|  | `battery_check_up` | 使用されていない列 | `date` |
|  | `car_inspection_check_up` | 使用されていない列 | `date` |
|  | `model_type` | 車種種別 | `integer` |
|  | `model_size` | 車両サイズ | `varchar(3)` |
|  | `ios_version` | iOSバージョン | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `sub_model` | 車種名(サブ) | `varchar(255)` |
|  | `purchased_date` | 購入日 | `date` |
|  | `air_check_up` | 空気圧点検日 | `date` |
|  | `eneos_check_up` | ENEOSアプリID取得日 | `date` |
|  | `first_mileage` | 最初の走行距離 | `integer` |
|  | `line_check_up` | LINE友だち登録日 | `date` |
|  | `air_check_up_after` | 空気圧/後 | `numeric(4,1)` |
|  | `air_check_up_before` | 空気圧/前 | `numeric(4,1)` |
|  | `eneos_id` | ENEOSアプリID | `varchar(10)` |
|  | `end_leasing_date` | リース契約終了日 | `date` |
|  | `insurance_affiliated_store` | 任意保険契約店舗 | `integer` |
|  | `insurance_end_date` | 任意保険契約終了 | `date` |
|  | `insurance_start_date` | 任意保険契約開始 | `date` |
|  | `leasing_classification` | リース顧客区分 | `integer` |
|  | `leasing_dealer` | リース特約店 | `varchar(255)` |
|  | `leasing_shop` | リース担当SS | `varchar(255)` |
|  | `maintenance_pack` | メンテナンスパック | `integer` |
|  | `non_life_insurance` | 任意保険損害保険会社 | `varchar(255)` |
|  | `security_number` | 任意保険証券番号 | `varchar(255)` |
|  | `start_leasing_date` | リース契約開始日 | `date` |
|  | `tightening_check_up` | 増し締めチェック登録日 | `date` |
|  | `leasing_company` | リース会社 | `varchar(255)` |

### テーブル No.5

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | メーカー名称 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.6

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | 車種名称 | `varchar(255)` |
|  | `size` | 車両サイズ | `varchar(3)` |
|  | `type` | 車両種別 | `integer` |
| FK | `brand_id` | メーカー名ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `model_number` | 車種名番号 | `varchar(50)[]` |

### テーブル No.7

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 車種名のサブ名称 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `model_id` | 車種名ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.8

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `data_type` | 使用されていない列 | `integer` |
|  | `number` | 4桁車番 | `varchar(20)` |
|  | `hiragana_prefix` | ひらがな | `varchar(50)` |
|  | `vehicle_class_number` | 分類番号 | `varchar(100)` |
|  | `regional_code` | 地域名 | `varchar(50)` |
|  | `qr_code_snapshot` | 車検証情報スナップショット | `jsonb` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |

### テーブル No.9

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `customer_info` | 顧客情報 | `boolean` |
|  | `eneos_check_up` | ENEOSアプリID取得日 | `boolean` |
|  | `air_check_up` | 空気圧点検日 | `boolean` |
|  | `checkup_done` | 声掛けリスト不要の判定 | `boolean` |
|  | `qr` | 車検証 | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `line_check_up` | LINE友だち登録日 | `boolean` |
|  | `last_contact_history` | 接点履歴 | `jsonb` |

### テーブル No.10

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `service_type` | カテゴリーの種類を保存 | `integer` |
|  | `status` | 見積ステータス | `integer` |
|  | `action` | データの保存処理を判別(手動リセットor納車済) | `integer` |
|  | `detail` | 商品価格などの詳細情報を保存 | `jsonb` |
|  | `date` | 納車済に変更された日付 | `date` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `update_reminder` | 声掛けリスト | `boolean` |
|  | `mileage` | 現在の走行距離 | `integer` |

### テーブル No.11

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `is_imported` | インポート | `boolean` |
|  | `data_type` | 使用されていない列 | `integer` |
|  | `family_name` | 姓 | `varchar(255)` |
|  | `kana_name` | カナ姓 | `varchar(255)` |
|  | `ios_version` | iOSバージョン | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `last_created_order_id` | 最後に作成された見積ID | `integer` |
|  | `line_friend_suffix` | LINE友だち末尾文字列 | `varchar(255)` |
|  | `line_qr_code` | LINE友だちQRコード文字列 | `varchar(100)` |
|  | `is_show_memo` | メモを表示 | `boolean` |
|  | `payment_method` | 決済方法 | `smallint[]` |
|  | `_address_line_1_data` | 使用されていない列 | `bytea` |
|  | `_address_line_2_data` | 使用されていない列 | `bytea` |
|  | `_email_data` | メールアドレスデータ/暗号化 | `bytea` |
|  | `_given_name_data` | 名データ/暗号化 | `bytea` |
|  | `_house_phone_number_data` | 連絡先2データ/暗号化 | `bytea` |
|  | `_mobile_phone_number_data` | 連絡先1データ/暗号化 | `bytea` |
|  | `address_line_1` | 使用されていない列 | `varchar(66)` |
|  | `address_line_2` | 使用されていない列 | `varchar(66)` |
|  | `email` | メールアドレス | `varchar(66)` |
|  | `given_name` | 名 | `varchar(66)` |
|  | `house_phone_number` | 連絡先2 | `varchar(66)` |
|  | `mobile_phone_number` | 連絡先1 | `varchar(66)` |
|  | `is_term_accepted` | お客様情報の取り扱い | `boolean` |
|  | `email_notification` | メール案内 | `integer` |
|  | `juristic` | 個人/法人 | `integer` |
|  | `letter_notification` | DM案内 | `integer` |
|  | `studless_tire_in_store` | タイヤ預かり | `integer` |
|  | `use_company_name_for_display` | 帳票の宛名に使用するチェックボックス | `boolean` |
|  | `address_line_1_decrypt` | 使用されていない列 | `varchar(255)` |
|  | `address_line_2_decrypt` | 使用されていない列 | `varchar(255)` |
|  | `birth_date_copy` | 生年月日/複製 | `date` |
|  | `building_copy` | 建物名/複製 | `varchar(255)` |
|  | `city_copy` | 市区町村/複製 | `varchar(255)` |
|  | `company_name_copy` | 会社名/複製 | `varchar(255)` |
|  | `driver_license_number_copy` | 運転免許証/複製 | `varchar(255)` |
|  | `house_number_copy` | 建物名/複製 | `varchar(255)` |
|  | `kana_name_given_copy` | カナ名/複製 | `varchar(255)` |
|  | `map_code_copy` | マップコード/複製 | `varchar(100)` |
|  | `member_number_copy` | 会員クラスA/複製 | `varchar(255)` |
|  | `member_number_email_copy` | 会員クラスB/複製 | `varchar(255)` |
|  | `member_number_eneos_copy` | 会員クラスC/複製 | `varchar(255)` |
|  | `member_number_other_copy` | 会員クラスD/複製 | `varchar(255)` |
|  | `normalized_house_phone_number_copy` | 正規化された連絡先2/複製 | `varchar(255)` |
|  | `normalized_mobile_phone_number_copy` | 正規化された連絡先1/複製 | `varchar(255)` |
|  | `postal_code_copy` | 郵便番号/複製 | `varchar(255)` |
|  | `prefecture_copy` | 都道府県/複製 | `varchar(255)` |
|  | `town_copy` | 番地/複製 | `varchar(255)` |
|  | `eneos_customer_id_copy` | ENEOSアプリID/複製 | `varchar(10)` |
|  | `_birth_date_data` | 生年月日/暗号化 | `bytea` |
|  | `_building_data` | 建物名/暗号化 | `bytea` |
|  | `_city_data` | 市区町村/暗号化 | `bytea` |
|  | `_company_name_data` | 会社名/暗号化 | `bytea` |
|  | `_driver_license_number_data` | 運転免許証/暗号化 | `bytea` |
|  | `_house_number_data` | 旧建物名データ/暗号化 | `bytea` |
|  | `_kana_name_given_data` | カナ名/暗号化 | `bytea` |
|  | `_map_code_data` | マップコード/暗号化 | `bytea` |
|  | `_member_number_data` | 会員クラスA/暗号化 | `bytea` |
|  | `_member_number_email_data` | 会員クラスB/暗号化 | `bytea` |
|  | `_member_number_eneos_data` | 会員クラスC/暗号化 | `bytea` |
|  | `_member_number_other_data` | 会員クラスD/暗号化 | `bytea` |
|  | `_normalized_house_phone_number_data` | 正規化された連絡先2/暗号化 | `bytea` |
|  | `_normalized_mobile_phone_number_data` | 正規化された連絡先1/暗号化 | `bytea` |
|  | `_postal_code_data` | 郵便番号/暗号化 | `bytea` |
|  | `_prefecture_data` | 都道府県/暗号化 | `bytea` |
|  | `_town_data` | 番地/暗号化 | `bytea` |
|  | `_eneos_customer_id_data` | ENEOSアプリID/暗号化 | `bytea` |
|  | `birth_date` | 生年月日 | `varchar(66)` |
|  | `building` | 建物名 | `varchar(66)` |
|  | `city` | 市区町村 | `varchar(66)` |
|  | `company_name` | 会社名 | `varchar(66)` |
|  | `driver_license_number` | 運転免許証 | `varchar(66)` |
|  | `house_number` | 旧建物名 | `varchar(66)` |
|  | `kana_name_given` | カナ名 | `varchar(66)` |
|  | `map_code` | マップコード | `varchar(66)` |
|  | `member_number` | 会員クラスA | `varchar(66)` |
|  | `member_number_email` | 会員クラスB | `varchar(66)` |
|  | `member_number_eneos` | 会員クラスC | `varchar(66)` |
|  | `member_number_other` | 会員クラスD | `varchar(66)` |
|  | `normalized_house_phone_number` | 正規化された連絡先2 | `varchar(66)` |
|  | `normalized_mobile_phone_number` | 正規化された連絡先1 | `varchar(66)` |
|  | `postal_code` | 郵便番号 | `varchar(66)` |
|  | `prefecture` | 都道府県 | `varchar(66)` |
|  | `town` | 番地 | `varchar(66)` |
|  | `eneos_customer_id` | ENEOSアプリID | `varchar(66)` |

### テーブル No.12

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_info_id` | 顧客情報ID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.13

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `memo` | メモ | `text` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_id` | 顧客ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `staff_id` | スタッフID | `integer` |

### テーブル No.14

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `phone_number` | 店舗電話番号 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_info_id` | 顧客情報ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.15

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | 地域名 | `varchar(100)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.16

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `address_line_1` | 使用されていない列 | `varchar(255)` |
|  | `address_line_2` | 使用されていない列 | `varchar(255)` |
|  | `prefecture` | 都道府県 | `varchar(255)` |
|  | `town` | 番地 | `varchar(255)` |
|  | `city` | 市区町村 | `varchar(255)` |
|  | `building` | 建物名 | `varchar(255)` |
|  | `postal_code` | 郵便番号 | `varchar(255)` |
|  | `name` | 店舗名 | `varchar(255)` |
|  | `email` | メールアドレス | `varchar(255)` |
|  | `phone_number` | 店舗電話番号 | `varchar(75)` |
|  | `fax_number` | 店舗ファックス番号 | `varchar(75)` |
|  | `is_active` | 店舗有効フラグ | `boolean` |
|  | `is_production` | 本番環境 | `boolean` |
|  | `default_region` | 地域名(初期値) | `varchar(50)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `inspection_factory_id` | 車検工場ID | `integer` |
| FK | `to_do_list_setting_id` | ToDoリスト設定ID | `integer` |

### テーブル No.17

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `active` | 有効フラグ | `boolean` |
| FK | `shop_id` | 店舗ID | `integer` |
|  | `family_name` | 姓 | `varchar(255)` |
|  | `sequence` | 並び順 | `integer` |
|  | `given_name` | 名 | `varchar(255)` |
|  | `name` | スタッフの姓名 | `varchar(255)` |
|  | `alive` | 有効フラグ | `boolean` |

### テーブル No.18

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `number` | 4桁車番 | `varchar(20)` |
|  | `hiragana_prefix` | ひらがな | `varchar(50)` |
|  | `vehicle_class_number` | 分類番号 | `varchar(100)` |
|  | `regional_code` | 地域名 | `varchar(50)` |
|  | `photo` | 車検証QRパス | `varchar(100)` |
|  | `model_number` | 車種名番号 | `varchar(255)` |
|  | `registered_date` | 初年度登録年月 | `date` |
|  | `weight` | 重量 | `integer` |
|  | `total_weight` | 総重量 | `integer` |
|  | `oil_type` | オイル種別 | `varchar(255)` |
|  | `vehicle_identification_number` | 車台番号 | `varchar(255)` |
|  | `specification_number` | 型式指定番号 | `varchar(255)` |
|  | `classification_number` | 類別区分番号 | `varchar(255)` |
|  | `engine_model_code` | 原動機型式 | `varchar(255)` |
|  | `expired_date` | 車検満了日 | `date` |
|  | `shop_id_ultrapart` | 使用されていない列 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.19

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `qr_code_id` | 車検証情報ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.20

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `type` | 声掛けリスト項目の種別 | `integer` |
|  | `is_active` | 声掛けリスト有効フラグ | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.21

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `condition_type` | 期間項目 | `integer` |
|  | `condition_unit` | 期間単位 | `integer` |
|  | `condition_value` | 期間数値 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `reminder_id` | 声掛けリストID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.22

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `price_type` | 価格種別 | `integer` |
|  | `product_type` | 商品種別 | `integer` |
|  | `name` | 商品名称 | `varchar(100)` |
|  | `price` | 価格 | `integer` |
|  | `is_active` | 商品有効フラグ | `boolean` |
|  | `attribute` | 複数の属性情報 | `jsonb` |
|  | `description` | 商品説明 | `varchar(255)` |
|  | `hash` | ハッシュ | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `product_view_id` | 商品表示設定ID | `integer` |
| FK | `service_id` | 商品カテゴリーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `jan_code` | JANコード | `varchar(255)` |
|  | `product_code` | 商品コード | `varchar(255)` |
|  | `remarks` | 備考 | `text` |

### テーブル No.23

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | 商品選択画面に表示される商品ビューの名称 | `varchar(100)` |
|  | `multiple_selection` | 複数選択フラグ | `boolean` |
|  | `display_mode` | UI表示モード | `varchar(50)` |
|  | `attribute` | 複数の属性データ | `jsonb` |
|  | `hash` | ハッシュ | `varchar(255)` |
|  | `tag` | 使用されていない列 | `ARRAY` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `parent_id` | 親ID | `integer` |
| FK | `service_id` | 商品カテゴリーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `use_reminder` | 声掛けリスト機能として保存するかどうかを判定 | `boolean` |

### テーブル No.24

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 工場名称 | `varchar(255)` |
|  | `email` | メールアドレス | `varchar(255)` |
|  | `address` | 住所 | `varchar(255)` |
|  | `touch_capacity` | タッチ数キャパシティ | `integer` |
|  | `car_capacity` | 車両台数キャパシティ | `integer` |
|  | `is_active` | 工場有効フラグ | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `drop_off` | 納車 | `smallint` |
|  | `pickup` | 来店日付/工場予約 | `smallint` |
|  | `mechanic_capacity` | 納車日付/工場予約 | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |

### テーブル No.25

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `order_item_type` | リペアキズ種別 | `integer` |
|  | `part_name` | リペアキズ名称 | `varchar(100)` |
|  | `will_be_fixed` | リペア作業完了予定 | `boolean` |
|  | `position` | リペアキズ箇所 | `varchar(255)` |
|  | `side` | 車両パネルの位置的属性 | `varchar(10)` |
|  | `type` | 凹みorキズ | `varchar(10)` |
|  | `repair_images` | リペアキズ画像 | `jsonb` |
|  | `touch` | タッチ数 | `integer` |
|  | `status` | 作業ステータス(完了or未完了) | `integer` |
|  | `completed_datetime` | 作業終了日 | `timestampwithtimezone` |
| FK | `booking_id` | 予約ID | `integer` |
|  | `actual_touch` | 実タッチ数 | `integer` |
|  | `after_repaired_image` | リペア修理後画像 | `varchar(100)` |

### テーブル No.26

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `status` | 見積データステータスキャンセル=-1,予約済=0,受付完了=1,リペア作業中=2,最終確認待ち=3,最終確認済=4,作業完了=5,納車済=6 | `integer` |
|  | `order_number` | 見積もり番号 | `varchar(100)` |
|  | `shop_name` | 店舗名 | `varchar(255)` |
|  | `branch_name` | 支店名 | `varchar(255)` |
|  | `dealer_name` | ディーラー名 | `varchar(255)` |
|  | `shop_comment` | 店舗メモ | `varchar(200)` |
|  | `mechanic_comments` | 工場メモ | `text` |
|  | `start_date` | 作業開始日 | `date` |
|  | `end_date` | 作業終了日 | `date` |
|  | `drop_off` | 納車日時 | `timestampwithtimezone` |
|  | `pickup` | 来店日/工場予約 | `timestampwithtimezone` |
|  | `product_category_snapshot` | カテゴリースナップショット | `text` |
|  | `alt_color_code` | ALT | `varchar(255)` |
|  | `color_code` | カラーコード | `varchar(255)` |
|  | `car_data` | 車両データ | `jsonb` |
|  | `shop_data` | 店舗データ | `jsonb` |
| FK | `factory_id` | 工場ID | `integer` |
| FK | `reviewer_id` | 最終確認者ID | `integer` |
|  | `staff_comment` | 最終確認コメント | `text` |
|  | `actual_pick_up_datetime` | 見積データが納車済で保存された日付 | `timestampwithtimezone` |
| FK | `order_id` | 見積もりID | `integer` |
|  | `memo` | メモ | `text` |

### テーブル No.27

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `factorybooking_id` | 工場予約ID | `integer` |
| FK | `mechanicstaff_id` | 工場技術者ID | `integer` |

### テーブル No.28

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `date` | キャパシティ設定が更新された日付 | `date` |
|  | `touch_capacity` | タッチ数キャパシティ | `integer` |
|  | `car_capacity` | 車両台数キャパシティ | `integer` |
|  | `mechanic_capacity` | 納車日/工場予約 | `integer` |
| FK | `factory_id` | 工場ID | `integer` |
| FK | `mechanic_id` | 工場技術者ID | `integer` |

### テーブル No.29

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `data_type` | 使用されていない列 | `integer` |
|  | `status` | 見積ステータス | `integer` |
|  | `order_number` | 見積番号 | `varchar(100)` |
|  | `sales_datetime` | 精算日時 | `timestampwithtimezone` |
|  | `customer` | 顧客 | `jsonb` |
|  | `customer_signature` | 顧客署名パス | `varchar(100)` |
|  | `car_info` | 車両情報 | `jsonb` |
|  | `suggestion_for_customer` | 使用されていない列 | `text` |
|  | `staff_comment` | 使用されていない列 | `text` |
|  | `staff_signature` | スタッフ署名パス | `varchar(100)` |
|  | `dropoff_datetime` | お預かり日時 | `timestampwithtimezone` |
|  | `pick_up_datetime` | 納車日時 | `timestampwithtimezone` |
|  | `actual_pick_up_datetime` | 納車済の日付 | `timestampwithtimezone` |
|  | `paid` | 使用されていない列 | `boolean` |
|  | `deposit_amount` | お預かり金額 | `numeric(12,2)` |
|  | `mileage` | 現在の走行距離 | `numeric(12,2)` |
|  | `repair_factory_id` | リペア工場ID | `integer` |
|  | `repair_factory` | リペア工場名称 | `varchar(255)` |
|  | `repair_start_date` | リペア作業開始日 | `date` |
|  | `repair_end_date` | リペア作業終了日 | `date` |
|  | `repair_slot_ids` | リペア予約ID(複数) | `integer[]` |
|  | `type` | 使用されていない列 | `varchar(50)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_info_id` | 顧客情報ID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `staff_id` | スタッフID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `is_car_rental` | 使用されていない列 | `boolean` |
|  | `total_price` | 合計価格 | `numeric(12,2)` |
|  | `applied_date` | 車検Step4/お申込日 | `timestampwithtimezone` |
|  | `car_inspection_receive_option` | 車検証・検査標章受取方法 | `smallint` |
|  | `car_inspection_step` | 車検STEP | `smallint` |
|  | `checking_staff_id` | 作業確認した担当スタッフのスタッフID | `integer` |
|  | `first_expected_date` | 使用されていない列 | `timestampwithtimezone` |
|  | `fixing_staff_id` | 見積もりを処理した担当スタッフのスタッフID | `integer` |
|  | `is_substitute_car` | 代車有 | `boolean` |
|  | `is_telephone` | 終了後TEL確認チェックボックス | `boolean` |
|  | `parking_fine_option` | 駐車違反金滞納有無 | `smallint` |
|  | `pre_inspection_date` | 事前点検スケジュール | `timestampwithtimezone` |
|  | `second_expected_date` | 使用されていない列 | `timestampwithtimezone` |
|  | `visit` | 車検Step4で、来店か郵送かを判定・保存する項目 | `smallint` |
|  | `work_completion_confirmation_items` | 作業開始前に表示されるポップアップのデータ | `jsonb` |
|  | `work_delivered_confirmation_items` | 作業完了前に表示されるポップアップのデータ | `jsonb` |
|  | `booked_staff` | 予約スタッフ名称 | `varchar(256)` |
|  | `work_complete_staff` | 作業完了スタッフ名称 | `varchar(256)` |
|  | `work_start_staff` | 作業開始スタッフ名称 | `varchar(256)` |

### テーブル No.30

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 商品名称 | `varchar(256)` |
|  | `product` | 商品の詳細情報 | `jsonb` |
|  | `quantity` | 数量 | `numeric(12,2)` |
|  | `price` | 価格 | `numeric(12,2)` |
|  | `uuid` | ランダムな一意識別番号 | `uuid` |
|  | `sequence` | 並び順 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `severity` | 使用されていない列 | `integer[]` |
|  | `result` | 見積もりアイテムの判定結果(O・X・△) | `integer` |

### テーブル No.31

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `name` | 車検見積もりで商品選択時に表示されるレベル1項目名 | `varchar(256)` |
| FK | `car_inspection_item_id` | 車検項目テーブルを紐付けるための外部キー | `integer` |

### テーブル No.32

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `uuid` | 各画像ごとの一意識別テキスト | `varchar(100)` |
|  | `repair_image` | リペアキズ画像パス | `varchar(100)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `order_item_id` | 見積商品ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.33

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `name` | 車検見積もりで選択された商品の名称 | `varchar(256)` |
|  | `quantity` | 数量 | `numeric(12,2)` |
|  | `unit` | 単位 | `varchar(100)` |
|  | `price` | 価格 | `numeric(12,2)` |
|  | `category_name` | カテゴリー名称 | `varchar(100)` |
| FK | `sub_group_id` | サブグループタブへの関連ID | `integer` |
|  | `product_type` | 商品種別 | `integer` |
|  | `attribute` | 車検見積もりごとに保存するための商品詳細属性データ | `jsonb` |
|  | `etr_product_id` | ETRBUY商品ID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
|  | `product_id` | 商品ID | `integer` |
|  | `factory_check_status` | 工場作業ステータス | `integer` |
|  | `is_complete` | カート画面の確認チェックボックス | `boolean` |

### テーブル No.34

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `given_name` | 名 | `varchar(255)` |
|  | `family_name` | 姓 | `varchar(255)` |
|  | `phone_number_1` | 連絡先1 | `varchar(255)` |
|  | `phone_number_2` | 連絡先2 | `varchar(255)` |
|  | `email` | メールアドレス | `varchar(255)` |
|  | `profile_picture` | 使用されていない列 | `varchar(100)` |
|  | `customer_code` | 顧客コード | `varchar(255)` |
|  | `image` | 画像ファイルパス | `varchar(100)` |
|  | `role` | 権限 | `integer` |
| FK | `branch_id` | 支店ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `user_id` | ユーザーID | `integer` |
|  | `attempt` | ログイン試行回数 | `integer` |
|  | `lock_time` | ログイン失敗によりアカウントが一時的にロックされた時間 | `timestampwithtimezone` |
|  | `ios_version` | iOSバージョン | `varchar(255)` |
|  | `password_at` | パスワード設定日 | `date` |

### テーブル No.35

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `userprofile_id` | ユーザープロフィールID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |

### テーブル No.36

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `password` | パスワード | `varchar(128)` |
|  | `last_login` | 最終ログイン日時 | `timestampwithtimezone` |
|  | `is_superuser` | adminユーザーフラグ | `boolean` |
|  | `username` | ユーザー名 | `varchar(150)` |
|  | `first_name` | 名 | `varchar(150)` |
|  | `last_name` | 姓 | `varchar(150)` |
|  | `email` | メールアドレス | `varchar(254)` |
|  | `is_staff` | スタッフアクティブフラグ | `boolean` |
|  | `is_active` | 有効フラグ | `boolean` |
|  | `date_joined` | IDが作成された日付 | `timestampwithtimezone` |

### テーブル No.37

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `header_name` | ヘッダー名称 | `varchar(256)` |
|  | `title_name` | 車検見積もりと保存されるタイトル名称 | `varchar(256)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `pi_number` | 事前点検紐づけ番号 | `integer` |
|  | `details` | 車検見積もりと保存される詳細情報 | `varchar(256)` |
|  | `item_name` | 車検見積もりと保存されるアイテム名称 | `varchar(256)` |
|  | `ledger_number` | 帳票番号 | `integer` |

### テーブル No.38

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `model_number` | 車種番号 | `varchar(100)` |
|  | `recommendation_type` | 適合カテゴリー | `varchar(10)` |
|  | `recommendation_data` | 適合型番情報 | `jsonb` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `classification_number` | 類別区分番号 | `varchar(255)` |
|  | `specification_number` | 型式指定番号 | `varchar(255)` |
|  | `product_code` | 商品コード | `varchar(100)` |
|  | `remarks` | 備考 | `text[]` |
|  | `brand` | メーカー名 | `varchar(100)` |
|  | `car_size` | 車両サイズ | `varchar(4)` |
|  | `car_type` | 車両種別 | `varchar(30)` |
|  | `model` | 車種名 | `varchar(100)` |
|  | `model_year` | 車種年式年月 | `date` |
|  | `model_year_end` | 車種年式終了年月 | `date` |

### テーブル No.39

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `factory_id` | 工場ID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |

### テーブル No.40

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | 車検ステップ2の項目名 | `varchar(255)` |
|  | `attribute` | レベル1名称やドラムロールの値などの詳細属性データ | `jsonb` |
|  | `input_type` | 入力方式を示す項目 | `integer` |
|  | `pi_number` | 事前点検紐づけ番号 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `parent_id` | 車検Step2/左右の対応関係を識別するID | `integer` |
| FK | `pi_group_id` | 関連するPI番号テーブルID | `integer` |
| FK | `service_id` | 商品カテゴリーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `ledger_number` | 帳票番号 | `integer` |

### テーブル No.41

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `pi_number` | 事前点検紐づけ番号 | `integer` |
|  | `name` | ポップアップの部品・作業料項目の名称 | `varchar(255)` |
|  | `price` | 価格 | `integer` |
|  | `unit` | 単位 | `varchar(100)` |
|  | `attribute` | ディーラーIDとユーザーID | `jsonb` |
|  | `product_type` | 商品種別 | `integer` |
|  | `service_type` | ステップ2に紐づくカテゴリー種別 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `pi_group_id` | 関連するPI番号テーブルID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.42

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `group_name` | ステップ2のタブ名称 | `varchar(100)` |
|  | `pi_number` | 事前点検紐づけ番号 | `integer` |
|  | `item_name` | 事前点検報告書に表示される項目左の情報 | `varchar(100)` |
|  | `details` | 事前点検報告書に表示される項目右の情報 | `varchar(100)` |
|  | `ledger_number` | 帳票番号 | `integer` |
|  | `sequence` | 並び順 | `integer` |

### テーブル No.43

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `created_at` | 作成日時 | `timestampwithtimezone` |
| FK | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 使用されていない列 | `varchar(255)` |
|  | `price` | 価格 | `numeric(12,2)` |
|  | `type` | 使用されていない列 | `integer` |
|  | `attribute` | 使用されていない列 | `jsonb` |
|  | `created_user_id` | 作成ユーザーID | `integer` |
|  | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.44

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `booking_slot_status` | 各予約枠のステータス | `integer` |
|  | `booking_slot_type` | 各予約枠の種別 | `smallint` |
|  | `shop_name` | 店舗名 | `varchar(255)` |
|  | `slot_date` | リペア作業開始日 | `date` |
|  | `touches` | 各項目のタッチ数 | `integer` |
|  | `device_uuid` | iPadの一意識別子 | `uuid` |
| FK | `factory_id` | 工場ID | `integer` |
| FK | `factory_booking_id` | 予約に関連するID | `integer` |

### テーブル No.45

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `stage_number` | リペア作業ステータス鈑金=0,パテ=1,サフェーサー=2,塗装=3,磨き=4 | `smallint` |
| FK | `mechanic_id` | 技術者ID(作業終了実施) | `integer` |
| FK | `order_item_id` | 見積商品ID | `integer` |

### テーブル No.46

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `log_event` | 修理項目ごとの作業ステータスを保存するためのデータ | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
| FK | `stage_id` | 各工程に関連するID | `integer` |

### テーブル No.47

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `zipcode` | 郵便番号 | `varchar(100)` |
|  | `region` | 地域 | `varchar(100)` |
|  | `prefecture` | 都道府県 | `varchar(100)` |
|  | `city` | 市区町村 | `varchar(100)` |
|  | `town` | 番地 | `varchar(100)` |
|  | `building` | 建物名 | `varchar(100)` |

### テーブル No.48

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `prefecture` | 都道府県 | `varchar(100)` |
|  | `town` | 番地 | `varchar(100)` |
|  | `prefecture_code` | 都道府県コード | `varchar(10)` |
|  | `town_code` | 番地コード | `varchar(10)` |
|  | `town_code_extension` | 番地拡張コード | `varchar(10)` |
|  | `searchable_code` | マップコード検索でフィルタ条件として使用されるコード値 | `varchar(15)` |

### テーブル No.49

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_id` | 顧客ID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `staff_id` | スタッフID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `status` | 安全点検見積ステータス保存済=0,予約済=1,作業中=2,作業終了=3 | `integer` |
|  | `customer_snapshot` | 顧客データのスナップショット | `jsonb` |
|  | `car_data_snapshot` | 車両データのスナップショット | `jsonb` |
|  | `license_plate_snapshot` | ナンバープレートデータのスナップショット | `jsonb` |
|  | `checkup_datetime` | 安全点検データ作成日 | `timestampwithtimezone` |
|  | `attribute` | 3、8、15項目などの詳細属性データ | `jsonb` |

### テーブル No.50

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 安全点検で選択された各チェック項目の名称 | `varchar(256)` |
|  | `attribute` | レベル1・レベル2・安全点検などの詳細属性 | `jsonb` |
|  | `result` | 選択された〇、X、△ | `integer` |
|  | `input_datetime` | ドラムロールからの入力日時 | `timestampwithtimezone` |
|  | `input_text` | フリー入力テキストデータ | `text` |
|  | `comment` | 安全点検備考/コメント | `text` |
|  | `alacarte_name` | 使用されていない列 | `varchar(100)` |
| FK | `checkup_order_id` | 安全点検データに関連するID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.51

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 固定値 | `varchar(50)` |
|  | `title` | 固定値 | `varchar(20)` |
|  | `app_type` | 固定値 | `integer` |
|  | `build_version` | アプリケーションのビルド | `varchar(20)` |
|  | `version` | バージョン | `varchar(20)` |
|  | `plist` | iOSアプリの付随情報 | `varchar(100)` |
|  | `ipa` | ダウンロードファイル | `varchar(100)` |
|  | `release_note` | 更新の備考 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `is_active` | 有効フラグ | `boolean` |

### テーブル No.52

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `type` | ヘッダー画像・ポップアップ・コンテンツ | `integer` |
|  | `image` | 画像ファイルパス | `varchar(100)` |
|  | `attribute` | 追加情報・個別属性 | `jsonb` |
|  | `product_view_hash` | 画像を紐づけるための、商品ビュー項目の一意識別子 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.53

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 使用されていない列 | `varchar(254)` |
|  | `x_position` | 使用されていない列 | `numeric(10,7)` |
|  | `y_position` | 使用されていない列 | `numeric(10,7)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.54

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `registered_year` | 初年度登録年 | `integer` |
|  | `katashiki` | 型式 | `varchar(100)` |
|  | `minimum_price` | 使用されていない列 | `numeric(20,2)` |
|  | `maximum_price` | 使用されていない列 | `numeric(20,2)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.55

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `file` | ファイルのURL | `varchar(100)` |
|  | `export_type` | エクスポートするデータの種類を示す種別 | `varchar(100)` |
|  | `export_status` | エクスポートステータス | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.56

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `file` | ファイルのURL | `varchar(100)` |
|  | `import_type` | インポートするデータの種類を示す種別 | `varchar(100)` |
|  | `file_info` | ファイル名、拡張子などの詳細情報 | `jsonb` |
|  | `status` | インポートステータス | `boolean` |
|  | `error_message` | インポートエラーメッセージ | `text` |
|  | `import_status` | インポートステータス | `integer` |
|  | `error_messages` | エラーメッセージリスト | `jsonb` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.57

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `message` | お知らせメッセージ | `text` |
|  | `picture` | お知らせ画像パス | `varchar(100)` |
|  | `start_date` | 開始日 | `date` |
|  | `end_date` | 終了日 | `date` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
|  | `title` | お知らせタイトル名称 | `varchar(255)` |

### テーブル No.58

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `status` | 見積もりステータスと同様のステータス種別 | `integer` |
|  | `service_type` | 見積もりのカテゴリー種別 | `integer` |
|  | `work_completion` | 作業完了 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.59

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `name` | オイル・タイヤ向けに表示されるポップアップ項目名称 | `varchar(256)` |
|  | `checked` | チェックボックス有効フラグ | `boolean` |
|  | `service_snapshot` | 対象のレコードが、オイルかタイヤかを識別するためのタイプ値 | `jsonb` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `service_id` | 商品カテゴリーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.60

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `criteria_title` | 車検見積もりに紐づくステップ2項目の情報 | `varchar(256)` |
|  | `name` | 車検見積もりに紐づくStep2項目の名称 | `varchar(256)` |
|  | `selected_status` | 車検ステップ2の項目が選択されたかどうかを示すフラグ | `integer` |
| FK | `car_inspection_item_id` | PIデータに関連するID | `integer` |
|  | `attribute` | 詳細属性データ | `jsonb` |
|  | `input_text` | ドラムロールからの入力日時 | `varchar(256)` |
|  | `input_type` | フリー入力テキストデータ | `integer` |
|  | `ledger_number` | 帳票番号 | `integer` |

### テーブル No.61

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `question` | アンケート質問内容 | `text` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `is_customer` | 顧客・スタッフフラグ | `boolean` |
| FK | `dealer_id` | ディーラーID | `integer` |
|  | `is_multiple_choice` | 複数選択式かどうかを表すフラグ | `boolean` |

### テーブル No.62

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 質問メニュー名称 | `varchar(254)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |

### テーブル No.63

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `question_id` | 質問に関連するID | `integer` |
| FK | `question_menu_id` | 質問メニューに関連するID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.64

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `name` | 各質問に対する各選択肢の名称 | `text` |
| FK | `question_id` | 質問に関連するID | `integer` |
|  | `is_free_input` | 自由入力方式フラグ | `boolean` |

### テーブル No.65

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `is_customer` | 顧客・スタッフフラグ | `boolean` |
|  | `answer_text` | アンケート回答内容 | `text` |
| FK | `answer_id` | 回答に関連するID | `integer` |
| FK | `choice_id` | 選択肢に関連するID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.66

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `question_menu_id` | 質問メニューに関連するID | `integer` |
| FK | `staff_id` | スタッフID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `customer_license_plate_id` | 顧客・ナンバープレートテーブルに関連するID | `integer` |
|  | `remarks` | アンケート回答備考 | `text` |

### テーブル No.67

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `name` | スタッフ名 | `varchar(255)` |
|  | `is_active` | スタッフ有効フラグ | `boolean` |
| FK | `factory_id` | 工場ID | `integer` |

### テーブル No.68

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `inspection_type` | 安全点検種別 | `integer` |
|  | `inspection_area` | 点検項目(3、8、15項目) | `varchar(100)` |
|  | `name` | 安全点検名称 | `varchar(255)` |
|  | `input_type` | 各安全点検項目の選択肢タイプを表す区分値数値=0,日付=1,選択肢リスト=2,数値+単位=3,入力=4 | `integer` |
|  | `attribute` | レベル1名称やドラムロールの値などの詳細属性データ | `jsonb` |
|  | `priority_sequence` | 優先順位 | `integer` |
| FK | `branch_id` | 支店ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `parent_id` | 左右に分かれる車両点検項目の親項目に関連するID | `integer` |
| FK | `service_id` | 商品カテゴリーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `category_name` | カテゴリー名称 | `varchar(255)[]` |
|  | `etr_category_id` | ETRBUYのカテゴリーID | `integer[]` |
|  | `pi_group_id` | PIデータに関連するID | `integer` |

### テーブル No.69

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
| FK | `factory_id` | 工場ID | `integer` |
| FK | `user_profile_id` | ユーザープロフィール | `integer` |

### テーブル No.70

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |
|  | `type` | 使用されていない列 | `varchar(10)` |
|  | `comment` | 安全点検の備考コメント | `varchar(255)` |
| FK | `branch_id` | 支店ID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `category` | カテゴリー | `varchar(255)` |
|  | `checkup_id` | 各点検項目に対する関連ID | `integer` |
|  | `product` | 安全点検ポップアップの最左列の項目 | `varchar(255)` |
|  | `status` | 点検項目の状態を表すO,X,△のステータス値 | `varchar(1)` |
|  | `sub_category` | 安全点検備考/分類2 | `varchar(255)` |

### テーブル No.71

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `type` | 適合カテゴリー種別 | `varchar(10)` |
|  | `remarks` | 適合備考 | `text` |
|  | `number` | 4桁車番 | `integer` |

### テーブル No.72

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | アイテム名称 | `varchar(256)` |
|  | `quantity` | 数量 | `numeric(12,2)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `portable_can_order_id` | 携行缶販売テーブルに関連するID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.73

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `step_one_selected_staff` | 車検Step1/スタッフ名称 | `varchar(256)` |
|  | `step_two_selected_staff` | 車検Step2/スタッフ名称 | `varchar(256)` |
|  | `step_three_selected_staff` | 車検Step3/スタッフ名称 | `varchar(256)` |
|  | `step_four_selected_staff` | 車検Step4/スタッフ名称 | `varchar(256)` |
|  | `step_five_selected_staff` | 車検Step5/スタッフ名称 | `varchar(256)` |
|  | `step_five_confirmation_items` | 車検Step5完了前に表示されるチェックリストに関するデータ | `jsonb` |
|  | `step_six_selected_staff` | 車検Step6/スタッフ名称 | `varchar(256)` |
|  | `step_six_work_done_staff` | 車検Step6/作業完了スタッフ名称 | `varchar(256)` |
|  | `step_six_confirmation_staff` | 車検Step6/作業確認スタッフ名称 | `varchar(256)` |
|  | `step_six_confirmation_items` | 車検Step6完了前に表示されるチェックリストに関するデータ | `jsonb` |
|  | `step_six_signature` | 車検Step6/署名パス | `varchar(100)` |
|  | `step_three_remarks` | 車検Step3/備考 | `text` |
|  | `step_five_remarks` | 車検Step5/備考 | `text` |
|  | `step_six_remarks` | 車検Step6/備考 | `text` |
|  | `mailing_address` | 車検Step4/送付先データ | `text` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.74

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `type` | 預かり・代車・パーツの種別 | `varchar(50)` |
|  | `drop_off` | 返却日時 | `timestampwithtimezone` |
|  | `pickup` | 預かり日時 | `timestampwithtimezone` |
|  | `car_conditions` | 車両外観の状態のチェックボックス | `jsonb` |
|  | `exterior_conditions` | 車両外観の状態 | `jsonb` |
|  | `fuel_remaining` | 燃料残量 | `integer` |
|  | `notes` | 特記事項 | `text` |
|  | `drop_off_confirmed` | 納車 | `boolean` |
|  | `no_missing_items` | 預かり証のチェックボックスの状態(未選択なし)を保存 | `boolean` |
|  | `drop_off_customer_signature` | 返却時の顧客署名パス | `varchar(100)` |
|  | `pickup_confirmed` | 預かり・代車で、引き取り・返却が確認されたか示す | `boolean` |
|  | `pickup_customer_signature` | 預かり時の顧客署名パス | `varchar(100)` |
|  | `status` | 預かり中・返却済ステータス | `varchar(8)` |
|  | `secondary_customer` | 代車貸出の第2顧客情報 | `jsonb` |
|  | `deposit_amount` | お預かり金額 | `numeric(12,2)` |
|  | `undecided` | パーツ発注の未定チェックボックス | `boolean` |
| FK | `car_data_id` | 車両データID | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_info_id` | 顧客情報ID | `integer` |
| FK | `drop_off_staff_id` | 納車時のスタッフ名称ID | `integer` |
| FK | `pickup_staff_id` | 返却スタッフID | `integer` |
| FK | `substitute_license_plate_id` | 代車ナンバープレートID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |

### テーブル No.75

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | パーツ発注書商品名称 | `varchar(256)` |
|  | `quantity` | 数量 | `numeric(12,2)` |
|  | `group_name` | パーツ発注におけるアイテムのグループ名 | `varchar(256)` |
|  | `price` | 価格 | `numeric(12,2)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `custody_order_id` | 預かり・代車・パーツテーブルに関連するID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.76

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `creation_type` | 作成種別(手動・カーメンテ見積もり) | `smallint` |
|  | `remarks` | 自店舗予約カレンダー備考 | `text` |
|  | `dropoff_datetime` | お預かり日時 | `timestampwithtimezone` |
|  | `pick_up_datetime` | 納車日時 | `timestampwithtimezone` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `customer_info_id` | 顧客情報ID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
| FK | `car_inspection_order_id` | 車検見積もりに関連するID | `integer` |
|  | `car_inspection_order_type` | 車検見積もりが「事前点検予約or車検予約」かを区別 | `smallint` |
|  | `free_input_customer` | 自店舗予約カレンダーでフリー入力された顧客データ | `jsonb` |
| FK | `reservation_staff_id` | 予約担当に関連するID | `integer` |
| FK | `work_staff_id` | 作業担当に関連するID | `integer` |

### テーブル No.77

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `reservation_sequence` | 各予約を並び替えるためのソート順インデックス | `smallint` |
|  | `start_time` | 開始時間 | `time` |
|  | `end_time` | 終了時間 | `time` |
|  | `services` | 予約のサービス情報 | `charactervarying[]` |
|  | `slot_date` | 予約の日付 | `date` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `pit_id` | PITデータに関連するID | `integer` |
| FK | `shop_booking_id` | 自店舗予約に関連するID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.78

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `name` | PIT名称 | `varchar(50)` |
| FK | `shop_id` | 店舗ID | `integer` |
|  | `active` | PITの有効フラグ | `boolean` |
|  | `sequence` | 並び順 | `integer` |

### テーブル No.79

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `uuid` | 各画像に付与する一意の識別テキスト | `uuid` |
|  | `type` | 見積データに紐づく画像の種類スタッフコメント=0,スタッフ署名=1,お客様署名=2,代車/お客様署名=3,代車/スタッフ署名=4,車検Step6/お客様署名=5 | `integer` |
|  | `image` | 画像ファイルパス | `varchar(100)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.80

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `engine_condition` | 車検Step1問診/ポップアップのエンジン項目 | `jsonb` |
|  | `speed_condition` | 車検Step1問診/ポップアップの加速項目 | `jsonb` |
|  | `handle_condition` | 車検Step1問診/ポップアップのハンドル項目 | `jsonb` |
|  | `brake_condition` | 車検Step1問診/ポップアップのブレーキ項目 | `jsonb` |
|  | `other_condition_1` | 車検Step1問診/ポップアップのその他①項目 | `jsonb` |
|  | `other_condition_2` | 車検Step1問診/ポップアップのその他②項目 | `jsonb` |
|  | `special_equipment` | 車検Step1問診/ポップアップの特殊装備項目 | `jsonb` |
|  | `modification` | 車検Step1問診/ポップアップの改造項目 | `jsonb` |
|  | `remarks` | 車検Step1問診/備考 | `text` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `order_id` | 見積もりID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.81

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `memo` | メモ | `text` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `license_plate_id` | ナンバープレートID | `integer` |
| FK | `reminder_id` | 声掛けリストID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.82

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `service_type` | 使用されていない列 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.83

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `type` | 使用されていない列 | `integer` |
|  | `from_day` | 使用されていない列 | `integer` |
|  | `from_period` | 使用されていない列 | `integer` |
|  | `to_day` | 使用されていない列 | `integer` |
|  | `to_period` | 使用されていない列 | `integer` |
| FK | `alert_filter_id` | 使用されていない列 | `integer` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.84

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `select` | アクティブフラグ | `boolean` |
|  | `detail` | 貸出車両ポップアップに表示される補償 | `text` |
|  | `price_1` | 補償金額1 | `numeric(10,2)` |
|  | `price_2` | 補償金額2 | `numeric(10,2)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.85

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `part` | 貸出車両借用書・整備保証書の差し替え箇所 | `varchar` |
|  | `image` | 画像ファイルパス | `varchar(100)` |
|  | `name` | 預かり・代車の帳票名称 | `varchar(255)` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `shop_id` | 店舗ID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

### テーブル No.86

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `name` | 車検実施店名称 | `varchar(255)` |
|  | `email` | メールアドレス | `varchar(255)` |
|  | `address` | 住所 | `varchar(255)` |
|  | `is_active` | 車検実施店有効フラグ | `boolean` |
|  | `drop_off` | 納車 | `smallint` |
|  | `pickup` | 使用されていない列 | `smallint` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `dealer_id` | ディーラーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |
|  | `company_name` | 会社名 | `varchar(255)` |
|  | `phone_number` | 店舗電話番号 | `varchar(255)` |

### テーブル No.87

| キー | カラム名(物理名) | カラム名(論理名) | データ型 |
|----|----|----|----|
| PK | `id` | ID | `integer` |
|  | `created_at` | 作成日時 | `timestampwithtimezone` |
|  | `updated_at` | 更新日時 | `timestampwithtimezone` |
|  | `alive` | 有効フラグ | `boolean` |
|  | `sorting_sequence` | 各ディーラー別Todoリスト項目の並び順を保持するデータ | `jsonb` |
| FK | `created_user_id` | 作成ユーザーID | `integer` |
| FK | `updated_user_id` | 更新ユーザーID | `integer` |

## 15.3. Web機能
##### 一般機能
- ディーラー:ディーラーの管理と構成
<img width="397" height="273" alt="image" src="https://github.com/user-attachments/assets/0fe3dc97-a0bf-41bd-90df-107d0a7bd02d" />

- アナウンス:システムのアナウンスと通知
<img width="366" height="321" alt="image" src="https://github.com/user-attachments/assets/87cbd71b-f51b-4fe0-b4ea-1f732e3f12d5" />

##### マスターデータ機能
- メーカーとモデル：自動車ブランドとモデルの管理
<img width="428" height="166" alt="image" src="https://github.com/user-attachments/assets/db0b4e64-e4ee-4d52-b3d2-6858f7f359bd" />

- 基本的な製品/サービスの推奨：コア製品の推奨構成
<img width="306" height="333" alt="image" src="https://github.com/user-attachments/assets/9e6e2446-a612-431a-86fa-55a55012273a" />

- その他の製品/サービスの推奨:追加の製品推奨管理
<img width="321" height="318" alt="image" src="https://github.com/user-attachments/assets/0c61367f-b860-4906-9c81-e097b8b75619" />

- その他の製品/サービスカテゴリ：製品カテゴリ管理
<img width="150" height="92" alt="image" src="https://github.com/user-attachments/assets/8c60ff63-8e88-48cb-b079-7e9b99214051" />

- 郵便番号アドレス：地理的位置管理
<img width="102" height="153" alt="image" src="https://github.com/user-attachments/assets/766c6c39-32f1-4be7-85b2-fd560536208f" />

##### 購入管理機能
- 購入管理：購入記録の追跡
<img width="428" height="509" alt="image" src="https://github.com/user-attachments/assets/829b98f2-2520-4536-9ce2-c219009e4852" />

- アンケート履歴：アンケート回答履歴と分析
<img width="425" height="204" alt="image" src="https://github.com/user-attachments/assets/fb4f01d1-2c1c-4069-9599-debe61c61b23" />

##### 注文管理機能
- 注文履歴分析：注文データの分析とレポート
<img width="425" height="316" alt="image" src="https://github.com/user-attachments/assets/5949f9f5-e235-4417-ac4f-4b5d99799e6c" />

- 帳票発行履歴：帳票発行履歴の追跡と管理
<img width="428" height="472" alt="image" src="https://github.com/user-attachments/assets/7a080458-c840-4ab5-a2d5-bfed4814ad3e" />

##### 売上分析機能
- リペア分析：リペアデータ分析
<img width="357" height="289" alt="image" src="https://github.com/user-attachments/assets/7fde8b6e-1640-4489-bf2c-90daeaede3d3" />

- 日次/月次売上実績：期間別売上履歴
<img width="425" height="315" alt="image" src="https://github.com/user-attachments/assets/111665ac-ceb3-43a3-a6ac-0ad4b0c9550f" />

##### 顧客登録分析機能
- 日次/月次顧客登録実績:定期的な顧客獲得指標
<img width="425" height="147" alt="image" src="https://github.com/user-attachments/assets/1ef26463-ddb9-4c78-b5cf-19338c48e273" />

##### ショップ情報設定機能
- タブレットユーザー：タブレットユーザーアカウント管理
<img width="425" height="276" alt="image" src="https://github.com/user-attachments/assets/ceadce4f-6ad5-46dd-8c51-0e81c11e0d7a" />

- Webユーザー:Webユーザーアカウント管理
<img width="425" height="276" alt="image" src="https://github.com/user-attachments/assets/28f9c7fe-6b9a-4131-a64f-14decd31297f" />

- ショップ：ショップの構成と管理
<img width="284" height="327" alt="image" src="https://github.com/user-attachments/assets/be7474b3-ebe7-44c4-912f-dad887e6c703" />

- アンケート作成：調査の設計と作成
<img width="425" height="219" alt="image" src="https://github.com/user-attachments/assets/388cdfa7-5898-424f-aa22-7486bfd959ed" />

- アンケートメニュー：アンケートメニューの設定
<img width="425" height="297" alt="image" src="https://github.com/user-attachments/assets/75b1d0a3-dc75-4ab9-b2b5-994af8238860" />

- Todoリスト：Todoリスト管理
<img width="425" height="122" alt="image" src="https://github.com/user-attachments/assets/56e53027-bef7-452d-847c-1f2368e02a87" />

- 声掛けリスト：声掛けリスト機能の設定
<img width="425" height="171" alt="image" src="https://github.com/user-attachments/assets/e5cf42e8-95ce-4f35-8c04-824a72a7792d" />

- 貸出車両借用書・整備保証書：貸出車両借用書・整備保証書管理
<img width="343" height="285" alt="image" src="https://github.com/user-attachments/assets/4028e33a-2a6c-4a29-89c5-1ed8712384b4" />

##### マスターデータ設定機能
- サービス:サービスタイプの構成
<img width="343" height="280" alt="image" src="https://github.com/user-attachments/assets/11eeda86-c74f-4a02-b144-c11311c75f04" />

- 商品:商品マスター管理
<img width="425" height="299" alt="image" src="https://github.com/user-attachments/assets/3c5a4a92-fb92-46aa-a96c-dfb2d3dcecb3" />

- 顧客：顧客データ管理
<img width="425" height="204" alt="image" src="https://github.com/user-attachments/assets/2931c2cd-e703-4b7e-9332-3d226f0164b8" />

##### インポート/エクスポート機能
- インポート結果:インポート操作の追跡
<img width="425" height="230" alt="image" src="https://github.com/user-attachments/assets/d0d403a0-045e-4c2e-9f25-2dd2146e9e04" />

- エクスポート結果：エクスポート操作の追跡
<img width="349" height="247" alt="image" src="https://github.com/user-attachments/assets/5de0ad46-8b7f-4f3d-ab5b-baf2a920c940" />

##### 工場機能
- 車検実施店設定：車検実施店設定管理
<img width="355" height="331" alt="image" src="https://github.com/user-attachments/assets/3a9b685b-3f83-4ee4-8f9b-9f2d37a9f1f9" />

- 工場：工場管理
<img width="425" height="276" alt="image" src="https://github.com/user-attachments/assets/cb06b36a-8e2d-4954-bde9-af02f84b04ae" />

- 工場ユーザー：工場ユーザー管理
<img width="425" height="313" alt="image" src="https://github.com/user-attachments/assets/ddace0e3-b44c-42ba-816c-b4aa5a307f94" />

- 工場予約：予約と注文状況の追跡
<img width="425" height="285" alt="image" src="https://github.com/user-attachments/assets/bb865fca-b1a6-4c09-937f-7e290515dfb7" />

- 1日あたりの容量設定：1日あたりの最大容量（タッチ＆カー）を管理
<img width="261" height="181" alt="image" src="https://github.com/user-attachments/assets/843d2b8f-9b49-48dd-80b4-f5060cafeafb" />

- 工場カレンダー：週/月カレンダーの予約ダッシュボード
<img width="255" height="177" alt="image" src="https://github.com/user-attachments/assets/049871c0-f40d-4425-882a-2cd46f1b2a11" />

##### その他
- 受注：購入履歴分析をエクスポート
<img width="428" height="331" alt="image" src="https://github.com/user-attachments/assets/5ce326fd-0252-4dc9-af81-ae24b92c9899" />

- 受注/購買履歴:注文状況やその他の詳細の追跡
<img width="425" height="378" alt="image" src="https://github.com/user-attachments/assets/47da084e-7ee5-4e7f-b862-e3711778c70e" />

- 台数：顧客の車のダッシュボードと管理
<img width="425" height="258" alt="image" src="https://github.com/user-attachments/assets/11888008-105f-458d-8f8e-9548514bdd72" />

- 販売履歴：販売データダッシュボード
<img width="408" height="302" alt="image" src="https://github.com/user-attachments/assets/a3fa9977-19a3-4412-b7a3-57a157e6be0b" />

- 車両情報/顧客情報:車両情報/顧客情報
<img width="425" height="258" alt="image" src="https://github.com/user-attachments/assets/0adb4513-2c32-4ff6-a3a6-7b9ced179a40" />

- 新規獲得記録：新規顧客獲得の追跡
<img width="425" height="315" alt="image" src="https://github.com/user-attachments/assets/d97f5afd-a151-4c24-89f9-fb0d6a9eb353" />

- 購入回数：購入頻度分析
<img width="425" height="273" alt="image" src="https://github.com/user-attachments/assets/39100db1-3656-471e-8bb4-e19edfcfd415" />

- 利用店舗数：店舗利用率指標
<img width="425" height="273" alt="image" src="https://github.com/user-attachments/assets/e8b08aa5-4727-4687-bafe-6141bcdad6d5" />

- 製品購入：製品購入分析
<img width="425" height="315" alt="image" src="https://github.com/user-attachments/assets/ffccd7a4-99e4-4663-b27e-471311637db4" />

- 顧客登録：顧客登録管理
<img width="425" height="273" alt="image" src="https://github.com/user-attachments/assets/ec1819f7-6ed7-4185-bd08-ff64b9a6a67b" />

- ログ:システムログとアクティビティの追跡
<img width="377" height="352" alt="image" src="https://github.com/user-attachments/assets/b97a0c94-29dc-4465-9c62-1ca0808d0e24" />

## 15.4. iOS機能
##### 認証およびログイン機能
<img width="425" height="293" alt="image" src="https://github.com/user-attachments/assets/fe66e362-143c-4f3e-b28d-e77a328e42de" />

##### ホームと検索機能
<img width="425" height="147" alt="image" src="https://github.com/user-attachments/assets/356e204d-ba52-4bc8-8299-c010f67ae956" />

##### カレンダーと予約機能
<img width="425" height="276" alt="image" src="https://github.com/user-attachments/assets/e023d862-ed82-483c-b159-3a34e09071ed" />

##### リペアカレンダー機能
<img width="425" height="285" alt="image" src="https://github.com/user-attachments/assets/c441b7bd-8353-429b-94a5-da4de5232b63" />

##### アクションリスト機能
<img width="445" height="168" alt="image" src="https://github.com/user-attachments/assets/87303d3e-24dd-4f4c-8f66-6c544441802a" />

##### お知らせ機能
<img width="403" height="339" alt="image" src="https://github.com/user-attachments/assets/a41a0245-732e-4a96-bb31-50369ae5e1ec" />

##### 設定機能
<img width="425" height="293" alt="image" src="https://github.com/user-attachments/assets/175e5922-42cc-4e2a-ae42-a4334a6d652c" />

##### 顧客/車両検索機能
<img width="425" height="147" alt="image" src="https://github.com/user-attachments/assets/724c18d5-dcd4-477c-b689-a5f6f6a8f8ac" />

##### 携行缶詰替え販売機能
<img width="414" height="310" alt="image" src="https://github.com/user-attachments/assets/5b2192e8-74b0-4436-af03-7c0d5dacab59" />

##### カーメンテナンス機能
<img width="414" height="320" alt="image" src="https://github.com/user-attachments/assets/d8a11aaf-518d-47b2-bedb-946f69d4413a" />

##### 安全点検機能
<img width="445" height="354" alt="image" src="https://github.com/user-attachments/assets/c1aac938-f690-4b00-8e18-2e3c71bb036b" />

##### 車両お預かり証機能
<img width="445" height="414" alt="image" src="https://github.com/user-attachments/assets/9ee5823d-8f9b-45d0-beb7-79b49b4f40b4" />

##### 代車貸出証機能
<img width="445" height="414" alt="image" src="https://github.com/user-attachments/assets/9431c5eb-6fee-40be-bc8c-a2da4e5432ec" />

##### 部品発注書機能
<img width="448" height="417" alt="image" src="https://github.com/user-attachments/assets/d44bb117-9c3d-4af7-be48-d7405ec6ebf5" />

##### 車両/顧客情報機能
<img width="454" height="157" alt="image" src="https://github.com/user-attachments/assets/2ecf9147-187f-4131-9896-802db258978e" />

##### 顧客情報機能
<img width="445" height="154" alt="image" src="https://github.com/user-attachments/assets/609fc927-1c47-4149-9cd9-61d43cb47a86" />

##### 車両履歴詳細機能
- Todoリスト
<img width="445" height="261" alt="image" src="https://github.com/user-attachments/assets/d3325576-6ae2-4e79-b3c3-2b0397f1e35b" />

- 声掛けリスト
<img width="445" height="380" alt="image" src="https://github.com/user-attachments/assets/197b82fe-f337-4244-9909-82fb25e242a5" />

##### 車検機能
<img width="445" height="526" alt="image" src="https://github.com/user-attachments/assets/2ed5442a-cf9f-44bc-8c61-51116858b911" />

##### 見積もり概要機能
<img width="425" height="329" alt="image" src="https://github.com/user-attachments/assets/0444d608-12b2-4272-a45c-ad8e578ed7a4" />

