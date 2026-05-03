---
layer: 3
parent: "[[RDB]]"
type: detail
created: 2026-05-03
---

# PostgREST（PostgreSQL の REST/RPC エンドポイント自動生成）

> **一言で言うと:** PostgreSQL のスキーマ情報をリフレクションし、テーブル/ビューを **REST リソース**として、SQL 関数を `/rpc/` 経由で **RPC**として、自動的に HTTP で公開する Haskell 製の API サーバー。バックエンドコードを書かずに DB を HTTP に晒し、認可は [[RLS（Row-Level-Security）]]、ビジネスロジックは SQL 関数、HTTP の配線は PostgREST が担う「DB-First」アーキテクチャの中核となる。Supabase が PostgREST を内部で使用していることで一気に普及した。

## 名前の混乱を先に整理

「PostgreRPC」「PostgREST」「PostgRPC」「pREST」「Hasura」と似た名前が並ぶが、それぞれ別物である。日本語圏で「PostgreRPC」と呼ばれる場合、ほぼ確実に PostgREST の `/rpc/` エンドポイント機能のことを指している（PostgREST の REST 機能と RPC 機能を合わせて「Postgres + RPC」と短縮した呼び名が独り歩きしたもの）。

| 名前 | 言語 | 公開プロトコル | 主な特徴 |
|---|---|---|---|
| **PostgREST** | Haskell | REST + `/rpc/` で関数呼び出し | デファクト。Supabase の中核データレイヤー |
| pREST | Go | REST | スキーマ自動探索が強力。DBA 向けツールに近い |
| PostgRPC | Rust | gRPC | gRPC 経由で SQL を実行する少数派プロジェクト |
| Hasura | Haskell（v2 GraphQL Engine）/ Rust（v3 DDN） | GraphQL（+ REST） | クエリ言語が GraphQL なため設計思想は異なる |

本ドキュメントは PostgREST を扱う。GraphQL での同等品としては [[API設計-REST-GraphQL|Hasura]] を参照すると対比が明確になる。

## なぜ PostgREST が生まれたか

伝統的な Web API の構築では、以下のような薄い層を**毎回手書き**していた:

```mermaid
flowchart LR
    Client[クライアント] --> Route["ルーティング<br/>(Express/Rails/Laravel)"]
    Route --> Auth["認証ミドルウェア"]
    Auth --> Authz["認可チェック<br/>(WHERE句やif文)"]
    Authz --> Validate["バリデーション"]
    Validate --> ORM["ORM/クエリビルダ"]
    ORM --> SQL[("PostgreSQL")]

    style Authz fill:#ffcdd2,stroke:#b71c1c
    style ORM fill:#fff3e0,stroke:#e65100
```

これらの層の大半は「クライアントからの HTTP リクエストを SQL に変換する」だけの**機械的な対応関係**で、ビジネスロジックを含まないコードであることが多い。PostgREST は「**この対応関係を全て自動化し、宣言的な手書きが必要なものは DB スキーマと SQL 関数だけ**」というラディカルな設計を採用した。

```mermaid
flowchart LR
    Client[クライアント] -->|"GET /users?id=eq.42"| PR["PostgREST<br/>(URL → SQL 変換)"]
    PR -->|"JWT 検証 + SET LOCAL ROLE"| PG[("PostgreSQL<br/>RLS で認可")]
    PG -->|"フィルタ済み行"| PR
    PR -->|"JSON レスポンス"| Client

    style PR fill:#c8e6c9,stroke:#2e7d32
```

## アーキテクチャ — リクエストの流れ

PostgREST が 1 リクエストごとに行う処理を時系列で見ると、薄いプロキシでありながら認証・認可・SQL 生成・型変換まで一貫して担っているのがわかる。

```mermaid
sequenceDiagram
    participant C as クライアント
    participant PR as PostgREST
    participant PG as PostgreSQL

    C->>PR: GET /orders?status=eq.paid<br/>Authorization: Bearer <JWT>
    PR->>PR: JWT 署名検証 + exp 確認
    PR->>PG: BEGIN
    PR->>PG: SET LOCAL ROLE authenticated
    PR->>PG: SET LOCAL request.jwt.claims = '{...}'
    PR->>PG: SELECT * FROM orders WHERE status = 'paid'
    PG->>PG: RLS ポリシー適用<br/>(auth.uid() を参照)
    PG-->>PR: 該当行のみ
    PR->>PG: COMMIT
    PR-->>C: JSON レスポンス + Content-Range ヘッダ
```

重要なのは、PostgREST が **JWT のクレームを PostgreSQL のセッション変数に注入する**点である。これにより SQL 側（RLS ポリシーや関数）から JWT の情報を `current_setting('request.jwt.claims', true)` で参照できる。詳細は [[SupabaseのJWT-RLS連携]] を参照。

## REST リソースの自動マッピング

PostgREST はテーブル/ビューを REST リソースとして自動公開する。HTTP メソッドと SQL 操作の対応は以下:

| HTTP | パス例 | 生成される SQL | 備考 |
|---|---|---|---|
| `GET` | `/users` | `SELECT * FROM users` | 全件取得 |
| `GET` | `/users?id=eq.42` | `SELECT * FROM users WHERE id = 42` | フィルタ |
| `GET` | `/users?select=id,name` | `SELECT id, name FROM users` | カラム選択 |
| `GET` | `/users?order=created_at.desc` | `... ORDER BY created_at DESC` | ソート |
| `GET` | `/users` + `Range: 0-9` | `... LIMIT 10 OFFSET 0` | ページネーション |
| `POST` | `/users` | `INSERT INTO users ...` | bodyが行データ |
| `PATCH` | `/users?id=eq.42` | `UPDATE users SET ... WHERE id = 42` | 部分更新 |
| `DELETE` | `/users?id=eq.42` | `DELETE FROM users WHERE id = 42` | 削除 |

### フィルタ演算子の URL 表現

`eq` `gt` `lt` `like` `in` `is` などの演算子をクエリパラメータで指定する独自の DSL が用意されている。

```
GET /products?price=gte.1000&price=lt.5000&category=in.(book,music)
```

| クエリパラメータ | 生成される SQL 条件 |
|---|---|
| `price=gte.1000` | `price >= 1000` |
| `price=lt.5000` | `price < 5000` |
| `category=in.(book,music)` | `category IN ('book','music')` |

### JOIN の代替 — Resource Embedding

PostgREST は外部キー制約を読み取り、`select=table(*)` 構文で関連テーブルを埋め込む（裏側は LATERAL JOIN または別クエリ）。

```
GET /orders?select=*,customers(name,email)
```

これにより、N+1 問題を発生させずに関連データを取得できる。GraphQL のフィールド選択に近い感覚で書ける点が特徴。

## RPC とストアドプロシージャ — 2 つの背景概念

PostgREST の `/rpc/` を理解するには、**RPC（Remote Procedure Call）**という分散システムのパラダイムと、**ストアドプロシージャ（Stored Procedure）**という DB の機能を、それぞれ独立した概念として押さえておくと整理しやすい。`/rpc/` はこの 2 つの交差点に位置する。

### RPC（Remote Procedure Call）とは

リモートのサーバー上にある関数（手続き）を、**ローカル関数を呼ぶのと同じ書き味で呼び出すパラダイム**。1984 年の Birrell & Nelson "Implementing Remote Procedure Calls" 論文で概念化され、1980 年代後半の Sun RPC（NFS の基盤、RFC 1057 が 1988 年）を経て、CORBA / DCOM / SOAP と進化した。2000 年代に REST が広く採用されると一旦下火になったが、2015 年の gRPC で再評価された。

```mermaid
timeline
    title RPC プロトコルの系譜
    1984 : Birrell & Nelson 論文 (RPC の概念化)
    1988 : Sun RPC / ONC RPC (RFC 1057, NFS の基盤)
    1991 : CORBA 1.0 (オブジェクト指向 RPC)
    1996 : DCOM (Windows 用)
    1998 : SOAP / XML-RPC (XML-based)
    2000 : Roy Fielding "REST" 博士論文
    2005 : JSON-RPC 1.0
    2010 : JSON-RPC 2.0 (名前付き引数・エラーコード標準化)
    2015 : gRPC (HTTP/2 + Protobuf で復活)
```

REST との本質的な違いは「**動詞中心 vs リソース中心**」のモデリング:

| 観点 | REST | RPC |
|---|---|---|
| 中心概念 | リソース（名詞） | 手続き（動詞） |
| URL の意味 | リソースの識別子 | 関数名 |
| 例 | `POST /transactions`<br/>（取引リソースを作成） | `POST /Bank/transfer`<br/>（送金関数を呼ぶ） |
| HTTP メソッド | リソース操作の意味（GET/POST/PUT/DELETE） | 副作用の有無のみ（多くは POST 一択） |
| 状態 | ステートレスが原則 | サーバー側状態を扱うこともある |
| データ表現 | JSON / XML / HAL など | Protobuf / JSON-RPC / Msgpack など |

主な RPC プロトコル:

| プロトコル | エンコード | トランスポート | 主な利用シーン |
|---|---|---|---|
| Sun RPC / ONC RPC | XDR | TCP/UDP | NFS の基盤 |
| SOAP | XML | HTTP | エンタープライズ系（旧来の B2B 連携） |
| JSON-RPC 2.0 | JSON | HTTP / WebSocket | Ethereum クライアント、Language Server Protocol |
| **gRPC** | Protobuf | HTTP/2 | マイクロサービス間通信 |
| **PostgREST `/rpc/`** | JSON | HTTP/1.1+ | DB 関数の HTTP 公開 |

### ストアドプロシージャ（Stored Procedure）とは

**DB サーバー側に保存され、DB エンジン内で実行される手続き**。1980 年代後半に Sybase Transact-SQL（1987 年）、Oracle PL/SQL（1989 年）など複数のベンダーで並行して実装され、SQL/PSM として **1996 年に標準化**された（Microsoft SQL Server は Sybase からの派生で T-SQL 系列）。アプリケーションコードではなく DB 内に「ロジックを置く」設計思想で、当時のクライアント/サーバーアーキテクチャでは標準的なパターンだった。

#### メリットとデメリット

| 観点 | メリット | デメリット |
|---|---|---|
| 性能 | DB に近接実行されるため N+1 やネットワーク往復を削減できる | DB の CPU/メモリを消費する（スケール困難） |
| 整合性 | 複雑なトランザクションを 1 関数で原子的に実行できる | 関数内のロジックがブラックボックス化しやすい |
| 認可 | EXECUTE 権限の単一窓口で集中管理できる | 権限設計が複雑になりやすい |
| 開発 | SQL の表現力をフルに使える | バージョン管理・テスト・デバッグの**ツール不足** |
| 移植性 | — | PL/pgSQL / T-SQL / PL/SQL は方言が大きく**ベンダーロック**が強い |

#### なぜ 1990〜2000 年代に廃れ、2010 年代以降に再評価されたか

ストアドプロシージャは以下の理由で 90 年代後半〜2000 年代に主役の座を ORM / アプリケーションコードに譲った:

- ベンダーロックが強く DB 移行の足枷になる
- バージョン管理がアプリケーションコードと別系統になり**マイグレーション**が破綻しがち
- ユニットテストやモックがしづらく CI に乗せにくい
- IDE のサポートが弱く可読性も低かった

しかし 2010 年代以降、以下の流れで「**安全側に倒したストアドプロシージャ活用**」が再評価された:

- PostgreSQL の言語ハンドラ（PL/pgSQL / PL/Python / plv8 など）の充実
- Sqitch / Flyway / dbmate などスキーマと関数を版管理するマイグレーションツールの普及
- pgTAP のような DB 内テストフレームワーク
- Supabase / PostgREST により「アプリケーションサーバー不要」なアーキテクチャの実現
- RLS と組み合わせた**認可ロジックの DB 集約**が、アプリ側 WHERE 句忘れによる事故を構造的に防ぐと再認識された

### PostgreSQL の FUNCTION と PROCEDURE の区別

PostgreSQL では「ストアドプロシージャ」と一括りにされがちな概念が、実は **2 種類の別物**として実装されている。**PostgREST は FUNCTION のみをサポートし、PROCEDURE は呼び出せない**。

| 観点 | `FUNCTION`（関数） | `PROCEDURE`（手続き） |
|---|---|---|
| 導入バージョン | 古くから | **PostgreSQL 11**（2018年） |
| 戻り値 | 必須（`RETURNS ...`） | なし（`OUT` パラメータは可） |
| 呼び出し構文 | `SELECT my_func(...)` | `CALL my_proc(...)` |
| トランザクション制御 | 関数内で `COMMIT`/`ROLLBACK` **不可**（外側のトランザクションに参加） | 関数内で `COMMIT`/`ROLLBACK` **可能**（バッチ処理向き） |
| volatility 指定 | `IMMUTABLE`/`STABLE`/`VOLATILE` | 指定不可 |
| PostgREST `/rpc/` | **対応** | **非対応**（CALL 構文を発行しないため） |
| 主な用途 | クエリの再利用、認可ロジック、ビジネスロジック | データ移行、バッチ処理、長時間トランザクション |

```sql
-- FUNCTION: 戻り値があり、外側のトランザクションに参加する
CREATE FUNCTION transfer_funds(p_from BIGINT, p_to BIGINT, p_amount NUMERIC)
RETURNS BIGINT  -- ← 戻り値必須
LANGUAGE plpgsql
AS $$ ... $$;

-- PROCEDURE: 戻り値なし、内部で COMMIT 可能（PostgreSQL 11+）
CREATE PROCEDURE migrate_legacy_data()
LANGUAGE plpgsql
AS $$
BEGIN
    -- 大量データを 10000 件ごとにコミットしてロックを開放
    LOOP
        UPDATE legacy_table SET ... WHERE migrated = false LIMIT 10000;
        EXIT WHEN NOT FOUND;
        COMMIT;  -- ← FUNCTION では書けない
    END LOOP;
END;
$$;
```

PostgREST の `/rpc/` で公開できるのは**戻り値のある FUNCTION のみ**。バッチ処理や長時間トランザクションを HTTP API として叩きたい場合は、PROCEDURE をラップした FUNCTION（`PERFORM` で `CALL` する）を作るか、別のジョブキュー機構（pg_cron / [[非同期処理とメッセージキュー]]）に逃がす設計判断が必要になる。

### 交差点としての PostgREST `/rpc/`

ここまでの整理を踏まえると、PostgREST の `/rpc/` は次の **2 つの概念の交差点**に位置する:

```mermaid
flowchart LR
    RPC["RPC<br/>(リモート関数呼び出し)"] -.- PR["PostgREST<br/>/rpc/"]
    SP["ストアドプロシージャ<br/>(DB に保存された関数)"] -.- PR

    style PR fill:#c8e6c9,stroke:#2e7d32
```

| 観点 | gRPC | 純粋なストアドプロシージャ | PostgREST `/rpc/` |
|---|---|---|---|
| 関数の保存場所 | アプリケーションサーバー | DB | DB |
| 呼び出し方 | RPC 経由（gRPC クライアント） | DB クライアント（psql / ORM） | RPC 経由（HTTP クライアント） |
| トランスポート | HTTP/2 + Protobuf | DB プロトコル | HTTP/1.1+ + JSON |
| 認証 | mTLS / JWT / metadata | DB ロール（ユーザー名/パスワード） | JWT（PostgREST が検証 + ロール切り替え） |
| 公開対象 | アプリケーションが書いた gRPC service | DB ロール経由で接続したクライアント | インターネット |

つまり PostgREST は「**ストアドプロシージャを HTTP/JSON という現代的な RPC プロトコルで公開する装置**」であり、80 年代の DB 中心アーキテクチャを現代の Web スタックに接続するブリッジになっている。これが教科書的な「アプリケーションサーバーが必須」というアーキテクチャ前提を覆し、Supabase のような「フロントエンド + DB」の 2 層構成を成立させている。

## `/rpc/` エンドポイント — DB の関数を HTTP で呼ぶ

ここが「PostgreRPC」と呼ばれる由来となる中核機能である。**SQL 関数（`CREATE FUNCTION ...`）を `/rpc/<関数名>` で HTTP 呼び出しできる**。

```sql
-- 注文を確定する関数
CREATE OR REPLACE FUNCTION place_order(
    p_product_id BIGINT,
    p_quantity INT
) RETURNS json
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id BIGINT;
    v_user_id UUID;
BEGIN
    -- JWT クレームの sub をユーザーIDとして取り出す
    -- （Supabase 環境では auth.uid() がこの処理をラップしている）
    v_user_id := nullif(
        current_setting('request.jwt.claims', true)::json->>'sub',
        ''
    )::uuid;

    -- 在庫確認 + 引き当て + 注文作成を1トランザクションで
    UPDATE products
    SET stock = stock - p_quantity
    WHERE id = p_product_id AND stock >= p_quantity;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'insufficient stock' USING ERRCODE = 'P0001';
    END IF;

    INSERT INTO orders (user_id, product_id, quantity)
    VALUES (v_user_id, p_product_id, p_quantity)
    RETURNING id INTO v_order_id;

    RETURN json_build_object('order_id', v_order_id);
END;
$$;
```

```bash
# 上記関数を HTTP から呼び出す
curl -X POST 'https://api.example.com/rpc/place_order' \
  -H 'Authorization: Bearer <JWT>' \
  -H 'Content-Type: application/json' \
  -d '{"p_product_id": 42, "p_quantity": 2}'

# → {"order_id": 12345}
```

### REST と RPC の使い分け

PostgREST のドキュメントでも明示されているとおり、**単純な CRUD は REST リソース、複数テーブルにまたがるロジックや副作用のある操作は `/rpc/` 関数**で表現するのが定石。

| ユースケース | 推奨スタイル | 理由 |
|---|---|---|
| ユーザー一覧取得 | `GET /users` (REST) | 単一テーブルの読み取り |
| ユーザー作成 | `POST /users` (REST) | 単一テーブルへのINSERT |
| 注文確定（在庫減算 + 注文作成） | `POST /rpc/place_order` | 複数テーブルの整合性が必要 |
| 集計レポート | `POST /rpc/sales_report` | 複雑なクエリの再利用 |
| パスワード変更 | `POST /rpc/change_password` | bcrypt等の処理を関数内で完結 |
| 検索（複雑な条件） | `POST /rpc/search_products` | OR/AND/NULL 判定の組み合わせ |

### `GET /rpc/<func>` vs `POST /rpc/<func>`

PostgREST は関数の volatility（`STABLE` / `IMMUTABLE` / `VOLATILE`）でメソッドを切り替える:

| 関数の volatility | 許容メソッド | 実行されるトランザクション |
|---|---|---|
| `IMMUTABLE` / `STABLE` | `GET` / `HEAD` / `OPTIONS` / `POST` | **read-only** |
| `VOLATILE`（PostgreSQL のデフォルト） | `OPTIONS` / `POST` のみ | read/write |

`STABLE` 関数を `GET` で公開すると、ブラウザや CDN にキャッシュされうるため、**読み取り専用の検索関数は `STABLE` を明示する**のが性能上のベストプラクティス。さらに `STABLE`/`IMMUTABLE` 関数は POST 経由でも read-only トランザクションで実行されるため、PostgREST をリードレプリカに向けて運用する構成と相性が良い。

## DB-First アーキテクチャ — 責務の再配置

PostgREST を採用すると、伝統的なバックエンドの責務が DB 側に大きく移動する。

```mermaid
flowchart TB
    subgraph "従来型バックエンド"
        T1["ルーティング"]
        T2["認証/認可"]
        T3["バリデーション"]
        T4["ビジネスロジック"]
        T5["SQL生成"]
        T6[("PostgreSQL")]
        T1 --> T2 --> T3 --> T4 --> T5 --> T6
    end

    subgraph "PostgREST型"
        P1["PostgREST<br/>(ルーティング/SQL生成)"]
        P2[("PostgreSQL<br/>RLS / CHECK / 関数")]
        P1 --> P2
    end

    style T4 fill:#fff3e0,stroke:#e65100
    style P2 fill:#c8e6c9,stroke:#2e7d32
```

| 責務 | 従来型 | PostgREST 型 |
|---|---|---|
| ルーティング | フレームワーク（Express等） | PostgREST が自動生成 |
| 認証 | ミドルウェア | PostgREST の JWT 検証 |
| 認可 | アプリケーションコード | RLS ポリシー |
| バリデーション | DTO/スキーマライブラリ | DB の `CHECK` 制約 + 型 |
| ビジネスロジック | サービス層 | SQL 関数（PL/pgSQL） |
| トランザクション制御 | ORM | PostgREST が 1 リクエスト = 1 トランザクションで包む |

このアーキテクチャは「**DB スキーマが API そのもの**」という極端な姿勢を取る。型/制約/ポリシーを DB に一元化することで、認可漏れや WHERE 句の付け忘れが構造的に発生しなくなる利点がある一方で、複雑なドメインロジックを PL/pgSQL で書くことになる難点もある。詳細は [[SupabaseのJWT-RLS連携]] の比較表を参照。

## コード例

### 1. テーブル + 関数のスキーマ定義

```sql
-- ロール定義（PostgREST が SET LOCAL ROLE で切り替える）
CREATE ROLE anon NOLOGIN;
CREATE ROLE authenticated NOLOGIN;
GRANT anon, authenticated TO postgres;

-- スキーマ
CREATE SCHEMA api;
GRANT USAGE ON SCHEMA api TO anon, authenticated;

-- テーブル
CREATE TABLE api.products (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL CHECK (length(name) BETWEEN 1 AND 200),
    price INT NOT NULL CHECK (price >= 0),
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE api.orders (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id UUID NOT NULL,
    product_id BIGINT NOT NULL REFERENCES api.products(id),
    quantity INT NOT NULL CHECK (quantity > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- anon は商品の読み取りだけ、authenticated は注文の書き込みも可
GRANT SELECT ON api.products TO anon, authenticated;
GRANT INSERT, UPDATE ON api.products TO authenticated;
GRANT SELECT, INSERT ON api.orders TO authenticated;

-- RLS
ALTER TABLE api.products ENABLE ROW LEVEL SECURITY;
ALTER TABLE api.orders   ENABLE ROW LEVEL SECURITY;

CREATE POLICY products_read ON api.products
    FOR SELECT TO anon, authenticated USING (true);

-- 自分の注文だけ見える/作れるポリシー
CREATE POLICY orders_owner ON api.orders
    FOR ALL TO authenticated
    USING (user_id = nullif(current_setting('request.jwt.claims', true)::json->>'sub','')::uuid)
    WITH CHECK (user_id = nullif(current_setting('request.jwt.claims', true)::json->>'sub','')::uuid);

-- /rpc/ で公開する関数（読み取り専用なので STABLE）
CREATE OR REPLACE FUNCTION api.search_products(q TEXT)
RETURNS SETOF api.products
LANGUAGE sql STABLE
AS $$
    SELECT * FROM api.products
    WHERE name ILIKE '%' || q || '%'
    ORDER BY created_at DESC
    LIMIT 50;
$$;

GRANT EXECUTE ON FUNCTION api.search_products(TEXT) TO anon, authenticated;
```

### 2. PostgREST の起動設定（`postgrest.conf`）

```ini
db-uri = "postgres://authenticator:secret@localhost:5432/mydb"
db-schemas = "api"
db-anon-role = "anon"

# JWT 検証 — Supabase なら同じシークレットを共有
jwt-secret = "your-256-bit-secret-key-must-be-at-least-32-chars"
jwt-aud = "authenticated"

server-port = 3000
server-host = "0.0.0.0"

# OpenAPI スペックを自動生成
openapi-mode = "follow-privileges"
```

```bash
# Docker での起動例（v14.10 は2025-04時点の最新マイナー）
# 本番環境では `:v14` のような major タグではなく `:v14.10` までピン留めし、
# アップグレードを意図的に行うのが安全
docker run -d --name postgrest \
  -p 3000:3000 \
  -e PGRST_DB_URI="postgres://authenticator:secret@host.docker.internal:5432/mydb" \
  -e PGRST_DB_SCHEMAS="api" \
  -e PGRST_DB_ANON_ROLE="anon" \
  -e PGRST_JWT_SECRET="$(openssl rand -base64 32)" \
  postgrest/postgrest:v14.10
```

### 3. curl での REST/RPC 呼び出し

```bash
# REST: 商品一覧（anon でも可）
curl 'http://localhost:3000/products?select=id,name,price&price=lt.5000&order=price.asc'

# REST: 商品作成（要 JWT）
curl -X POST 'http://localhost:3000/products' \
  -H 'Authorization: Bearer <JWT>' \
  -H 'Content-Type: application/json' \
  -H 'Prefer: return=representation' \
  -d '{"name":"新商品","price":2000,"stock":10}'

# RPC: 検索関数（STABLE なので GET も可）
curl 'http://localhost:3000/rpc/search_products?q=book'

# RPC: 副作用ありの関数（POST 必須）
curl -X POST 'http://localhost:3000/rpc/place_order' \
  -H 'Authorization: Bearer <JWT>' \
  -H 'Content-Type: application/json' \
  -d '{"p_product_id":42,"p_quantity":2}'
```

### 4. TypeScript — 型生成済みクライアント

PostgREST が公開する OpenAPI から型を生成するか、Supabase のクライアントを使うのが一般的。

```typescript
// supabase-js を PostgREST 単体に向けて使う
import { createClient } from "@supabase/supabase-js";

const client = createClient(
  "http://localhost:3000",       // PostgREST のエンドポイント
  "<anon JWT>",
  { auth: { persistSession: false } }
);

// REST: クエリビルダ風 API（裏で /products?... に変換される）
const { data: products, error } = await client
  .from("products")
  .select("id, name, price")
  .lt("price", 5000)
  .order("price", { ascending: true });

// RPC: 関数呼び出し
const { data: orderResult, error: rpcErr } = await client
  .rpc("place_order", { p_product_id: 42, p_quantity: 2 });

// 引数名は SQL 関数の仮引数名と一致させる必要がある
```

### 5. Go — 生 HTTP での RPC 呼び出し

```go
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
)

func placeOrder(jwt string, productID int64, quantity int) (int64, error) {
    body, _ := json.Marshal(map[string]any{
        "p_product_id": productID,
        "p_quantity":   quantity,
    })

    req, _ := http.NewRequest("POST",
        "http://localhost:3000/rpc/place_order",
        bytes.NewReader(body))
    req.Header.Set("Authorization", "Bearer "+jwt)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return 0, err
    }
    defer resp.Body.Close()

    if resp.StatusCode != http.StatusOK {
        msg, _ := io.ReadAll(resp.Body)
        return 0, fmt.Errorf("rpc failed: %d %s", resp.StatusCode, msg)
    }

    var out struct {
        OrderID int64 `json:"order_id"`
    }
    if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
        return 0, err
    }
    return out.OrderID, nil
}
```

## よくある落とし穴

### 1. スキーマ変更後に PostgREST がキャッシュした古いスキーマを使い続ける

PostgREST は起動時とリスナーシグナル受信時にスキーマをリロードする。`CREATE TABLE` や `CREATE FUNCTION` を実行しても、PostgREST 側で**スキーマキャッシュをリロードしないと 404 のまま**になる。

```sql
-- スキーマ変更後にリロードを通知
NOTIFY pgrst, 'reload schema';

-- 設定をリロード（postgrest.conf や環境変数の変更を反映）
NOTIFY pgrst, 'reload config';
```

`db-channel-enabled` と `db-channel = "pgrst"` は**デフォルトで有効**なため明示的な設定は不要。`ddl_command_end` イベントトリガと組み合わせて自動リロード化するのが実運用では一般的。

注意: **PostgreSQL のリードレプリカに PostgREST を接続している場合 LISTEN/NOTIFY が動かないため、リスナー自体が起動失敗する**。リードレプリカ運用ではコンテナ再起動か `SIGUSR1` シグナル送信でリロードするしかない。

### 2. `db-anon-role` の権限を絞り忘れて全テーブルが読める

PostgREST のデフォルトでは「anon ロールに `GRANT SELECT` されたテーブル」が誰でも読める状態で公開される。新規テーブルを追加した際に、デフォルトで anon に SELECT 権限が付与される設定（`ALTER DEFAULT PRIVILEGES`）になっていると意図せぬ漏洩が起きる。

```sql
-- 安全側: anon にはデフォルトで何も与えない
ALTER DEFAULT PRIVILEGES IN SCHEMA api REVOKE ALL ON TABLES FROM anon;

-- 個別に明示的に GRANT する
GRANT SELECT ON api.products TO anon;
```

### 3. `service_role` 相当のロール（BYPASSRLS 持ち）を JWT に乗せて公開してしまう

PostgREST は JWT の `role` クレームをそのまま `SET LOCAL ROLE` に使う。攻撃者が任意の role 文字列を JWT に詰められると（= JWT secret が漏洩した場合）、`BYPASSRLS` 権限を持つロールを指定して RLS をバイパスできてしまう。

| 対策 | 内容 |
|---|---|
| `db-pre-request` フック | リクエスト前に role 名を検証する関数を仕込む |
| ロール作成ポリシー | 公開用 DB には `BYPASSRLS` 持ちロールを作らない |
| JWT secret ローテーション | 定期的に rotate する仕組みを用意 |

### 4. `/rpc/` 関数のトランザクションは PostgREST が自動で BEGIN/COMMIT する

PL/pgSQL 関数内で `BEGIN; ... COMMIT;` を書くとエラーになる。**1 リクエスト = 1 トランザクション**で PostgREST が包むため、関数内では `BEGIN`/`COMMIT` を書かず、必要なら `SAVEPOINT` を使う。例外を `RAISE` すれば自動的にロールバックされる。

### 5. クライアント主導フィルタの脆さ

`/users?id=eq.42` のフィルタ条件はクライアントから完全制御可能。攻撃者は `?id=in.(1,2,3,...,1000000)` のような重い条件を投げて DoS できる。RLS で**読める行**を制限していても、**読み取り処理の負荷**は別問題。

```sql
-- 関数経由で公開し、引数の上限を強制する
CREATE FUNCTION api.bulk_lookup(ids BIGINT[])
RETURNS SETOF api.users
LANGUAGE plpgsql STABLE AS $$
BEGIN
    IF array_length(ids, 1) > 100 THEN
        RAISE EXCEPTION 'too many ids (max 100)';
    END IF;
    RETURN QUERY SELECT * FROM api.users WHERE id = ANY(ids);
END;
$$;
```

### 6. 複雑なドメインロジックを PL/pgSQL で書きすぎる

「DB-First」を徹底すると、結局ロジックの大半が PL/pgSQL 関数として DB 内に住み着く。**PL/pgSQL のバージョン管理・テスト・デバッグの難しさ**は伝統的なアプリケーションコードに比べて大きい。複雑なロジックは外部のバックエンド（Edge Functions / FaaS）に逃がす判断軸を持つこと。

```mermaid
flowchart LR
    Q{"このロジックの複雑度"} -->|"単一トランザクションで完結"| F["PL/pgSQL 関数<br/>= /rpc/"]
    Q -->|"外部 API 呼び出し<br/>長時間処理<br/>分岐が複雑"| E["Edge Function<br/>or バックエンド"]

    style F fill:#c8e6c9,stroke:#2e7d32
    style E fill:#fff3e0,stroke:#e65100
```

### 7. `PROCEDURE` を `/rpc/` で呼ぼうとする

PostgreSQL 11 で導入された `PROCEDURE`（`CALL` 構文で呼ぶ手続き）は **PostgREST の `/rpc/` では呼び出せない**。PostgREST は内部的に `SELECT my_func(...)` を発行するため、戻り値のない PROCEDURE は対象外になる。

```sql
-- ❌ /rpc/migrate_data を呼んでも 404 になる
CREATE PROCEDURE migrate_data() LANGUAGE plpgsql AS $$ ... $$;

-- ✅ FUNCTION でラップすれば /rpc/ から呼べる
CREATE FUNCTION migrate_data_wrap() RETURNS void
LANGUAGE plpgsql AS $$
BEGIN
    CALL migrate_data();  -- 内部で PROCEDURE を呼ぶ
END;
$$;
```

ただし FUNCTION 内では `COMMIT`/`ROLLBACK` ができないため、PROCEDURE 本来の「途中コミット」効果は失われる。バッチ処理や長時間トランザクションは PostgREST `/rpc/` の対象から外し、pg_cron や [[非同期処理とメッセージキュー]] に逃がすのが筋。

## AIによる実装のアンチパターン

| アンチパターン | なぜ問題か | 対策 |
|---|---|---|
| RLS を無効化したまま PostgREST を公開 | 認可がバックエンドに無いため全テーブルが誰でも読み書き可能 | `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` をマイグレーションの必須項目化 |
| `STABLE` を付けない読み取り専用関数 | `GET /rpc/` で呼べない上、CDN/ブラウザキャッシュが効かない | 副作用がない関数は明示的に `STABLE`/`IMMUTABLE` を宣言 |
| `SECURITY DEFINER` 関数を無条件に作る | RLS をバイパスする権限昇格関数になり、引数次第で別ユーザーのデータを操作可能 | 必要な場合のみ使い、関数内で `auth.uid()` を厳密に検証する |
| `db-uri` に super user を指定 | 万一 PostgREST が侵害された場合に DB 全体が陥落 | 専用の `authenticator` ロールを作り、`anon`/`authenticated` への切り替え権限のみを付与 |
| 全テーブルに対して `GRANT ALL ON ALL TABLES IN SCHEMA api TO anon` | 認可の設計が崩壊する | テーブル単位・カラム単位で必要な権限のみ GRANT |
| クライアントから生 SQL 相当のクエリ条件を全許容 | DoS や情報量推測攻撃の温床。ページネーションも壊れる | `max-rows` 設定で上限を強制、複雑な検索は `/rpc/` 関数経由 |

## 実務での使用シーン

1. **MVP / プロトタイプの即時 API 化** — DB スキーマだけ書けば即座に CRUD API が手に入る。フロントエンドだけ書けばプロダクトが動き始める。
2. **管理画面の裏側** — 社内ツールで認可が単純（管理者のみ）な場合、Retool や AppSmith から PostgREST を直接叩く構成は工数を大きく削減する。
3. **Supabase ベースのフルスタック開発** — PostgREST + RLS + JWT を Supabase が統合パッケージとして提供。Next.js / Flutter からのデータレイヤーとして機能する。
4. **既存 PostgreSQL データベースを社内 API として公開** — 既存の DWH や分析 DB を、ロールと RLS で絞った上で社内ツールに公開する。
5. **リードレプリカへの読み取り専用 API** — `db-uri` をリードレプリカに向け、`anon` ロールに SELECT のみ許可する構成で読み取り API を低コストに用意する。

## 関連トピック

- [[RDB]] — 親トピック。PostgREST はテーブル/関数を HTTP に公開する薄層
- [[RLS（Row-Level-Security）]] — PostgREST の認可は RLS が担う
- [[SupabaseのJWT-RLS連携]] — PostgREST + JWT + RLS の統合パッケージとしての Supabase
- [[セッションとJWT]] — PostgREST の認証は JWT に依存
- [[API設計-REST-GraphQL]] — REST/GraphQL/RPC の比較。PostgREST は REST と RPC のハイブリッド
- [[コネクションプール]] — PostgREST 自身は内部で接続プールを持つが、PgBouncer 等と組み合わせる場合のトランザクションプーリング互換性に注意
- [[トランザクション]] — PostgREST は 1 リクエスト = 1 トランザクションで包む。FUNCTION では `COMMIT`/`ROLLBACK` 不可で PROCEDURE では可、という区別の前提知識

## 参考リソース

- [PostgREST 公式ドキュメント](https://docs.postgrest.org/) — REST 構文、`/rpc/`、認証、設定の網羅的リファレンス
- [PostgREST 公式チュートリアル](https://docs.postgrest.org/en/stable/tutorials/tut0.html) — JWT 認証付き API を 30 分で立ち上げる
- [PostgREST: Functions as RPC](https://docs.postgrest.org/en/stable/references/api/functions.html) — `/rpc/` の volatility と HTTP メソッドの対応・read-only トランザクションの扱い
- [PostgreSQL: CREATE FUNCTION / CREATE PROCEDURE](https://www.postgresql.org/docs/current/sql-createfunction.html) — FUNCTION と PROCEDURE の構文上の違いの一次ソース
- [Supabase Architecture](https://supabase.com/docs/guides/getting-started/architecture) — PostgREST を中核に据えた BaaS の構成例
- [JSON-RPC 2.0 Specification](https://www.jsonrpc.org/specification) — シンプルな RPC プロトコルの定義（PostgREST は別系統だが概念整理に有用）
- [gRPC](https://grpc.io/) — 現代の RPC プロトコルのデファクト。PostgREST `/rpc/` との対比で理解が深まる
