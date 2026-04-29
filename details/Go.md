---
layer: 4
parent: "[[プログラミング言語の系譜と選択]]"
type: detail
created: 2026-04-29
---

# Go（Go / Golang）

> **一言で言うと:** Go は2007年に Google で **Robert Griesemer / Rob Pike / Ken Thompson**（Unix・C・UTF-8 の生みの親）が「**C++ のビルドの遅さとデプロイの複雑さ**」への反発から設計した言語。**「Less is exponentially more」（少なさは指数的な力になる）** という Pike の哲学のもと、機能を意図的に削ぎ落とし、**Goroutine と Channel による CSP 並行モデル**・**構造的型インターフェース**・**強制された gofmt による単一フォーマット**を中核に据えた。クラウドインフラ（Docker / Kubernetes / Terraform）と高負荷 API サーバーの主流言語となり、2026年2月の Go 1.26 では Green Tea GC がデフォルト化され、cgo オーバーヘッドが30%削減された。

## 誕生と歴史的経緯

| 年月 | 主な転換点 |
|---|---|
| 2007 | Google 社内で Griesemer / Pike / Thompson が設計開始（C++ ビルド待ちへの不満が動機） |
| 2009 | OSS 化（BSD ライセンス） |
| 2012 | 1.0 リリース（Go 1 promise = 後方互換の約束） |
| 2014 | Docker / Kubernetes が Go 採用、クラウド時代の中心言語へ |
| 2015 | `context` パッケージ標準化（キャンセル伝播の標準 API） |
| 2018 | Go modules 導入（1.11、GOPATH 時代の終わり） |
| 2022 | Generics 導入（1.18、10年越しの議論決着） |
| 2024-02 | 1.22 / Range over int、net/http ルーティング改善 |
| 2024-08 | 1.23 / Range over function（カスタムイテレータ） |
| 2025-02 | 1.24 / Generic type aliases、Swiss Tables maps（用途により30〜60%高速化） |
| 2025-08 | 1.25 / Green Tea GC 実験的、JSON v2 実験版 |
| **2026-02** | **1.26 / Green Tea GC デフォルト化、cgo 30%高速化、goroutine leak profile** |

### 設計者と動機

Go の設計者3名はいずれも Bell Labs の系譜:

- **Ken Thompson** — Unix と B言語の作者、C 言語の設計者の一人、UTF-8 設計
- **Rob Pike** — Plan 9 OS と Limbo 言語の設計者、UTF-8 共同設計者
- **Robert Griesemer** — Java HotSpot コンパイラ、V8 開発の経験

開発の発端は2007年9月、Google 社内で **C++ プログラムのビルドが終わるのを待っている時間**に、Pike と Griesemer が「**もっとシンプルで、コンパイルが速く、並行プログラミングが簡単な言語が欲しい**」と話したこと。Thompson が加わって設計が始まった。

### 解決しようとした問題

Google 内部の問題意識（2007年時点）:

| 問題 | 既存言語の状況 | Go の解 |
|------|------------|---------|
| C++のビルドが遅い | テンプレートと include で数十分 | 依存関係を明示・パッケージ単位の高速ビルド |
| 並行プログラミングが難しい | スレッド・mutex が複雑 | Goroutine + Channel（CSP） |
| デプロイが複雑 | 依存ライブラリの動的リンク地獄 | 静的単一バイナリ |
| 可読性が下がる | 言語機能が多すぎて方言化 | 機能を意図的に削減・gofmtで統一 |
| GCの停止時間が長い | Java の Stop-the-world | 低レイテンシ並行GC |

これらの判断が、その後の**クラウドネイティブ時代（Docker・Kubernetes・Prometheus・Terraform）の中心言語**となる土台となった。

### バージョン進化の山場

| バージョン | 年 | 主な貢献 |
|---|---|---|
| 1.0 | 2012 | 初の安定版。**Go 1 promise**（後方互換の約束） |
| 1.5 | 2015 | コンパイラとランタイムを Go 自身で書き直し（self-hosting） |
| 1.7 | 2016 | context パッケージ標準化 |
| 1.11 | 2018 | **Go modules** 導入（GOPATH 不要に） |
| 1.13 | 2019 | エラーラップ（`errors.Is`/`As`/`Unwrap`） |
| 1.18 | 2022 | **Generics 導入**（10年越し） |
| 1.21 | 2023 | `min`/`max`/`clear` 組み込み、`slices`/`maps`/`cmp` 標準ライブラリ |
| 1.22 | 2024 | **Range over int**、net/http ルーティング強化 |
| 1.23 | 2024 | **Range over function**（カスタムイテレータ） |
| 1.24 | 2025-02 | Generic type aliases、Swiss Tables maps（用途により30〜60%高速化） |
| 1.25 | 2025-08 | Green Tea GC（実験的、10-40% GC オーバーヘッド削減） |
| **1.26** | **2026-02** | **Green Tea GC デフォルト化、cgo 30%高速化、goroutine leak profile** |

## 設計思想

### 1. "Less is exponentially more"（Rob Pike, 2012）

Pike の有名な講演タイトル。**機能を増やすことではなく、減らすことが言語の力を高める**という哲学。Goには**意図的に存在しないもの**が多数ある:

| 他言語にあって Go にないもの | 理由 |
|---|---|
| クラスと継承 | 継承の濫用が複雑性を生む。コンポジションで十分 |
| 例外 | 制御フローを隠す。明示的なエラー戻り値が読みやすい |
| 三項演算子 | `if-else` で十分。読みやすさを優先 |
| マクロ・メタプログラミング | コードを読めば動作が分かるべき |
| 演算子オーバーロード | `+` の意味が文脈で変わるのは混乱 |
| `while` ループ | `for` で全部書ける |
| `?` 構文糖（Optional等） | 明示的に書く |

**「言語仕様書が短い」ことを誇る言語**は珍しい。Go の言語仕様は印刷して50ページ程度（C++ は1000ページ超）。

### 2. Convention over Configuration

`gofmt` という標準フォーマッタがあり、**スタイルの議論を始める前に終わらせる**:

```go
// gofmt で常にこうなる（タブインデント、特定の括弧位置）
func Add(a int, b int) int {
	return a + b
}
```

ESLint/Prettier の設定で論争する JS 系言語と対照的。Go コミュニティは「コードレビューでスタイルの議論をしない」文化が定着している。

### 3. 並行性のファーストクラス対応（CSP モデル）

Tony Hoare の **CSP（Communicating Sequential Processes、1978）** を言語レベルで実装した数少ない言語:

> **Don't communicate by sharing memory; share memory by communicating.**
> （メモリ共有でやり取りするな、やり取りでメモリを共有しろ）

Goroutine（軽量スレッド）と Channel（型安全なメッセージパッシング）が、Mutex の上を行くシンプルさで並行プログラミングを書ける。詳細は[[並行性の基本概念]]も参照。

### 4. コンパイル速度と単一バイナリ

```bash
# Goプログラムは1コマンドで単一バイナリにコンパイル
$ go build -o myapp main.go

# 結果: 依存ライブラリがすべて埋め込まれた単一実行ファイル
$ file myapp
myapp: ELF 64-bit LSB executable, x86-64, statically linked

# Dockerイメージが極小化できる
FROM scratch  # 何も入っていないベースイメージ
COPY myapp /
ENTRYPOINT ["/myapp"]
# → 数MB の Docker image で本番デプロイ可能
```

これが Docker / Kubernetes と相性が抜群な理由。詳細は[[Dockerイメージ]]を参照。

## 型システム

### 静的型 + 構造的インターフェース

```go
// インターフェースは「メソッドの集合」
type Reader interface {
    Read(p []byte) (n int, err error)
}

// 明示的な implements 宣言は不要
type FileReader struct {
    f *os.File
}

func (r *FileReader) Read(p []byte) (n int, err error) {
    return r.f.Read(p)
}

// FileReader は Read メソッドを持つので、自動的に Reader として扱える
var r Reader = &FileReader{f: file}
```

[[TypeScript]] のようにプロパティで判定するのではなく、**メソッドシグネチャの集合**で判定する。詳細は[[インターフェース]]を参照。

### Generics（1.18+）

10年越しの議論を経て2022年に導入:

```go
// 型パラメータ T （any は interface{} のエイリアス）
func Map[T, U any](s []T, f func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = f(v)
    }
    return result
}

// 使用
nums := []int{1, 2, 3}
doubled := Map(nums, func(n int) int { return n * 2 })
// [2, 4, 6]

// 型制約 — 「Ordered」のような制約を組み合わせ
import "cmp"

func Max[T cmp.Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}

Max(1, 2)        // 2 (int)
Max(3.14, 2.71)  // 3.14 (float64)
Max("a", "b")    // "b" (string)
```

**1.24 では Generic Type Aliases も追加**:

```go
type Pair[A, B any] = struct {
    First  A
    Second B
}
```

詳細は[[ジェネリクス]]を参照。

### ポインタ — 値か参照かを明示する

```go
type Counter struct { count int }

// 値レシーバ — 値のコピーに対して操作（変更は呼び出し元に反映されない）
func (c Counter) Get() int { return c.count }

// ポインタレシーバ — ポインタ経由で本体を操作
func (c *Counter) Increment() { c.count++ }

c := Counter{}
c.Increment() // *Counter のメソッドを値に対して呼べる（自動で &c に変換）
fmt.Println(c.count) // 1
```

詳細は[[Goのポインタ]]を参照。

## 並行モデル — Goroutine と Channel

### Goroutine — 軽量スレッド

```go
// go キーワードで関数を別の Goroutine で実行
go fetch("https://example.com")

// 数万〜数十万の Goroutine が現実的に動く（メモリは2KB程度から）
for i := 0; i < 100000; i++ {
    go work(i)
}
```

OS スレッドとの違い:

| 観点 | OS スレッド | Goroutine |
|------|----------|-----------|
| メモリ | 1〜8 MB | 2 KB（必要に応じて拡張） |
| 切り替えコスト | 高い（カーネル経由） | 低い（ユーザー空間） |
| 数の上限 | 数千 | 数百万 |
| スケジューラ | OS | Go ランタイム（M:N モデル） |

### Channel — 型付きメッセージパッシング

```go
ch := make(chan int)

// 送信
go func() {
    ch <- 42  // chan に送信
}()

// 受信
value := <-ch  // chan から受信（送信があるまでブロック）
fmt.Println(value)  // 42

// バッファ付き channel
buf := make(chan int, 3)
buf <- 1
buf <- 2
buf <- 3  // バッファが満杯になるまでブロックしない
```

### select — 複数 channel の多重化

```go
// タイムアウト付きの待ち合わせ
select {
case data := <-ch1:
    fmt.Println("ch1から:", data)
case data := <-ch2:
    fmt.Println("ch2から:", data)
case <-time.After(5 * time.Second):
    fmt.Println("タイムアウト")
case <-ctx.Done():
    fmt.Println("コンテキストキャンセル")
    return
}
```

### 実用パターン: Worker Pool

```go
func workerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }()
    }

    go func() {
        wg.Wait()
        close(results)
    }()
}

// 使用
jobs := make(chan Job, 100)
results := make(chan Result, 100)

go workerPool(10, jobs, results) // ワーカー10個並列

// ジョブ投入
for _, j := range incomingJobs {
    jobs <- j
}
close(jobs)

// 結果受信
for r := range results {
    handle(r)
}
```

これが Goroutine + Channel の真骨頂。Java/Python の Thread + Lock では遥かに複雑になる。

### Context — キャンセル伝播の標準

```go
import "context"

func handler(w http.ResponseWriter, r *http.Request) {
    ctx, cancel := context.WithTimeout(r.Context(), 3*time.Second)
    defer cancel()

    result, err := slowOperation(ctx)
    if err != nil {
        http.Error(w, err.Error(), http.StatusGatewayTimeout)
        return
    }
    w.Write(result)
}

func slowOperation(ctx context.Context) ([]byte, error) {
    select {
    case data := <-fetchData():
        return data, nil
    case <-ctx.Done():
        return nil, ctx.Err() // タイムアウトまたはキャンセル
    }
}
```

**Go の慣習: 関数の最初の引数は `ctx context.Context`**。リクエストキャンセルやタイムアウトが綺麗に伝播する。

## 代表的なイディオム

### エラーハンドリング — 明示的な return

```go
func readConfig(path string) (*Config, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, fmt.Errorf("open config: %w", err) // %w でラップ
    }
    defer f.Close()

    var cfg Config
    if err := json.NewDecoder(f).Decode(&cfg); err != nil {
        return nil, fmt.Errorf("decode config: %w", err)
    }

    return &cfg, nil
}

// 呼び出し側
cfg, err := readConfig("/etc/app.json")
if err != nil {
    log.Fatal(err)
}

// エラーチェーンの検査（1.13+）
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println("Path error:", pathErr.Path)
}
```

**Go の `if err != nil` ボイラープレート**は批判の的だが、設計者は意図的にこの形式を選んでいる。例外と異なり「**どこでエラーが発生し、どこでハンドルされるか**」がコードを読むだけで分かる。

詳細は[[エラーハンドリングとフォールバックの設計戦略]]を参照。

### インターフェース — 小さく定義する

```go
// 標準ライブラリの実例: io パッケージ
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// コンポジション
type ReadWriter interface {
    Reader
    Writer
}

// io.Copy は Reader と Writer だけを要求する
func Copy(dst Writer, src Reader) (written int64, err error) {
    // ...
}

// File も bytes.Buffer も net.Conn も全部使える
io.Copy(file, conn)
io.Copy(buffer, file)
```

**「インターフェースは利用側で定義する」のがGoらしさ**。`UserRepository` のような巨大インターフェースを実装側で定義するのではなく、必要なメソッドだけを利用側で抽出する。

### defer — スコープ終了時の処理

```go
func processFile(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // 関数終了時に必ず呼ばれる

    mu.Lock()
    defer mu.Unlock()

    // 処理...
    return nil
}
```

リソース解放を「**確保直後に書く**」ことで漏れを防ぐ。

### 標準ライブラリの強さ — ほぼ素のままで動く Web サーバー

```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func main() {
    // 1.22+ で導入された強化されたルーティング
    mux := http.NewServeMux()
    mux.HandleFunc("GET /users/{id}", func(w http.ResponseWriter, r *http.Request) {
        id := r.PathValue("id")
        json.NewEncoder(w).Encode(User{ID: 1, Name: "Alice (ID: " + id + ")"})
    })

    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

**Express.js や FastAPI のような外部フレームワークなしで、本番運用可能な Web サーバーが書ける**。これが Go の魅力の一つ。

## エコシステム

### Go modules（1.11+）

```bash
# 新規プロジェクト
$ go mod init github.com/user/myapp

# go.mod に依存が記録される
require (
    github.com/gorilla/mux v1.8.1
    golang.org/x/crypto v0.32.0
)

# 依存をダウンロード
$ go mod tidy

# 1.24 で追加: ツールも go.mod で管理
go tool stringer -type=Pill
```

### 主要なツール群

```bash
go build      # コンパイル
go test       # テスト実行
go vet        # 静的解析
go fmt        # フォーマット（gofmt のラッパー）
go doc        # ドキュメント表示
go mod tidy   # 依存関係整理
go work       # マルチモジュール（モノレポ）
```

**すべて標準ツール**。Linter（`staticcheck`）と Test ヘルパー（`testify`）以外、サードパーティに頼らないでも開発できる。

### 主要 Web フレームワーク

| フレームワーク | 特徴 |
|--------------|------|
| **net/http**（標準） | 1.22+ で十分強力、外部依存なし |
| **chi** | net/http 互換のミドルウェアルーター、軽量 |
| **gin** | 高速・JSON API 向け |
| **echo** | gin に似た書き味、よく整理された API |
| **fiber** | Express.js ライク、Fasthttp ベース |

GraphQL なら **gqlgen**、ORM なら **GORM** や **sqlc**（生 SQL → 型付きコード生成）。

### ガベージコレクションの進化

Go の GC は**低レイテンシ並行 GC**で、stop-the-world は**通常1ms以下**:

- 1.5: 並行マークスイープ導入
- 1.8: 平均 stop-the-world < 100μs
- 1.25: **Green Tea GC**（実験的、10-40% GC オーバーヘッド削減）
- **1.26: Green Tea GC がデフォルト化**

これが「**GC ありなのにシステムプログラミングに使える**」と言われる理由。

## よくある落とし穴

### 1. nil interface vs nil concrete

Go 最大の罠の一つ:

```go
type MyError struct{ Code int }
func (e *MyError) Error() string { return fmt.Sprintf("err: %d", e.Code) }

func process() error {
    var err *MyError = nil // 具象型が nil
    return err // ❌ interface としては nil ではない！
}

err := process()
if err != nil {
    fmt.Println("失敗:", err) // ✅ こちらに入る — 直感に反する
}
```

理由: interface 値は「**型情報 + 値**」のペアで、型情報が `*MyError` で値が `nil` の状態は、interface 全体としては nil ではない。

```go
// ✅ 正しい書き方
func process() error {
    if condition {
        return &MyError{Code: 1}
    }
    return nil // 直接 nil を返す
}
```

### 2. Goroutine リーク

```go
// ❌ ch に送信側がいなくなると、永遠に待ち続ける
func worker() {
    ch := make(chan int)
    go func() {
        result := <-ch // ch に何も送られず、Goroutine がリーク
        fmt.Println(result)
    }()
}

// ✅ context でキャンセル可能にする
func worker(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case result := <-ch:
            fmt.Println(result)
        case <-ctx.Done():
            return // 確実に終了
        }
    }()
}
```

**1.26 から `goroutineleak` プロファイルでリークを検出可能**。

### 3. ループ変数のクロージャキャプチャ（〜 1.21）

```go
// ❌ Go 1.21 以前の挙動: 全 Goroutine が同じ i を見る
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // すべて 5 と表示される（ループ終了後の値）
    }()
}

// 旧来の対策
for i := 0; i < 5; i++ {
    i := i // ループごとに新しい変数を作る
    go func() { fmt.Println(i) }()
}

// ✅ Go 1.22+ — ループごとに新しい変数になる仕様変更
// 上記の旧コードもそのまま正しく動く
```

**Go 1.22 で言語仕様が変わり、ループ変数は反復ごとに新しいスコープに**。古いチュートリアルの workaround コードはもう不要。

### 4. nil map への書き込み

```go
var m map[string]int
m["key"] = 1 // ❌ panic: assignment to entry in nil map

// ✅ make で初期化が必要
m = make(map[string]int)
m["key"] = 1
```

slice は nil でも append できる（自動的に確保される）が、map は make が必要。この非対称性は initial Go プログラマがハマる。

### 5. 文字列とバイトとルーンの混同

```go
s := "こんにちは"

// バイト列としての長さ（UTF-8）
len(s) // 15（日本語1文字 = 3バイト × 5）

// ルーン（文字）としての長さ
utf8.RuneCountInString(s) // 5

// 添字アクセスはバイト
s[0] // 0xe3（'こ' の最初のバイト）— 文字ではない！

// 文字単位で処理するなら range
for i, r := range s {
    fmt.Printf("%d: %c\n", i, r) // i はバイト位置、r はルーン
}
```

詳細は[[サロゲートペア・結合文字・書記素クラスタ]]も参照。

### 6. 変数シャドウイング

```go
// ❌ シャドウィングで err を捨ててしまう
data, err := readFile()
if err != nil {
    // ...
}

if cfg, err := parseConfig(data); err != nil { // 新しい err を作っている
    return err
}
// ↓ ここの err は readFile の err（= nil の可能性）
fmt.Println(err)
```

`go vet -shadow` で検出可能。

### 7. デフォルトの GOMAXPROCS が cgroup を無視（〜 1.24）

```go
// 旧来: コンテナで CPU 制限していても、ノード全体の CPU 数を見ていた
runtime.NumCPU() // 例: 物理ホストが 64コアだと 64 が返る
// → 不要な goroutine 過剰スケジューリング

// Go 1.25+ — container-aware GOMAXPROCS（cgroup を見る）
runtime.NumCPU() // コンテナの cpu limit に応じた値
```

Kubernetes 環境で重要な改善。

## AIによる実装のアンチパターン

| アンチパターン | なぜ問題か | 対策 |
|---|---|---|
| `interface{}` を多用 | 型安全性を失う、Java の Object 的な使い方は Go らしくない | Generics (1.18+) を使う |
| 巨大なインターフェース定義 | 「UserRepository に20個メソッド」のような Java 風 | 小さなインターフェースを利用側で定義 |
| `panic` を例外のように使う | Go の panic はプロセスを落とす、recoverable にすべきではない | 通常エラーは `error` 戻り値で表現 |
| 構造体に多数のフィールド + 多数のメソッド | OOP の継承を真似ようとしている | コンポジションで小さく分割 |
| `goroutine` を `await` のように軽く使う | リーク・競合の原因 | 必ず終了戦略（channel 受信 or ctx 監視）を持つ |
| エラーをそのまま `return err` でバブル | 文脈が失われ、デバッグ困難 | `fmt.Errorf("操作の説明: %w", err)` でラップ |
| 値レシーバとポインタレシーバを混在 | Linter 警告、混乱の元 | 構造体ごとに統一（基本はポインタレシーバ） |
| `defer` をループ内で使う | リソース解放がループ終了まで遅延 | 関数を分けるか、明示的に Close() |
| `time.Sleep` で同期 | テストが遅い、競合状態を隠す | `chan` または `sync.WaitGroup` で待ち合わせ |
| `fmt.Println` でデバッグ | 本番でも残ってしまう | `log/slog`（標準、構造化ログ）を使う |

## 関連トピック

- [[プログラミング言語の系譜と選択]] — 親トピック
- [[Goのポインタ]] — Go のポインタ詳細
- [[並行性の基本概念]] — Goroutine + Channel の理論的背景（CSP）
- [[ロック]] / [[セマフォとミューテックスの比較]] — 並行制御の基礎
- [[インターフェース]] — Go の構造的interfaceの位置づけ
- [[ジェネリクス]] — Go 1.18+ の Generics
- [[エラーハンドリングとフォールバックの設計戦略]] — Go のエラーハンドリング哲学
- [[Dockerイメージ]] — Go の単一バイナリと相性の良い基盤
- [[サロゲートペア・結合文字・書記素クラスタ]] — Go の文字列処理（UTF-8）

## 参考リソース

- [Go公式ドキュメント](https://go.dev/doc/) — 言語仕様・標準ライブラリ
- [Effective Go](https://go.dev/doc/effective_go) — Go チームによる慣用句集
- [Go 1.26 リリースノート](https://go.dev/doc/go1.26)
- [Tour of Go](https://go.dev/tour/) — インタラクティブなチュートリアル
- [Go by Example](https://gobyexample.com/) — コード例で学ぶ
- [Less is exponentially more (Rob Pike, 2012)](https://commandcenter.blogspot.com/2012/06/less-is-exponentially-more.html) — 設計思想の原典
- [The Go Programming Language (Donovan & Kernighan)](https://www.gopl.io/) — 公式の教科書、Kernighanは『プログラミング言語C』の共著者
- 書籍:『Go言語による並行処理』(Katherine Cox-Buday, O'Reilly) — 並行モデルの深掘り
- 書籍:『Go言語プログラミングエッセンス』(mattn) — 日本語の実践書

## 学習メモ

- Go の真価は「**書いて2年経ったコードが、今書いたかのように読める**」こと。後方互換の徹底（Go 1 promise）と機能の少なさが、長期保守の負担を劇的に下げる
- Goroutine + Channel は強力だが、**並行性の問題が消えるわけではない**。データ競合や [[デッドロック]] は普通に起きる。CSP モデルは「設計を強制する」ものではなく「設計しやすくする」もの
- Generics（1.18）導入時、Go コミュニティで「Go らしさが失われる」という議論があったが、結果として `slices` / `maps` / `cmp` 等の標準ライブラリで自然に使われる落としどころに収まった
- 1.26 の Green Tea GC デフォルト化と cgo 30%高速化は、**「Go は性能では Rust に勝てない」という通説を揺らがせる**水準。クラウドネイティブ領域での採用は今後さらに進む
- AI コード生成が苦手とする領域: 「**Goらしい小さなインターフェース設計**」「**並行プリミティブの正しい使い分け**」。表面的な構文は生成できるが、設計判断は人間のレビューが必須
