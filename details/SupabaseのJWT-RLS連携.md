---
layer: 3
parent: "[[RLS（Row-Level-Security）]]"
type: detail
created: 2026-03-31
---

# Supabase の JWT-RLS 連携（JWT-based Row Level Security）

> **一言で言うと:** Supabase は PostgREST を介して JWT のクレームを PostgreSQL のセッション変数に注入し、RLS ポリシーがそれを参照することで、バックエンドサーバーを書かずに行レベルのアクセス制御を実現する。認可ロジックがアプリケーションコードから DB ポリシーに移動するため、WHERE 句の付け忘れという構造的なセキュリティホールが原理的に発生しない。

## 全体像 — リクエストの流れ

従来のアーキテクチャでは「クライアント → バックエンド → DB」の経路で、バックエンドが認可判断を行う。Supabase ではクライアントが PostgREST（REST API 層）を通じて PostgreSQL に直接アクセスし、RLS が認可を強制する。

```mermaid
sequenceDiagram
    participant Client as クライアント
    participant Auth as Supabase Auth
    participant API as PostgREST
    participant DB as PostgreSQL

    Client->>Auth: ログイン（email/password, OAuth等）
    Auth-->>Client: JWT（sub, role, app_metadata 等）

    Client->>API: GET /rest/v1/orders<br/>Authorization: Bearer <JWT>
    API->>API: JWT 署名検証 + 有効期限確認
    API->>DB: SET LOCAL request.jwt.claims = '{...}'
    API->>DB: SET LOCAL ROLE authenticated
    API->>DB: SELECT * FROM orders
    DB->>DB: RLS ポリシー評価<br/>auth.uid() でJWTのsubを参照
    DB-->>API: フィルタ済みの行のみ
    API-->>Client: JSON レスポンス
```

重要なのは、PostgREST が **JWT を検証した後、そのクレーム全体を PostgreSQL のセッション変数にセットする** 点である。これにより、SQL の世界から JWT の情報を参照できるようになる。

## PostgREST の役割

PostgREST は Haskell で書かれた REST API サーバーで、PostgreSQL のスキーマから自動的に RESTful API を生成する。Supabase のデータアクセス層の中核を担う。

### PostgREST が各リクエストで行うこと

```sql
-- 1. JWT のクレーム全体をセッション変数に格納
SELECT set_config('request.jwt.claims',
  '{"sub":"d0a1b2c3-...","role":"authenticated","email":"user@example.com","app_metadata":{"tenant_id":42}}',
  true  -- トランザクションローカル（SET LOCAL 相当）
);

-- 2. JWT の role クレームに基づいて PostgreSQL ロールを切り替え
SET LOCAL ROLE authenticated;

-- 3. クライアントのリクエストに対応する SQL を実行
SELECT * FROM orders;
-- → RLS ポリシーが自動適用される

-- 4. トランザクション終了時にロールと変数が自動リセットされる
```

`set_config` の第3引数 `true` は「現在のトランザクション内でのみ有効」を意味する。トランザクションが終了すれば変数は自動的に消えるため、[[コネクションプール]]でのリセット忘れ問題が構造的に発生しない。これは [[RLS（Row-Level-Security）]] の「落とし穴3: `current_setting` のリセット忘れ」に対する Supabase の解決策でもある。

### Supabase のロール体系

| ロール | 用途 | RLS の適用 |
|---|---|---|
| `anon` | 未認証ユーザー（公開 API） | 適用される |
| `authenticated` | 認証済みユーザー | 適用される |
| `service_role` | サーバーサイド管理操作 | **バイパスする** |
| `supabase_admin` | Supabase 内部管理 | バイパスする |

`service_role` キーは RLS をバイパスするため、**絶対にクライアントに露出させてはならない**。サーバーサイドの管理操作（Webhook ハンドラ、バッチ処理等）でのみ使用する。

## `auth.uid()` の内部実装

Supabase が提供する `auth.uid()` は、PostgreSQL の関数として定義されている:

```sql
-- Supabase の auth スキーマに定義されている関数（簡略化）
CREATE OR REPLACE FUNCTION auth.uid() RETURNS uuid AS $$
  SELECT nullif(
    current_setting('request.jwt.claims', true)::json->>'sub',
    ''
  )::uuid;
$$ LANGUAGE sql STABLE;
```

本質的には **セッション変数 `request.jwt.claims` から `sub` クレームを取り出しているだけ** である。`sub` が空文字列の場合は `nullif` により NULL を返し、不正な UUID キャストを防ぐ。[[RLS（Row-Level-Security）]] で学んだ `current_setting('app.current_tenant')` と同じ仕組みをJWT のクレーム全体に拡張したものと理解できる。

同様に、他のクレームも `auth.jwt()` で取得できる:

```sql
-- JWT のクレーム全体を JSON として取得
CREATE OR REPLACE FUNCTION auth.jwt() RETURNS json AS $$
  SELECT current_setting('request.jwt.claims', true)::json;
$$ LANGUAGE sql STABLE;

-- ポリシーでの使用例: app_metadata 内の tenant_id を参照
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    USING (tenant_id = (auth.jwt()->'app_metadata'->>'tenant_id')::int);
```

## 安全性の根拠

「クライアントが DB に直接アクセスして安全なのか？」という疑問に対する回答を、攻撃ベクトルごとに整理する。

### 多層防御の構造

```mermaid
flowchart TB
    subgraph "攻撃者の視点"
        A1["JWT を改ざんしたい"] --> D1["署名検証で失敗<br/>（秘密鍵がなければ署名を生成できない）"]
        A2["RLS をバイパスしたい"] --> D2["authenticated ロールには<br/>BYPASSRLS 権限がない"]
        A3["生 SQL を実行したい"] --> D3["PostgREST は REST API のみ公開<br/>SQL インジェクションの余地がない"]
        A4["期限切れ JWT を使いたい"] --> D4["PostgREST が exp クレームを検証<br/>期限切れなら 401"]
        A5["他人の行を読みたい"] --> D5["RLS ポリシーが auth.uid() で<br/>自動フィルタ"]
        A6["service_role キーを使いたい"] --> D6["クライアントに露出していなければ<br/>取得不可能"]
    end

    style D1 fill:#c8e6c9,stroke:#2e7d32
    style D2 fill:#c8e6c9,stroke:#2e7d32
    style D3 fill:#c8e6c9,stroke:#2e7d32
    style D4 fill:#c8e6c9,stroke:#2e7d32
    style D5 fill:#c8e6c9,stroke:#2e7d32
    style D6 fill:#c8e6c9,stroke:#2e7d32
```

### 安全性の前提条件

この安全モデルが成立するには、以下の前提が必要である:

| 前提 | 破られた場合のリスク | 対策 |
|---|---|---|
| JWT 署名の秘密鍵が漏洩していない | 任意のユーザーになりすませる | 環境変数で管理し、ローテーションの仕組みを用意する |
| `service_role` キーがクライアントに露出していない | RLS が完全にバイパスされる | サーバーサイドのみで使用し、フロントエンドのコードに含めない |
| 全テーブルに RLS が有効化されている | ポリシー未設定のテーブルは全行が露出する | マイグレーション時にRLS 有効化を必須チェックとする |
| ポリシーのロジックに抜けがない | 意図しない行が見える | ポリシーのテストを自動化する（後述） |

## 責務の分離 — 従来型との比較

### 従来のバックエンド認可

```mermaid
flowchart LR
    subgraph "従来型"
        C1[クライアント] --> BE[バックエンド<br/>認証 + 認可 + ビジネスロジック]
        BE --> DB1[(DB)]
    end
```

認可ロジックがバックエンドのアプリケーションコード内に散在する:

```typescript
// 従来型: 認可ロジックがアプリケーションコードに埋め込まれている
app.get("/api/orders", async (req, res) => {
  const user = req.user; // 認証ミドルウェアで解決済み

  // 認可チェック — ここを書き忘れるとデータ漏洩
  const orders = await db.query(
    "SELECT * FROM orders WHERE tenant_id = $1",
    [user.tenantId]
  );

  res.json(orders);
});

// 別のエンドポイントで WHERE を忘れると全テナントのデータが見える
app.get("/api/reports", async (req, res) => {
  // バグ: tenant_id のフィルタを忘れている
  const reports = await db.query("SELECT * FROM reports");
  res.json(reports);
});
```

### Supabase の RLS 認可

```mermaid
flowchart LR
    subgraph "Supabase型"
        C2[クライアント] -->|JWT| API2[PostgREST<br/>JWT検証 + SQL変換]
        API2 --> DB2[(DB<br/>RLS で認可)]
    end
```

認可ロジックが DB 層のポリシーに集約される:

```sql
-- RLS 型: 認可ロジックは DB のポリシーに1箇所で定義
CREATE POLICY tenant_isolation ON orders
    FOR ALL TO authenticated
    USING (tenant_id = (auth.jwt()->'app_metadata'->>'tenant_id')::int);

-- reports テーブルにも同様のポリシーを設定すれば
-- エンドポイントを追加しても認可漏れは発生しない
CREATE POLICY tenant_isolation ON reports
    FOR ALL TO authenticated
    USING (tenant_id = (auth.jwt()->'app_metadata'->>'tenant_id')::int);
```

### 責務の比較

| 責務 | 従来型 | Supabase + RLS |
|---|---|---|
| **認証**（誰か） | バックエンドのミドルウェア | Supabase Auth + JWT |
| **認可**（何ができるか） | バックエンドのコード（WHERE 句） | PostgreSQL の RLS ポリシー |
| **データアクセス** | バックエンドの ORM / クエリ | PostgREST が自動生成 |
| **ビジネスロジック** | バックエンドのコード | Edge Functions / DB関数 |
| **入力バリデーション** | バックエンドのコード | DB 制約 + PostgREST の型チェック |

重要な観点: Supabase 型は「認可の責務を DB に移動させた」のであって、「バックエンドを不要にした」わけではない。複雑なビジネスロジックが必要な場合は、Supabase Edge Functions（Deno ベースのサーバーレス関数）やデータベース関数（PL/pgSQL）で補完する。

## コード例

### TypeScript — Supabase クライアントからのデータアクセス

```typescript
import { createClient } from "@supabase/supabase-js";

// anon キーはクライアントに公開しても安全（RLS が保護するため）
const supabase = createClient(
  "https://your-project.supabase.co",
  "eyJhbGciOi...（anon key）"
);

// ログイン → JWT が自動管理される
await supabase.auth.signInWithPassword({
  email: "user@example.com",
  password: "password",
});

// RLS により自分のデータだけが返る（WHERE 句は不要）
const { data: orders, error } = await supabase
  .from("orders")
  .select("id, product_name, amount")
  .order("created_at", { ascending: false });

// 他人の行を明示的に指定しても RLS がブロックする
const { data } = await supabase
  .from("orders")
  .select("*")
  .eq("user_id", "other-user-uuid"); // → 空配列が返る
```

### Go — サーバーサイドでの service_role 利用

```go
package main

import (
	"context"
	"fmt"
	"log"

	"github.com/jackc/pgx/v5"
)

func main() {
	// service_role はサーバーサイドでのみ使用（RLS をバイパスする）
	conn, err := pgx.Connect(context.Background(),
		"postgres://postgres.your-project:password@db.your-project.supabase.co:5432/postgres")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close(context.Background())

	// service_role 接続では RLS が適用されない — 管理操作に使用
	rows, err := conn.Query(context.Background(),
		"SELECT id, tenant_id, product_name FROM orders LIMIT 10")
	if err != nil {
		log.Fatal(err)
	}
	defer rows.Close()

	for rows.Next() {
		var id, tenantID int64
		var name string
		if err := rows.Scan(&id, &tenantID, &name); err != nil {
			log.Fatal(err)
		}
		fmt.Printf("Order %d (tenant %d): %s\n", id, tenantID, name)
	}
}
```

## RLS ポリシーのテスト

ポリシーのロジックに抜けがないことを保証するため、テストを自動化する:

```sql
-- pgTAP を使った RLS ポリシーのテスト例
BEGIN;
SELECT plan(3);

-- テナント42としてセッションを設定
SET LOCAL ROLE authenticated;
SELECT set_config('request.jwt.claims',
  '{"sub":"user-1","role":"authenticated","app_metadata":{"tenant_id":"42"}}',
  true);

-- テスト1: 自テナントの行が見える
SELECT isnt_empty(
  'SELECT * FROM orders WHERE tenant_id = 42',
  'テナント42の行が見えること'
);

-- テスト2: 他テナントの行が見えない
SELECT is_empty(
  'SELECT * FROM orders WHERE tenant_id = 99',
  'テナント99の行が見えないこと'
);

-- テスト3: 他テナントへの INSERT が拒否される
SELECT throws_ok(
  'INSERT INTO orders (tenant_id, product_name, amount) VALUES (99, ''test'', 100)',
  'new row violates row-level security policy for table "orders"',
  '他テナントへの挿入が拒否されること'
);

SELECT finish();
ROLLBACK;
```

## よくある落とし穴

### 1. anon キーと service_role キーの混同

Supabase は2種類の API キーを発行する:

| キー | 公開可否 | RLS | 用途 |
|---|---|---|---|
| `anon` | 公開可 | 適用される | クライアントサイド |
| `service_role` | **非公開** | バイパスする | サーバーサイド |

`service_role` キーをフロントエンドのコードや Git リポジトリに含めると、RLS による保護が完全に無効化される。

### 2. RLS を有効化せずにテーブルを作成する

Supabase のダッシュボード（Table Editor）から作成したテーブルは現在デフォルトで RLS が有効化されるが、SQL で直接 `CREATE TABLE` を実行した場合は PostgreSQL のデフォルト（RLS 無効）が適用される。RLS が無効なテーブルにはダッシュボード上で警告バナーが表示される。いずれの方法でも、マイグレーション時に RLS の有効化を確認する習慣をつけること。

```sql
-- マイグレーション時のチェックリスト
ALTER TABLE new_table ENABLE ROW LEVEL SECURITY;
-- ポリシーを作成するまで、デフォルト拒否で全行が見えなくなる（安全側）
```

### 3. ポリシー内でのパフォーマンス問題

`auth.uid()` は単純なセッション変数の参照なので高速だが、ポリシー内でサブクエリを使うと全クエリに影響する:

```sql
-- 遅い: ポリシーが毎回サブクエリを実行する
CREATE POLICY team_access ON documents
    FOR SELECT TO authenticated
    USING (
      team_id IN (
        SELECT team_id FROM team_members
        WHERE user_id = auth.uid()  -- 毎行評価される
      )
    );

-- 改善: team_members に適切なインデックスを追加
CREATE INDEX idx_team_members_user_id ON team_members(user_id);
```

### 4. JWT の有効期限内は権限変更が反映されない

ユーザーのロールや `app_metadata` を変更しても、既存の JWT には古い情報が含まれたままである。[[セッションとJWT]] で学んだ通り、JWT はステートレスであるため即時無効化ができない。Supabase のデフォルトの JWT 有効期限は3600秒（1時間）である。

## 関連トピック

- [[RLS（Row-Level-Security）]] — 親トピック。PostgreSQL の RLS の基本構文と仕組み
- [[セッションとJWT]] — JWT の構造、署名検証、ステートレス認証の仕組み
- [[OAuth2とOpenID-Connect]] — Supabase Auth が内部で利用する認証プロトコル
- [[コネクションプール]] — PostgREST の `SET LOCAL` によるセッション変数管理

## 参考リソース

- [Supabase 公式: Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security) — ポリシー設計の実践ガイド
- [Supabase 公式: Auth Architecture](https://supabase.com/docs/guides/auth/architecture) — GoTrueとJWTの発行フロー
- [PostgREST 公式ドキュメント](https://docs.postgrest.org/) — JWT 検証とロール切り替えの仕組み
- [pgTAP](https://pgtap.org/) — PostgreSQL のユニットテストフレームワーク
