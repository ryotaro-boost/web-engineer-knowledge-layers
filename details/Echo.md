---
layer: 4
parent: "[[Go]]"
type: detail
created: 2026-06-30
---

# Echo v4（Go ウェブフレームワーク）

> **一言で言うと:** Echo は Go の標準 `net/http` の薄いラッパーとして作られた**高速・ミニマルなウェブフレームワーク**。Radix Tree ベースの高速ルーター、`echo.Context` を中心にしたリクエスト処理、そして「玉ねぎモデル」のミドルウェアチェーンが3本柱。Gin と並ぶ二大選択肢で、「**整理された API と公式ミドルウェアの充実**」が選定理由になりやすい。インポートパスは `github.com/labstack/echo/v4`。

Go の標準ライブラリだけでも Web サーバーは書ける（[[Go]] の「標準ライブラリの強さ」を参照）。では**なぜフレームワークを足すのか**——パスパラメータ・JSON バインド・バリデーション・ミドルウェア・グループ化といった「どの API でも必ず書く定型」を、毎回手で書かずに済ませるためだ。Echo はその定型を最小の抽象で覆う立場をとる。

## なぜ v4 なのか — バージョンの位置づけ

Echo はメジャーバージョンを**インポートパスに埋め込む**（Go modules の semantic import versioning）。そのため `v4` は単なる数字ではなく、`import` 文に現れる:

```go
import "github.com/labstack/echo/v4"
```

| 系統 | 位置づけ |
|---|---|
| v3 以前 | パスに `/v3` を持たない旧構造。現在は使わない |
| **v4** | **2018年〜現在の安定版。本番採用の主流。`context.Context` 統合・標準 `http.Handler` 互換が成熟** |
| v5 | 開発中の次期メジャー。API が一部変わるため、現場では当面 v4 が標準 |

新規参画時にまず確認すべきは `go.mod` の `github.com/labstack/echo/v4 vX.Y.Z` の行。**v4 系内のマイナー更新は後方互換**なので、パッチ追従は基本的に安全だ。

## 3分で全体像

- **何を解決するか** — `net/http` の素朴な `ServeMux` では面倒な「パスパラメータ抽出・JSON 変換・横断処理（ログ/認証/CORS）・ルートのグループ化」をまとめて引き受ける
- **これだけ覚える3つ** — ①ルーティングは `e.GET("/users/:id", handler)`、②ハンドラは `func(c echo.Context) error` の一形式、③共通処理は**ミドルウェア**で差し込む
- **詰まったら開く** — ルーターの仕組みは [[Radix Treeとルーター]]、ミドルウェアの考え方は [[ルーティングとミドルウェア]]、横断処理の入れ子は [[玉ねぎモデル]]

## 中核1: ルーティング

```go
package main

import (
	"net/http"

	"github.com/labstack/echo/v4"
)

func main() {
	e := echo.New()

	// メソッド + パスでハンドラを登録
	e.GET("/users/:id", getUser)        // :id はパスパラメータ
	e.POST("/users", createUser)
	e.GET("/files/*", serveFile)        // * はワイルドカード（残り全部）

	e.Logger.Fatal(e.Start(":8080"))
}

func getUser(c echo.Context) error {
	id := c.Param("id")                 // :id を取り出す
	q := c.QueryParam("verbose")        // ?verbose=1 のクエリ
	return c.JSON(http.StatusOK, map[string]string{"id": id, "verbose": q})
}
```

Echo の内部ルーターは **Radix Tree（基数木）** で実装されており、登録ルート数が増えても探索コストがほぼ一定に保たれる。`/users/:id` のような動的セグメントと `/users/me` のような静的セグメントが混在しても、最長一致のルールで正しく振り分ける。この仕組み自体は Echo 固有ではなく多くの高速ルーターが採用する設計で、背景は [[Radix Treeとルーター]] を参照。

### ハンドラのシグネチャは1つだけ

Echo のハンドラは**必ず `func(c echo.Context) error`**。`net/http` の `func(w http.ResponseWriter, r *http.Request)` と違い、**`error` を返す**のが最大の差。返したエラーは後述の集中エラーハンドラが受け取るため、各ハンドラで `http.Error(...)` を散らさずに済む。

## 中核2: echo.Context

`echo.Context`（`c`）は**1リクエストの全情報と便利メソッドを束ねた窓口**。標準の `http.Request` / `http.ResponseWriter` を内部に持ちつつ、よく使う操作をメソッド化している。

```go
func createUser(c echo.Context) error {
	// 1. リクエストボディ(JSON)を構造体に流し込む = バインド
	var u User
	if err := c.Bind(&u); err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, "invalid body")
	}

	// 2. バリデーション（後述のValidatorを登録しておく）
	if err := c.Validate(&u); err != nil {
		return err
	}

	// 3. レスポンスを返す（JSON/String/HTML/Blob など多彩）
	return c.JSON(http.StatusCreated, u)
}

type User struct {
	Name  string `json:"name"  validate:"required"`
	Email string `json:"email" validate:"required,email"`
}
```

> [!info] 用語ミニ辞典：バインド（Bind）
> HTTP リクエストの生のボディ（JSON や form データ）を、Go の構造体フィールドへ自動で対応づけて詰める処理のこと。`json:"name"` のような**構造体タグ**がマッピングの地図になる。手で `json.Unmarshal` を呼ぶのと等価だが、Content-Type を見て JSON / XML / form を自動判別してくれる点が便利。

### バリデーションは「自前 Validator の登録」が必要

ここは初参画でハマりやすい。**Echo はバリデーション機能を内蔵していない**。`c.Validate()` を呼ぶには、`echo.Validator` インターフェースを満たす実装を自分で登録する。定番は `go-playground/validator`:

```go
import "github.com/go-playground/validator/v10"

type CustomValidator struct{ validator *validator.Validate }

func (cv *CustomValidator) Validate(i interface{}) error {
	if err := cv.validator.Struct(i); err != nil {
		return echo.NewHTTPError(http.StatusBadRequest, err.Error())
	}
	return nil
}

func main() {
	e := echo.New()
	e.Validator = &CustomValidator{validator: validator.New()} // ← これを忘れると c.Validate() は ErrValidatorNotRegistered を返す（検証が黙って無効化される）
	// ...
}
```

「なぜ内蔵しないのか」——Echo はミニマル志向で、検証ライブラリの選択を利用者に委ねる立場をとる。バリデーションの一般論（どこで何を検証すべきか）は [[バリデーション]]、入力検証とサニタイズの線引きは [[バリデーションとサニタイズとエスケープ]] を参照。

## 中核3: ミドルウェア — 玉ねぎモデル

ミドルウェアは「ハンドラの前後に挟む共通処理」。Echo では**リクエストが外側から内側へ入り、レスポンスが内側から外側へ抜ける**——この入れ子構造を [[玉ねぎモデル]] と呼ぶ。

```mermaid
graph LR
    Req["リクエスト"] --> L["Logger"]
    L --> R["Recover"]
    R --> A["Auth"]
    A --> H["ハンドラ<br/>(c echo.Context) error"]
    H --> A2["Auth(後処理)"]
    A2 --> R2["Recover(後処理)"]
    R2 --> L2["Logger(応答時間記録)"]
    L2 --> Res["レスポンス"]
```

```go
import "github.com/labstack/echo/v4/middleware"

func main() {
	e := echo.New()

	// 全ルートに適用される共通ミドルウェア
	e.Use(middleware.Logger())   // アクセスログ
	e.Use(middleware.Recover())  // panic を 500 に変換しプロセスを守る
	e.Use(middleware.CORS())     // CORS ヘッダ付与（詳細は後述）

	e.GET("/health", func(c echo.Context) error {
		return c.String(http.StatusOK, "ok")
	})
	e.Logger.Fatal(e.Start(":8080"))
}
```

### 自作ミドルウェアの形

ミドルウェアは「**ハンドラを受け取り、包んだハンドラを返す関数**」。この高階関数の形がチェーンを作る:

```go
func RequestID(next echo.HandlerFunc) echo.HandlerFunc {
	return func(c echo.Context) error {
		id := c.Request().Header.Get("X-Request-ID")
		if id == "" {
			id = generateID()
		}
		c.Set("request_id", id)        // c.Set/Get でハンドラへ値を渡す
		c.Response().Header().Set("X-Request-ID", id)
		return next(c)                 // ← next を呼ばないと、ここで処理が止まる
	}
}

// e.Use(RequestID) で登録
```

ポイントは `return next(c)` の一行。**これを呼び忘れると後続のミドルウェアとハンドラに到達しない**（リクエストがそこで握りつぶされる）。認証ミドルウェアで「未認証なら `next` を呼ばず 401 を返す」のは、この性質を意図的に使った定石だ。ミドルウェア全般の設計論は [[ルーティングとミドルウェア]] を参照。

### `middleware.Recover()` の役割 — panic でプロセスを落とさない

Go では**ハンドラ内で `panic` が起きると、そのままだとサーバープロセス全体が落ちる**。`Recover()` は各リクエストを `recover()` で包み、panic を HTTP 500 に変換してプロセスを生かし続ける。本番では実質必須のミドルウェアだ。なぜ panic をエラー戻り値ではなくこちらで拾うのかは [[エラーハンドリング]]・[[Go]] のエラーハンドリング哲学とあわせて理解したい。

## グループ化 — 共通プレフィックスと共通ミドルウェア

API のバージョニングや認証境界を表現するのに使う:

```go
func main() {
	e := echo.New()

	// /api/v1 配下をまとめ、このグループだけ JWT 認証を要求
	api := e.Group("/api/v1")
	api.Use(echojwt.JWT([]byte("secret")))  // github.com/labstack/echo-jwt/v4
	api.GET("/me", getMe)
	api.GET("/orders", listOrders)

	// 認証不要の公開ルートはグループ外
	e.POST("/login", login)
}
```

グループは「URL プレフィックス + そのプレフィックス配下に共通で効くミドルウェア」をひとまとめにする。認証境界を**グループの内/外で表現**できるのが実務的な価値。JWT の中身とセッション方式の比較は [[セッションとJWT]]、認可の設計は [[認証と認可]] を参照。

## エラーハンドリング — 集中管理

Echo は**全ハンドラ・ミドルウェアが返した `error` を1か所で受け取る**仕組みを持つ。これが「ハンドラが `error` を返す」設計の真価だ。

```go
// ハンドラ側はビジネスロジックのエラーを素直に返すだけ
func getUser(c echo.Context) error {
	u, err := db.Find(c.Param("id"))
	if err == ErrNotFound {
		return echo.NewHTTPError(http.StatusNotFound, "user not found")
	}
	if err != nil {
		return err // 想定外エラーは集中ハンドラが 500 に変換
	}
	return c.JSON(http.StatusOK, u)
}

// 集中エラーハンドラをカスタマイズ（レスポンス形式を統一）
func main() {
	e := echo.New()
	e.HTTPErrorHandler = func(err error, c echo.Context) {
		code := http.StatusInternalServerError
		msg := "internal server error"
		if he, ok := err.(*echo.HTTPError); ok {
			code = he.Code
			msg = he.Message.(string)
		}
		// クライアントに返す JSON 形式をプロジェクトで統一
		c.JSON(code, map[string]string{"error": msg})
	}
}
```

各ハンドラで `c.JSON(500, ...)` を散らす代わりに、**エラー時のレスポンス整形を1か所に集約**できる。エラーレスポンスの設計指針は [[エラーハンドリング]]、フォールバック戦略全般は [[エラーハンドリングとフォールバックの設計戦略]] を参照。

## CORS — 公式ミドルウェアで設定

ブラウザからの異なるオリジンへのリクエストを許可する設定も公式ミドルウェアで完結する:

```go
e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
	AllowOrigins: []string{"https://app.example.com"},
	AllowMethods: []string{http.MethodGet, http.MethodPost},
	AllowHeaders: []string{echo.HeaderAuthorization, echo.HeaderContentType},
}))
```

`middleware.CORS()`（引数なし）は**全オリジン許可（`*`）のゆるい設定**になる点に注意。本番では `AllowOrigins` を明示的に絞る。CORS が何を防ぎ何を防がないかは [[CORS]] を参照——「CORS はサーバーを守るものではなくブラウザの同一オリジンポリシーの例外許可」という本質を押さえておくと、設定ミスを誤った安心に変えずに済む。

## よくある落とし穴

### 1. Validator 未登録で `c.Validate()` が黙ってスルーされる

`e.Validator` を設定せずに `c.Validate()` を呼んでも**panic はしない**——`ErrValidatorNotRegistered` という error を返すだけだ。怖いのはこの先で、**`Validate` の戻り値を捨てていると検証が一切効かないまま正常系として通過する**。「バリデーションを書いたのに不正値が通る」障害の典型原因がこれ。`Bind` と `Validate` は別物（`Bind` は「詰める」、`Validate` は「中身を検証する」）であり、両方を意識的に呼び、かつ `Validate` の戻り値を必ずハンドルする。

### 2. ミドルウェアで `next(c)` を呼び忘れる

リクエストが無言で止まり、タイムアウトするまで気づきにくい。自作ミドルウェアでは「early return（401 等）以外の経路では必ず `return next(c)`」を徹底する。

### 3. `c.Bind()` のバインド範囲を取り違える

`DefaultBinder.Bind` は **①パスパラメータ → ②クエリ（GET / DELETE / HEAD のみ）→ ③ボディ** の順でバインドする。注意点が2つ。

- **パスパラメータはバインドされるが、構造体に `param:"id"` タグが要る**。タグがなければ `:id` は空のまま（「Bind したのに ID が空」の正体はタグ忘れ）。`c.Param("id")` で個別に取る手もある。
- **POST / PUT ではクエリ文字列はバインドされない**。ボディとクエリで同名フィールドが衝突したときの優先順位の混乱を避けるための仕様（Echo 'issue #1670'、v4.1.11 以降）。`?foo=bar` を POST で受けたいなら `c.QueryParam("foo")` で明示的に取る。

```go
type CreateReq struct {
	ID   string `param:"id"`   // パスの :id をここに入れるにはこのタグが必須
	Name string `json:"name"`  // ボディの name
}
```

### 4. `c.Get()` の戻り値は `interface{}` — 型アサーション必須

```go
// ミドルウェアで c.Set("user", user) した値を取り出す
user, ok := c.Get("user").(*User) // 型アサーションを忘れると interface{} のまま
if !ok {
	return echo.NewHTTPError(http.StatusInternalServerError, "user not in context")
}
```

`c.Set/Get` は `interface{}` を介すため、Go の型安全の網の外。アサーションの `ok` を必ず確認する（[[Go]] の「nil interface vs nil concrete」と同じく interface 値の扱いが絡む）。

### 5. ハンドラ内の重い処理を goroutine に投げて `c` を使い続ける

```go
func handler(c echo.Context) error {
	go func() {
		// ❌ レスポンス返却後に c を触ると競合・パニック
		c.Logger().Info(c.Param("id"))
	}()
	return c.NoContent(http.StatusAccepted)
}
```

`echo.Context` は**リクエストのライフサイクルに紐づく**。ハンドラが return した後（レスポンス送信後）に別 goroutine から `c` を触ると未定義動作になる。非同期処理に渡すなら、必要な値（ID 等）を**先にコピーして**渡す。goroutine の終了戦略は [[Go]] の Goroutine リーク項を参照。

## AI実装のアンチパターン

| アンチパターン | なぜ問題か | 対策 |
|---|---|---|
| ハンドラ内で `c.JSON(500, ...)` を散在させる | エラーレスポンス形式が不統一・重複 | `HTTPErrorHandler` に集約し、ハンドラは `error` を返すだけにする |
| `middleware.CORS()` を引数なしで本番投入 | 全オリジン許可になりセキュリティ穴 | `CORSWithConfig` で `AllowOrigins` を明示 |
| Validator 未登録のまま `c.Validate()` を生成 | panic はせず `ErrValidatorNotRegistered` を返すだけで、戻り値を捨てると検証が黙って無効化される | `e.Validator` の登録コードまで一式で提示させ、`Validate` の戻り値を必ずハンドルする |
| 認証チェックを各ハンドラ冒頭にコピペ | DRY 違反・抜け漏れ | `Group` + 認証ミドルウェアで境界を一元化 |
| `c.Bind` だけで検証済みとみなす | バインドは型変換のみ、値の妥当性は未検証 | `Bind` の後に必ず `Validate` |
| panic を `Recover()` 任せにしてログを握りつぶす | 原因が追えない | Recover のログ出力を確認、想定エラーは `error` で返す |

## 関連トピック

- [[Go]] — 親トピック。`net/http`・goroutine・エラーハンドリング哲学の前提
- [[GORM]] — Echo のハンドラから呼ぶ DB アクセス層の定番 ORM（セットで使うことが多い）
- [[Goの環境構築とツールチェーン]] — Echo アプリをビルド/テスト/依存解決する `go` コマンド一式
- [[ルーティングとミドルウェア]] — ルーティング/ミドルウェアの一般設計論
- [[Radix Treeとルーター]] — Echo の高速ルーターの内部データ構造
- [[玉ねぎモデル]] — ミドルウェアチェーンの入れ子構造
- [[バリデーション]] / [[バリデーションとサニタイズとエスケープ]] — `c.Validate` の背景
- [[エラーハンドリング]] / [[エラーハンドリングとフォールバックの設計戦略]] — 集中エラーハンドラの設計
- [[認証と認可]] / [[セッションとJWT]] — グループ + JWT ミドルウェアの設計
- [[CORS]] — CORS ミドルウェアの正しい理解

## 参考リソース

- [Echo 公式ガイド](https://echo.labstack.com/docs) — ルーティング・ミドルウェア・レシピ集
- [Echo GoDoc](https://pkg.go.dev/github.com/labstack/echo/v4) — API リファレンス
- [go-playground/validator](https://github.com/go-playground/validator) — Validator 登録の定番
- [echo-jwt](https://github.com/labstack/echo-jwt) — 公式 JWT ミドルウェア（インポートは `github.com/labstack/echo-jwt/v4`）
