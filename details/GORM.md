---
layer: 4
parent: "[[Go]]"
type: detail
created: 2026-06-30
---

# GORM（Go ORM）

> **一言で言うと:** GORM は Go で最も広く使われる **ORM（Object-Relational Mapping、オブジェクト関係マッピング）**。Go の構造体をそのままテーブルとして扱い、`db.Create/First/Find/Save/Delete` というメソッドで SQL を書かずに CRUD を回せる。生 SQL（`database/sql`）と SQL ビルダ（sqlc など）の中間で、「**生産性は高いが、生成される SQL を読めないと N+1 やインデックス未使用で足を撃つ**」のが本質。インポートパスは `gorm.io/gorm`。

> [!info] 用語ミニ辞典：ORM とは
> アプリのオブジェクト（Go の構造体）とリレーショナル DB の行・テーブルの間を自動変換するライブラリの総称。「`user := User{Name: "Alice"}; db.Create(&user)`」と書けば、裏で `INSERT INTO users (name) VALUES ('Alice')` が発行される。**SQL を直接書く手間を減らす**のが目的だが、裏返すと**どんな SQL が飛んでいるかが見えにくくなる**。この「便利さと不透明さのトレードオフ」が ORM 理解の中心。

GORM は [[データアクセス層]] の具体的な実装手段の一つ。なぜ生 SQL ではなく ORM を選ぶのか、その判断軸は [[データアクセス層]] を先に読むと位置づけが掴める。

## バージョンの位置づけ — 「GORM v2」とは

現場で「GORM v2」と言うとき、それは**インポートパス `gorm.io/gorm`**（2020年の全面書き直し版）を指す。旧 v1（`github.com/jinzhu/gorm`）とは API が大きく異なり、現在の新規プロジェクトは必ず `gorm.io/gorm` を使う。

```go
import (
	"gorm.io/gorm"
	"gorm.io/driver/postgres" // DB ごとにドライバを別パッケージで足す
)
```

DB 接続を司る**ドライバはコア（`gorm.io/gorm`）と分離**されている。PostgreSQL なら `gorm.io/driver/postgres`、MySQL なら `gorm.io/driver/mysql`、SQLite なら `gorm.io/driver/sqlite`。参画時は `go.mod` でどのドライバが入っているかを確認する。RDB の選択差は [[PostgreSQLとMySQLの比較]] を参照。

## 接続とモデル定義

```go
func main() {
	dsn := "host=localhost user=app password=secret dbname=app port=5432 sslmode=disable"
	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatal(err)
	}

	// database/sql の *sql.DB を取り出してコネクションプールを設定（重要）
	sqlDB, _ := db.DB()
	sqlDB.SetMaxOpenConns(25)                 // 同時接続の上限
	sqlDB.SetMaxIdleConns(25)                 // アイドル保持数
	sqlDB.SetConnMaxLifetime(5 * time.Minute) // 接続の寿命
}
```

接続プールの設定（`SetMaxOpenConns` 等）は GORM 独自ではなく、その下の標準 `database/sql` の機能。**この値を未設定のまま本番に出すと、DB の最大接続数を食い潰すか、逆に接続不足でリクエストが詰まる**。チューニングの考え方は [[コネクションプール]] を参照。

### モデル = テーブルにマッピングされる構造体

```go
type User struct {
	gorm.Model          // ID, CreatedAt, UpdatedAt, DeletedAt を埋め込む（後述）
	Name   string       `gorm:"size:100;not null"`
	Email  string       `gorm:"uniqueIndex"`        // ユニークインデックスを張る
	Age    int          `gorm:"default:0"`
	Orders []Order                                  // 1対多のリレーション（has many）
}

type Order struct {
	gorm.Model
	UserID uint        // 外部キー（User.ID を参照する慣習名）
	Amount int
}
```

`gorm:"..."` という**構造体タグ**がカラムの制約（型・NULL 可否・インデックス・デフォルト値）を表す。テーブル名は構造体名の**複数形スネークケース**が既定（`User` → `users`）。命名規約が暗黙に効くため、「なぜテーブル名が複数形に？」で戸惑わないこと。

## 基本 CRUD

```go
// Create — INSERT。引数はポインタ（生成された ID が書き戻されるため）
user := User{Name: "Alice", Email: "alice@example.com"}
db.Create(&user)
// → user.ID に採番された値が入る

// Read — 主キーで1件 / 条件で複数件
var u User
db.First(&u, 1)                         // SELECT ... WHERE id = 1 ORDER BY id LIMIT 1
db.Where("age > ?", 18).Find(&users)    // SELECT ... WHERE age > 18

// Update — 単一カラム / 構造体まとめて
db.Model(&u).Update("Age", 30)          // UPDATE users SET age=30 WHERE id=...
db.Model(&u).Updates(User{Name: "Bob", Age: 31}) // ゼロ値フィールドは無視される（後述の罠）

// Delete — gorm.Model 使用時は論理削除（後述）
db.Delete(&u, 1)
```

### `?` プレースホルダは SQL インジェクション対策の要

`db.Where("age > ?", userInput)` の `?` は、値を**プレースホルダ経由で安全に渡す**（プリペアドステートメント相当）。これを使う限り、ユーザー入力がそのまま SQL 文字列に連結されることはない。一方で**文字列連結で条件を組み立てると即座に脆弱**:

```go
// ❌ SQL インジェクションの穴
db.Where(fmt.Sprintf("name = '%s'", userInput)).Find(&users)

// ✅ プレースホルダを使う
db.Where("name = ?", userInput).Find(&users)
```

なぜプレースホルダが安全なのかは [[SQLインジェクションとXSS]] を参照。ORM を使っていても**生 SQL 片を渡せる API（`Where`/`Raw`）では自分でこの規律を守る必要がある**——「ORM だから自動で安全」は誤解。

## アソシエーションと N+1 問題 — GORM 最大の勘所

リレーション（関連）の取得方法が、ORM のパフォーマンスを左右する最大ポイント。

```go
// ❌ N+1 問題: ユーザー100人分のループで Orders を都度取得 → SQL が 1 + 100 回
var users []User
db.Find(&users)                    // 1回: SELECT * FROM users
for _, u := range users {
	db.Where("user_id = ?", u.ID).Find(&u.Orders) // 100回: ユーザーごとに1クエリ
}

// ✅ Preload: 関連をまとめて2クエリで取得
db.Preload("Orders").Find(&users)
// 1回目: SELECT * FROM users
// 2回目: SELECT * FROM orders WHERE user_id IN (1,2,3,...)
```

> [!info] 用語ミニ辞典：N+1 問題
> 親レコード N 件を取得した後、関連する子レコードを「1件ずつ」追加クエリで取りに行ってしまい、合計 `1 + N` 回の SQL が飛ぶアンチパターン。件数が増えるほど DB 往復が線形に膨らみ、API が遅くなる典型原因。**ORM は便利さゆえにこれを"気づかず"書いてしまう**のが怖いところ。

GORM はこれを **`Preload`（Eager ロード＝先読み）** で解決する。`Preload` は親の主キー集合を `IN` 句でまとめて1クエリにする。逆に関連を「アクセスされた時に初めて引く」のが Lazy ロード。この2方式の選び分け（先読みは無駄取得、遅延読みは N+1 を生む）は [[EagerロードとLazyロード]] を参照——**「どちらが正しい」ではなく、画面が何を必要とするかで決める**のが要点。

JOIN で1クエリにまとめたい場合は `Joins` を使う。ここでは `User` が会社に属する（`belongs to`）関連 `Company` を持つ前提で示す:

```go
type User struct {
	gorm.Model
	Name      string
	CompanyID uint     // belongs to の外部キー
	Company   Company  // 1対1（ユーザーは1社に属する）
	Orders    []Order  // 1対多
}

// 1クエリで JOIN（1対1 や、フィルタ条件を関連テーブルに掛けたいとき向き）
db.Joins("Company").Find(&users) // SELECT ... FROM users LEFT JOIN companies ...
```

`Preload`（複数クエリ + `IN`）と `Joins`（単一 JOIN）は使い分ける。**1対多（`Orders`）で `Joins` を使うと親行が子の数だけ重複**するため、1対多は基本 `Preload`、1対1（`Company`）や絞り込み目的は `Joins` が定石。

## トランザクション

複数の書き込みを「全部成功か、全部取り消し」でまとめる:

```go
// 推奨: クロージャ版。return err で自動 Rollback、nil で自動 Commit
err := db.Transaction(func(tx *gorm.DB) error {
	if err := tx.Create(&order).Error; err != nil {
		return err // ここで return すると Rollback
	}
	if err := tx.Model(&user).Update("balance", user.Balance-order.Amount).Error; err != nil {
		return err
	}
	return nil // ここまで来たら Commit
})
```

クロージャ内では `db` ではなく**引数の `tx` を使う**のが鉄則（`db` を使うとトランザクション外で実行されてしまう）。手動で `db.Begin()/Commit()/Rollback()` も書けるが、`Rollback` の呼び忘れで接続が漏れやすいためクロージャ版が安全。トランザクションの分離レベルや一貫性の理屈は [[トランザクション]] を参照。

## マイグレーション — AutoMigrate の光と影

```go
// 構造体定義からテーブル/カラム/インデックスを生成・追加
db.AutoMigrate(&User{}, &Order{})
```

`AutoMigrate` はモデル定義と DB スキーマの差分を見て、**テーブル作成・カラム追加・インデックス追加**を行う。開発初期は手軽だが、**カラム削除・型変更・リネームは行わない**（データ損失を避けるため意図的に保守的）。

そのため本番運用では「`AutoMigrate` に全面依存せず、破壊的変更は明示的なマイグレーションツールで管理する」のが定石。スキーマ変更を**バージョン管理された SQL/コードとして残す**理由——ロールバック可能性、レビュー可能性、複数環境での再現性——は [[マイグレーション]] を参照。`AutoMigrate` は「スキーマの真実の源」にはしないのが安全。

## gorm.Model と論理削除（ソフトデリート）

`gorm.Model` を埋め込むと `ID / CreatedAt / UpdatedAt / DeletedAt` が付く。このうち `DeletedAt` が**論理削除**のスイッチになる:

```go
db.Delete(&user) // 実際には DELETE せず UPDATE users SET deleted_at = NOW()
db.Find(&users)  // 自動で WHERE deleted_at IS NULL が付き、削除済みは見えない
```

> [!info] 用語ミニ辞典：論理削除（ソフトデリート）
> 行を物理的に消さず、「削除済みフラグ（ここでは `deleted_at` の日時）」を立てるだけにする方式。誤削除からの復旧や監査証跡に有利な反面、テーブルに死んだ行が溜まり続け、クエリに常に `deleted_at IS NULL` 条件が必要になる。

ここが**初参画者を最も驚かせる挙動**: `Delete` したのに行が残る。物理削除したいときは `db.Unscoped().Delete(&user)`。論理削除はメリットとコスト（インデックス効率の悪化、ユニーク制約との相性問題）が両面あるため、テーブルごとに採用是非を判断する。

## 生 SQL へのエスケープハッチ

ORM で表現しづらい複雑なクエリは、生 SQL に逃がせる。**「ORM を使う＝生 SQL を一切書かない」ではない**:

```go
var results []Result
db.Raw("SELECT name, COUNT(*) AS cnt FROM users GROUP BY name HAVING COUNT(*) > ?", 1).
	Scan(&results)
```

GORM が生成する SQL が非効率なとき、あるいはウィンドウ関数・CTE など ORM の表現力を超えるときは、迷わず `Raw`/`Exec` を使う。重要なのは**生成 SQL をログで見て判断できること**。開発時は `gorm.Config{Logger: ...}` で SQL ログを有効にし、「自分のコードがどんな SQL を吐くか」を常に確認する習慣をつける。

## よくある落とし穴

### 1. `Updates(struct)` がゼロ値を無視する

```go
// Age を 0 に更新したいのに、0 は Go のゼロ値なので無視される
db.Model(&u).Updates(User{Age: 0}) // UPDATE 文に age が含まれない！

// 対策: map を使うか、Select で対象カラムを明示
db.Model(&u).Updates(map[string]interface{}{"age": 0})
db.Model(&u).Select("Age").Updates(User{Age: 0})
```

構造体経由の更新は「ゼロ値＝未指定」とみなす仕様。`false`・`0`・`""` への更新が「効かない」バグの定番原因。

### 2. N+1 を `Preload` 忘れで踏む

最頻出。ループ内で関連にアクセスするコードを書いたら、**まず `Preload` / `Joins` を検討**する。SQL ログで「同じ形のクエリが件数分並んでいないか」を確認する癖をつける。切り分けの一般手順は [[バックエンドパフォーマンス切り分けガイド]] を参照。

### 3. `Find` のエラーと「レコードなし」の混同

```go
err := db.First(&u, id).Error
if errors.Is(err, gorm.ErrRecordNotFound) {
	// 404 を返す: 「見つからない」は正常系の分岐
}
```

`First`/`Take`/`Last` は**レコードが無いと `gorm.ErrRecordNotFound` を返す**が、`Find`（複数件取得）は0件でもエラーにならない（空スライスが返る）。「見つからない」をエラーで判定するか件数で判定するかを取り違えない。[[Go]] の `errors.Is` でラップされたエラーを検査する。

### 4. コネクションプール未設定で接続枯渇

前述の `SetMaxOpenConns` 等を設定しないと、デフォルトでは無制限に接続を開こうとする。高負荷時に DB の `max_connections` を超えてエラーが連鎖する。[[コネクションプール]] の設計指針に沿って明示設定する。

### 5. インデックスはタグで張れても「効くか」は別問題

`gorm:"index"` でインデックスを定義できるが、**実際にそのインデックスがクエリで使われるか（あるいは不要か）は GORM の管轄外**。複合インデックスの列順や選択性は自分で設計する。判断軸は [[インデックス設計の判断基準]]・[[インデックス]] を参照。

## AI実装のアンチパターン

| アンチパターン | なぜ問題か | 対策 |
|---|---|---|
| ループ内で関連を都度取得 | N+1 で SQL が件数分発行され遅延 | `Preload` / `Joins` でまとめる。SQL ログで本数を確認 |
| `fmt.Sprintf` で WHERE 句を組む | SQL インジェクション | 必ず `?` プレースホルダを使う |
| `Updates(struct)` でゼロ値更新を期待 | ゼロ値が無視され更新されない | `map` か `Select` で対象カラムを明示 |
| `AutoMigrate` を本番スキーマ管理の唯一手段にする | カラム削除/型変更を扱えず差分が崩れる | 破壊的変更は[[マイグレーション]]ツールで管理 |
| コネクションプール無設定 | 接続枯渇・性能不安定 | `SetMaxOpenConns` 等を明示設定 |
| 生成 SQL を確認せず ORM を信頼 | 非効率クエリ・想定外の JOIN に気づけない | SQL ロガーを有効化し、生成 SQL をレビュー |
| `First` のエラーを全部 500 にする | 「未検出」は正常系の 404 | `gorm.ErrRecordNotFound` を分岐 |

## 関連トピック

- [[Go]] — 親トピック。構造体タグ・`errors.Is`・ポインタ渡しの前提
- [[Echo]] — GORM を呼び出す側のウェブフレームワーク（ハンドラ → GORM のデータ層）
- [[データアクセス層]] — ORM か生 SQL か、層の設計判断
- [[EagerロードとLazyロード]] — Preload / Lazy の選び分け（N+1 の根）
- [[トランザクション]] — `db.Transaction` の背景理論
- [[マイグレーション]] — AutoMigrate の限界と本番運用
- [[コネクションプール]] — `SetMaxOpenConns` 等の設計
- [[インデックス]] / [[インデックス設計の判断基準]] — タグで張った先の設計判断
- [[SQLインジェクションとXSS]] — プレースホルダがなぜ安全か
- [[PostgreSQLとMySQLの比較]] — ドライバ選択の前提となる RDB の差
- [[バックエンドパフォーマンス切り分けガイド]] — N+1 を含む遅延の切り分け

## 参考リソース

- [GORM 公式ドキュメント](https://gorm.io/docs/) — モデル定義・クエリ・アソシエーション
- [GORM GoDoc](https://pkg.go.dev/gorm.io/gorm) — API リファレンス
- [database/sql](https://pkg.go.dev/database/sql) — GORM の土台。プール設定はここの API
- [sqlc](https://sqlc.dev/) — 「生 SQL → 型付き Go コード生成」という GORM の対抗アプローチ
