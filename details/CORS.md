---
layer: 6
parent: "[[CORS]]"
type: detail
created: 2026-03-28
---

# CORS の応用・エッジケース

> **一言で言うと:** CORS の概念・基本フロー・主要フレームワーク実装は [[CORS]] トピックにまとまっている。本ドキュメントでは、トピックでは扱いきれない応用トピック（プリフライトキャッシュのブラウザ差異、サブドメインワイルドカード、WebSocket・iframe との関係、開発環境での回避策、COOP/COEP/CORP との関係）に焦点を当てる。

## プリフライトキャッシュ（Access-Control-Max-Age）のブラウザ別上限

`Access-Control-Max-Age` でプリフライト結果のキャッシュ秒数をサーバーから指定できるが、**ブラウザ側に独自の上限値があり、それを超えた値を指定しても上限でクランプされる**。

| ブラウザ | 上限値（実装上の cap） |
|---------|---------------------|
| Chromium 系（Chrome / Edge） | 7,200 秒（2 時間） |
| Firefox | 86,400 秒（24 時間） |
| Safari | 約 600 秒（10 分）— 主要ブラウザの中で最も短い |

サーバーで `Access-Control-Max-Age: 86400` を指定しても、Chrome では 2 時間ごとにプリフライトが再送される。**「Max-Age を大きく設定したのにプリフライトが頻繁に飛ぶ」**という症状はこのブラウザ上限が原因のことが多い。

実用上は **`Max-Age: 7200`** を指定しておけば全主要ブラウザで効果がある。

## サブドメインのワイルドカード許可ができない

`Access-Control-Allow-Origin` のワイルドカード `*` は「すべてのオリジン」を意味し、**部分ワイルドカード（`https://*.example.com`）は仕様上使えない**。サブドメインを許可したい場合は、サーバー側でリクエストの `Origin` ヘッダをパターンマッチしてから動的にレスポンスを返す必要がある。

```typescript
import express, { Request, Response, NextFunction } from "express";

const app = express();

// サブドメインの許可パターン
const ORIGIN_PATTERN = /^https:\/\/([a-z0-9-]+\.)?example\.com$/;

app.use((req: Request, res: Response, next: NextFunction) => {
  const origin = req.headers.origin;
  if (origin && ORIGIN_PATTERN.test(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
    res.setHeader("Vary", "Origin"); // キャッシュ汚染防止に必須
  }
  next();
});
```

**注意点:**
- 正規表現の `.` をエスケープしないと `examplexcom` のような攻撃ドメインも一致してしまう
- ホワイトリスト方式（明示列挙）で済むなら、正規表現は使わない方が安全
- `Vary: Origin` を付け忘れると CDN/ブラウザキャッシュが他オリジン用のレスポンスを返して CORS エラーが発生する

## WebSocket における CORS の不在

WebSocket のハンドシェイクは HTTP の `Upgrade` リクエストで開始されるが、**CORS の制約は適用されない**。ブラウザは WebSocket 接続時に CORS のプリフライトを送信せず、`Access-Control-Allow-Origin` ヘッダもチェックしない。

代わりに、WebSocket リクエストには **`Origin` ヘッダ**が必ず含まれるため、**サーバー側でこのヘッダを明示的に検証する責任がある**。

```typescript
import { WebSocketServer } from "ws";
import { createServer } from "http";

const ALLOWED_ORIGINS = new Set([
  "https://app.example.com",
  "https://admin.example.com",
]);

// ws v8 系では verifyClient も使えるが、HTTP サーバーの upgrade イベントで
// 処理する書き方が現在の推奨パターン（より柔軟で型情報も扱いやすい）
const server = createServer();
const wss = new WebSocketServer({ noServer: true });

server.on("upgrade", (request, socket, head) => {
  const origin = request.headers.origin;
  if (!origin || !ALLOWED_ORIGINS.has(origin)) {
    socket.write("HTTP/1.1 403 Forbidden\r\n\r\n");
    socket.destroy();
    return;
  }
  wss.handleUpgrade(request, socket, head, (ws) => {
    wss.emit("connection", ws, request);
  });
});

server.listen(8080);
```

**「WebSocket は CORS で守られている」という誤解は危険**。CSRF と同じく、ブラウザは攻撃サイトからの WebSocket 接続をブロックしないため、サーバー側の Origin 検証が唯一の防御線となる。

## iframe + postMessage — CORS が関係しないクロスオリジン通信

クロスオリジンの `<iframe>` 内のドキュメントとは **CORS ではなく `postMessage` API でやり取りする**。同一オリジンポリシーにより iframe 内の DOM や JavaScript には直接アクセスできないが、`postMessage` は明示的にメッセージを送る仕組みとして設計されている。

```typescript
// 親ウィンドウから iframe へ送信
const iframe = document.querySelector<HTMLIFrameElement>("#child");
iframe?.contentWindow?.postMessage(
  { type: "auth", token: "abc123" },
  "https://child.example.com" // 必ず targetOrigin を指定
);

// iframe 側で受信
window.addEventListener("message", (event) => {
  // 必ず Origin を検証
  if (event.origin !== "https://parent.example.com") return;
  console.log("受信:", event.data);
});
```

**よくあるミス:**
- `postMessage` の第 2 引数に `"*"` を指定 → 任意のオリジンにメッセージが届きうる（機密情報を含む場合は致命的）
- 受信側で `event.origin` を検証していない → 攻撃者が iframe を埋め込んで偽メッセージを送れる

## 開発環境での CORS 回避策

開発時にフロント（`localhost:5173` 等）から API（`localhost:8080` 等）を呼ぶ際の CORS エラーは、**サーバー側の CORS 設定で `localhost` を許可する**のが正攻法だが、本番設定との分岐が増える。代替策として **ビルドツールの proxy 機能**を使うとサーバー側の変更が不要になる。

### Vite の proxy 設定

```typescript
// vite.config.ts
export default defineConfig({
  server: {
    proxy: {
      "/api": {
        target: "http://localhost:8080",
        changeOrigin: true,
        // /api/users → http://localhost:8080/api/users
      },
    },
  },
});
```

フロントから `/api/users` にリクエストすると、Vite の dev server が同一オリジンとしてリクエストを受け、バックエンドにプロキシする。ブラウザから見ると同一オリジン通信なので CORS は発生しない。

### 絶対にやってはいけない回避策

`chrome.exe --disable-web-security --user-data-dir=/tmp/chrome` で Chrome の同一オリジンポリシーを無効化する手法は、**開発者自身のセッションを攻撃に晒す**ので絶対に使わない。Vite/webpack-dev-server の proxy か、サーバー側で `localhost` を許可する正攻法を使う。

## COOP / COEP / CORP — クロスオリジン分離との関係

CORS は「クロスオリジン通信を許可する」仕組みだが、近年は逆に「クロスオリジンとの分離をより厳格にする」ヘッダ群が追加されている。これらは **Spectre 等のサイドチャネル攻撃**への対策として導入された。

| ヘッダ | 役割 |
|--------|------|
| `Cross-Origin-Opener-Policy` (COOP) | ブラウザのウィンドウグループを同一オリジンに限定し、`window.opener` 経由のクロスオリジンアクセスを遮断 |
| `Cross-Origin-Embedder-Policy` (COEP) | iframe・画像などの埋め込みリソースに CORP / CORS を強制 |
| `Cross-Origin-Resource-Policy` (CORP) | リソースを「同一オリジン / 同一サイト / クロスオリジン」のどこから埋め込み可能か宣言 |

`SharedArrayBuffer` や `performance.measureUserAgentSpecificMemory()` のような高精度 API は、**COOP: same-origin** + **COEP: require-corp** が両方設定されたコンテキスト（クロスオリジン分離環境）でのみ利用可能。

**重要な注意:** COOP と COEP は**ドキュメント側（HTML レスポンス）**に付与するヘッダ、CORP は**埋め込まれるリソース側（画像・スクリプト・フォント等のレスポンス）**に付与するヘッダで、適用箇所が異なる。

```http
# ドキュメント側（HTML を返すレスポンス）に付与
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

```http
# 画像・スクリプトなどリソース側のレスポンスに付与
# （CDN や静的ファイル配信側の設定）
Cross-Origin-Resource-Policy: same-site
```

CORS が「許可の仕組み」なのに対し、これらは「分離の仕組み」であり、目的が逆方向。両方を理解しておくと「なぜ CORS を設定したのに `SharedArrayBuffer` が使えないのか」のような混乱を避けられる。

## Authorization ヘッダとプリフライト

`Authorization` ヘッダを付与したリクエストは、**メソッドが GET であってもプリフライトが発生する**。これは `Authorization` が CORS の「単純ヘッダ」リストに含まれないため。

```typescript
// このリクエストはプリフライトが飛ぶ
fetch("https://api.example.com/me", {
  headers: { Authorization: `Bearer ${token}` },
});
```

サーバー側ではプリフライト応答に `Access-Control-Allow-Headers: Authorization` を含める必要がある。トピックの基本例ではこれが含まれているが、自前で CORS ミドルウェアを書く場合に忘れがちなポイント。

## 関連トピック

- [[CORS]] — Layer 6 トピック。CORS の概念・基本フロー・主要フレームワーク実装の包括的解説
- [[CSRF]] — CORS はリクエスト送信を止めないため CSRF 対策にはならない
- [[ルーティングとミドルウェア]] — CORS はミドルウェアとして実装される代表例
- [[認証と認可]] — `Allow-Credentials` と Cookie 認証・Bearer 認証の関係
