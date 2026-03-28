---
layer: 4
parent: "[[ルーティングとミドルウェア]]"
type: detail
created: 2026-03-28
---

# Radix Treeとルーター（Radix Tree / Compact Trie）

> **一言で言うと:** Radix Tree（基数木）はトライ（Trie）の圧縮版で、共通プレフィックスを1つのノードにまとめることでメモリ効率と探索速度を両立するデータ構造。Go の高性能HTTPルーター（Chi, Gin, Fiber, Echo）はすべてRadix Treeを使ってURLパスをマッチングしている。

## トライ（Trie）からRadix Treeへ

### トライの問題点

トライ（Trie / Prefix Tree）は文字列の検索に特化した木構造で、各ノードが1文字を表す。URLパスのマッチングに使えるが、共通プレフィックスの長い部分で**1文字ごとにノードを作る**ため、無駄が大きい。

```mermaid
graph TD
    subgraph "トライ — ノード数が多い"
        T_ROOT["(root)"]
        T_a1["/"]
        T_a2["a"]
        T_a3["p"]
        T_a4["i"]
        T_a5["/"]
        T_u["u"]
        T_s["s"]
        T_e["e"]
        T_r["r"]
        T_s2["s"]
        T_p["p"]
        T_o["o"]
        T_s3["s"]
        T_t["t"]
        T_s4["s"]

        T_ROOT --> T_a1
        T_a1 --> T_a2
        T_a2 --> T_a3
        T_a3 --> T_a4
        T_a4 --> T_a5
        T_a5 --> T_u
        T_u --> T_s
        T_s --> T_e
        T_e --> T_r
        T_r --> T_s2
        T_a5 --> T_p
        T_p --> T_o
        T_o --> T_s3
        T_s3 --> T_t
        T_t --> T_s4
    end
```

### Radix Tree — 共通プレフィックスの圧縮

Radix Tree（Patricia Trie とも呼ばれる）は、**分岐のないノード列を1つのノードに圧縮**する。

```mermaid
graph TD
    subgraph "Radix Tree — ノードを圧縮"
        R_ROOT["/api/"]
        R_U["users"]
        R_P["posts"]

        R_ROOT --> R_U
        R_ROOT --> R_P
    end

    style R_ROOT fill:#e3f2fd,stroke:#1565c0
    style R_U fill:#e8f5e9,stroke:#2e7d32
    style R_P fill:#e8f5e9,stroke:#2e7d32
```

15ノードが3ノードに圧縮された。実際のHTTPルーターでは数百のルートを登録しても、共通プレフィックス（`/api/v1/`等）が圧縮されるため、木の深さが浅く保たれる。

## HTTPルーターでの活用

### ルート登録とツリー構築

以下のルートを登録した場合のRadix Treeの構造を見てみる:

```
GET  /api/users
GET  /api/users/:id
POST /api/users
GET  /api/users/:id/posts
GET  /api/posts
GET  /api/posts/:id
GET  /health
```

```mermaid
graph TD
    ROOT["(root)"]
    API["/api/"]
    HEALTH["/health<br/>→ healthHandler"]

    USERS["users"]
    POSTS_TOP["posts"]

    USERS_END["(end)<br/>→ listUsersHandler<br/>→ createUserHandler"]
    USERS_PARAM["/:id"]
    USERS_PARAM_END["(end)<br/>→ getUserHandler"]
    USERS_PARAM_POSTS["/posts<br/>→ getUserPostsHandler"]

    POSTS_END["(end)<br/>→ listPostsHandler"]
    POSTS_PARAM["/:id<br/>→ getPostHandler"]

    ROOT --> API
    ROOT --> HEALTH
    API --> USERS
    API --> POSTS_TOP
    USERS --> USERS_END
    USERS --> USERS_PARAM
    USERS_PARAM --> USERS_PARAM_END
    USERS_PARAM --> USERS_PARAM_POSTS
    POSTS_TOP --> POSTS_END
    POSTS_TOP --> POSTS_PARAM

    style ROOT fill:#fff,stroke:#333
    style API fill:#e3f2fd,stroke:#1565c0
    style HEALTH fill:#e8f5e9,stroke:#2e7d32
    style USERS fill:#fff3e0,stroke:#e65100
    style POSTS_TOP fill:#fff3e0,stroke:#e65100
    style USERS_PARAM fill:#fce4ec,stroke:#c62828
    style POSTS_PARAM fill:#fce4ec,stroke:#c62828
```

### ルート検索の流れ

`GET /api/users/42/posts` を検索する場合:

```mermaid
flowchart LR
    A["入力: /api/users/42/posts"] --> B["ルート: '/'"]
    B -->|"'/api/' にマッチ"| C["ノード: '/api/'"]
    C -->|"'users' にマッチ"| D["ノード: 'users'"]
    D -->|"'/42' → パラメータ :id=42"| E["ノード: '/:id'"]
    E -->|"'/posts' にマッチ"| F["ノード: '/posts'\nハンドラ発見"]

    style F fill:#4caf50,color:#fff
```

4ノードの走査でハンドラに到達する。ルート数に関わらず、パスの深さ（セグメント数）に比例する計算量で済む。

## 線形探索との性能比較

| 観点 | 線形探索（Express） | Radix Tree（Chi, Gin） |
|------|-------------------|----------------------|
| 検索の計算量 | O(n) — 登録ルート数に比例 | O(k) — URLのセグメント数に比例 |
| 100ルート、パス深さ3 | 最悪100回の比較 | 最悪3〜4ノードの走査 |
| 1000ルート、パス深さ3 | 最悪1000回の比較 | **変わらず** 3〜4ノードの走査 |
| パラメータ抽出 | 正規表現マッチング | ツリー走査中に直接抽出 |
| メモリ使用量 | ルートごとにパターンを保持 | 共通プレフィックスを共有 |

**Expressが線形探索でも問題にならない理由:** 一般的なWebアプリケーションのルート数は数十〜数百程度であり、この規模ではRadix Treeと線形探索の差は数マイクロ秒にすぎない。ルーティングのオーバーヘッドがボトルネックになることは稀。Radix Treeが真に効果を発揮するのは、APIゲートウェイのように数千〜数万のルートを持つシステムや、1秒間に数十万リクエストを処理する高スループット環境。

## パラメータとワイルドカードの実装

Radix Treeベースのルーターは、パスパラメータ（`:id`）やワイルドカード（`*`）を特別なノードとして扱う。

```mermaid
graph TD
    ROOT["(root)"]
    API["/api/"]
    FILES["/files/"]

    USERS["users/"]
    PARAM_NODE[":id<br/>（パラメータノード）<br/>任意の1セグメントにマッチ"]
    WILDCARD["*filepath<br/>（ワイルドカードノード）<br/>残り全セグメントにマッチ"]

    ROOT --> API
    ROOT --> FILES
    API --> USERS
    USERS --> PARAM_NODE
    FILES --> WILDCARD

    style PARAM_NODE fill:#fce4ec,stroke:#c62828
    style WILDCARD fill:#f3e5f5,stroke:#7b1fa2
```

| ノード種別 | パターン例 | マッチ例 | 優先度 |
|-----------|----------|---------|--------|
| 静的（Static） | `/users/new` | `/users/new` のみ | **最高** |
| パラメータ（Param） | `/users/:id` | `/users/42`, `/users/abc` | 中 |
| ワイルドカード（Catch-all） | `/files/*filepath` | `/files/a/b/c.txt` | **最低** |

**優先度の原則:** 静的ノード > パラメータノード > ワイルドカードノード。これにより、`/users/new` と `/users/:id` が共存しても、`/users/new` へのリクエストは常に静的マッチが優先される。Expressの「定義順依存」の問題がRadix Treeでは構造的に解消される。

## コード例

### Go — Radix Treeの簡易実装（ルーターの原理）

```go
package main

import (
	"fmt"
	"strings"
)

type nodeType int

const (
	static   nodeType = iota // "/users"
	param                    // ":id"
	catchAll                 // "*filepath"
)

type node struct {
	path     string
	ntype    nodeType
	children []*node
	handler  string // 簡略化: ハンドラ名を文字列で持つ
}

// ルートを追加する（簡略版）
func (n *node) addRoute(path, handler string) {
	segments := strings.Split(strings.Trim(path, "/"), "/")
	current := n

	for _, seg := range segments {
		var child *node
		for _, c := range current.children {
			if c.path == seg {
				child = c
				break
			}
		}
		if child == nil {
			nt := static
			if strings.HasPrefix(seg, ":") {
				nt = param
			} else if strings.HasPrefix(seg, "*") {
				nt = catchAll
			}
			child = &node{path: seg, ntype: nt}
			current.children = append(current.children, child)
		}
		current = child
	}
	current.handler = handler
}

// パスを検索してハンドラとパラメータを返す
func (n *node) search(path string) (string, map[string]string) {
	segments := strings.Split(strings.Trim(path, "/"), "/")
	params := make(map[string]string)
	current := n

	for _, seg := range segments {
		var matched *node

		// 優先度: static > param > catchAll
		for _, c := range current.children {
			if c.ntype == static && c.path == seg {
				matched = c
				break
			}
		}
		if matched == nil {
			for _, c := range current.children {
				if c.ntype == param {
					params[c.path[1:]] = seg // ":id" → "id"
					matched = c
					break
				}
			}
		}
		if matched == nil {
			return "", nil
		}
		current = matched
	}
	return current.handler, params
}

func main() {
	root := &node{path: "/"}

	root.addRoute("/api/users", "listUsers")
	root.addRoute("/api/users/new", "newUserForm")
	root.addRoute("/api/users/:id", "getUser")
	root.addRoute("/api/posts/:id", "getPost")

	tests := []string{
		"/api/users",
		"/api/users/new",  // staticが優先される
		"/api/users/42",   // paramにマッチ
		"/api/posts/7",
		"/api/unknown",    // マッチしない
	}

	for _, path := range tests {
		handler, params := root.search(path)
		if handler != "" {
			fmt.Printf("%-25s → %s %v\n", path, handler, params)
		} else {
			fmt.Printf("%-25s → 404\n", path)
		}
	}
	// /api/users                → listUsers map[]
	// /api/users/new            → newUserForm map[]
	// /api/users/42             → getUser map[id:42]
	// /api/posts/7              → getPost map[id:7]
	// /api/unknown              → 404
}
```

### TypeScript — Radix Treeベースのルート検索

```typescript
type NodeType = 'static' | 'param' | 'catchAll';

interface RouteNode {
  path: string;
  type: NodeType;
  children: RouteNode[];
  handler?: string;
}

function createNode(path: string, handler?: string): RouteNode {
  let type: NodeType = 'static';
  if (path.startsWith(':')) type = 'param';
  else if (path.startsWith('*')) type = 'catchAll';
  return { path, type, children: [], handler };
}

function addRoute(root: RouteNode, path: string, handler: string): void {
  const segments = path.split('/').filter(Boolean);
  let current = root;

  for (const seg of segments) {
    let child = current.children.find(c => c.path === seg);
    if (!child) {
      child = createNode(seg);
      current.children.push(child);
    }
    current = child;
  }
  current.handler = handler;
}

function search(root: RouteNode, path: string): { handler: string; params: Record<string, string> } | null {
  const segments = path.split('/').filter(Boolean);
  const params: Record<string, string> = {};
  let current = root;

  for (const seg of segments) {
    // 優先度順に探索: static → param → catchAll
    const staticMatch = current.children.find(c => c.type === 'static' && c.path === seg);
    if (staticMatch) {
      current = staticMatch;
      continue;
    }

    const paramMatch = current.children.find(c => c.type === 'param');
    if (paramMatch) {
      params[paramMatch.path.slice(1)] = seg;
      current = paramMatch;
      continue;
    }

    return null; // マッチしない
  }

  return current.handler ? { handler: current.handler, params } : null;
}

// 使用例
const root = createNode('');
addRoute(root, '/api/users', 'listUsers');
addRoute(root, '/api/users/new', 'newUserForm');
addRoute(root, '/api/users/:id', 'getUser');

console.log(search(root, '/api/users'));       // { handler: 'listUsers', params: {} }
console.log(search(root, '/api/users/new'));    // { handler: 'newUserForm', params: {} }
console.log(search(root, '/api/users/42'));     // { handler: 'getUser', params: { id: '42' } }
console.log(search(root, '/api/unknown'));      // null
```

## よくある落とし穴

### 1. パラメータルートと静的ルートの衝突に気づかない

`/users/new`（静的）と `/users/:id`（パラメータ）を両方登録すると、一部のルーターはエラーを出す（Gin）か、優先度ルールで解決する（Chi）。フレームワークごとの衝突解決ルールを把握していないと、意図しないハンドラに到達する。

### 2. 末尾スラッシュの扱い

`/api/users` と `/api/users/` を別のルートとして扱うかどうかはルーターの実装依存。多くの高性能ルーターは自動リダイレクトオプションを提供しているが、デフォルトの挙動を確認しないと404が発生する。

```go
// Chi: 末尾スラッシュの自動リダイレクト
r.Use(middleware.RedirectSlashes)
```

### 3. Radix Treeの制約 — 正規表現パターンの限界

Radix Treeベースのルーターは、パスパラメータに正規表現制約を付けにくい。`/users/:id` で `:id` が数値のみであることを保証するには、ルーターの外（ミドルウェアやハンドラ）で[[バリデーション]]する必要がある。

```go
// ❌ Radix Treeルーターでは一般に正規表現制約を書けない
r.Get("/users/{id:[0-9]+}", handler) // Chiはこの記法をサポート（例外的）

// ✅ バリデーションミドルウェアで対応
r.With(validateNumericID).Get("/users/{id}", handler)
```

### 4. ルーター性能がボトルネックだと思い込む

前述の通り、ルーティングの処理時間は通常マイクロ秒単位。DB クエリ（ミリ秒）や外部API呼び出し（数十〜数百ミリ秒）と比べて無視できるほど小さい。ルーターの性能差を理由にフレームワークを選定するのは、ほとんどの場合で間違い。

## 関連トピック

- [[ルーティングとミドルウェア]] — 親トピック。ルーターの役割と設計パターン
- [[B-TreeとB+Tree]] — 同じ木構造でもB-Treeはディスク I/O 最適化、Radix Treeはメモリ上の文字列検索に特化。問題領域が異なる
- [[データ構造とアルゴリズム]] — トライ、木構造の基礎知識
- [[バリデーション]] — Radix Treeでは表現できないパラメータ制約をミドルウェアで補完する

## 参考リソース

- go-chi/chi ソースコード — `tree.go` にRadix Tree実装がある
- julienschmidt/httprouter — Go初期の高性能ルーター。Radix Tree実装の参考実装として有名
- 『Introduction to Algorithms』（CLRS）— Trie / Radix Tree の理論的背景
