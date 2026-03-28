---
layer: 2
parent: "[[HTTP-HTTPS]]"
type: detail
created: 2026-03-28
---

# HTTPとHTTPSの違い（HTTP vs HTTPS — What TLS Actually Adds）

> **一言で言うと:** HTTPSはHTTPに[[TLS-SSL|TLS]]レイヤーを挟んだもの。プロトコルの意味論（メソッド、ヘッダ、ステータスコード）は同一だが、TLSが**機密性・完全性・真正性**を追加する。その代償はハンドシェイクのRTTコストと証明書管理の運用負荷。

## 何が同じで、何が違うか

HTTPとHTTPSの違いは「封筒に入れて手紙を送るか、はがきで送るか」に似ている。書かれている内容（HTTPメッセージ）は同じだが、封筒（TLS）があれば途中で誰にも読まれない。

```mermaid
graph TB
    subgraph HTTP["HTTP（平文）"]
        direction TB
        C1["クライアント"] -->|"GET /api/users\nAuthorization: Bearer abc123\n（誰でも読める）"| S1["サーバー"]
    end

    subgraph HTTPS["HTTPS（暗号化）"]
        direction TB
        C2["クライアント"] -->|"🔒 暗号化されたデータ\n（中間者には読めない）"| S2["サーバー"]
    end

    style HTTP fill:#ffcdd2,stroke:#c62828
    style HTTPS fill:#c8e6c9,stroke:#2e7d32
```

| 観点 | HTTP | HTTPS |
|------|------|-------|
| ポート | 80 | 443 |
| URL スキーム | `http://` | `https://` |
| 通信内容 | 平文（盗聴可能） | TLSで暗号化 |
| 改ざん検知 | なし | TLSのMAC/AEADで検知 |
| サーバー認証 | なし | 証明書で検証 |
| ブラウザ表示 | 「保護されていない通信」警告 | 鍵アイコン表示 |
| HTTP/2 対応 | 仕様上は可能だが、ブラウザがTLS必須 | ブラウザ実装のHTTP/2はTLS上のみ |
| SEO | Googleがランキングで不利に扱う | ランキングシグナルとして有利 |

## TLSが追加する3つの保護

### 1. 機密性（Confidentiality）— 盗聴の防止

クライアントとサーバー間の通信が暗号化される。Wi-Fiの盗聴、[[ISP]]によるトラフィック解析、国家レベルの監視に対して通信内容を保護する。

**HTTPの場合に見える情報:**
```
GET /api/users HTTP/1.1
Host: api.example.com
Cookie: session_id=abc123
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...

← これらがすべて平文でネットワークを流れる
```

**HTTPSの場合に見える情報:**
```
← 暗号化されたバイト列のみ。外部から分かるのは:
   - 接続先のIPアドレス
   - SNI（Server Name Indication）によるドメイン名（TLS 1.3 + ECHで隠蔽可能）
   - 転送データのおおよそのサイズ
   - 通信のタイミング
```

### 2. 完全性（Integrity）— 改ざんの検知

TLSはすべての通信にメッセージ認証コード（MAC / AEAD）を付与する。通信経路上でデータが1ビットでも変更されると検知できる。

**HTTPの場合のリスク:**
- ISPがHTMLに広告スクリプトを注入する（実際に発生した事例がある）
- 公共Wi-Fiでレスポンスの内容を書き換えてフィッシングサイトに誘導する
- ソフトウェアのダウンロードを悪意あるバイナリにすり替える

### 3. 真正性（Authenticity）— なりすましの防止

サーバーが提示するTLS証明書をクライアントが検証することで、「通信相手が本当にそのドメインの所有者であること」を確認する。

```mermaid
graph LR
    CA["認証局（CA）<br/>Let's Encrypt等"] -->|"証明書を発行"| Server["example.com<br/>サーバー"]
    Browser["ブラウザ"] -->|"1. 接続要求"| Server
    Server -->|"2. 証明書を提示"| Browser
    Browser -->|"3. CAの署名を検証"| CA_Root["ルート証明書<br/>（ブラウザに内蔵）"]

    style CA fill:#fff9c4
    style CA_Root fill:#fff9c4
    style Browser fill:#e1f5fe
    style Server fill:#c8e6c9
```

HTTPでは接続先が本物かどうかを検証する仕組みがないため、DNSスプーフィングやARPスプーフィングで偽のサーバーに誘導されても気づけない。

## HTTPS接続確立のコスト

HTTPSではTCPハンドシェイクの後にTLSハンドシェイクが追加される。

```mermaid
sequenceDiagram
    participant C as クライアント
    participant S as サーバー

    rect rgb(255, 243, 224)
        Note over C,S: TCP 3ウェイハンドシェイク（1.5 RTT）
        C->>S: SYN
        S->>C: SYN+ACK
        C->>S: ACK
    end

    rect rgb(227, 242, 253)
        Note over C,S: TLS 1.3 ハンドシェイク（1 RTT）
        C->>S: ClientHello + 鍵共有パラメータ
        S->>C: ServerHello + 証明書 + Finished
        C->>S: Finished
    end

    rect rgb(232, 245, 233)
        Note over C,S: 暗号化されたHTTP通信
        C->>S: GET /api/data（暗号化済み）
        S->>C: 200 OK + レスポンス（暗号化済み）
    end
```

| 接続方式 | 合計RTT | 東京↔米国西海岸（RTT≈150ms）での時間 |
|---------|---------|------|
| HTTP（TCP のみ） | 1.5 RTT | 約225ms |
| HTTPS + TLS 1.2 | 3.5 RTT（TCP 1.5 + TLS 2） | 約525ms |
| HTTPS + TLS 1.3 | 2.5 RTT（TCP 1.5 + TLS 1） | 約375ms |
| HTTPS + TLS 1.3 0-RTT | 1.5 RTT（再接続時） | 約225ms |
| HTTP/3（QUIC） | 1 RTT（TCP+TLS統合） | 約150ms |

TLS 1.3と0-RTT（再接続時にハンドシェイクを省略）により、HTTPSのパフォーマンスペナルティは大幅に縮小している。HTTP/3（QUIC）ではTCPとTLSのハンドシェイクを統合し、1 RTTで接続を確立する。

## Mixed Content — HTTPSページ内のHTTPリソース

HTTPSページ内からHTTPでリソースを読み込む「Mixed Content」は、ブラウザが段階的にブロックする。

| 種類 | 例 | ブラウザの挙動 |
|------|-----|-------------|
| **Mixed Active Content** | `<script src="http://...">`、`<iframe>`、`fetch("http://...")` | **完全にブロック**（ページの安全性を損なうため） |
| **Mixed Passive Content** | `<img src="http://...">`、`<video>` | 警告表示、一部ブラウザでブロック |

```javascript
// ❌ HTTPSページからHTTPリソースを読み込む
fetch('http://api.example.com/data')  // ブラウザがブロック

// ✅ プロトコル相対URLまたはHTTPSを使う
fetch('https://api.example.com/data')

// ✅ 同一オリジンなら相対パスが最も安全
fetch('/api/data')
```

## HSTS — HTTPに戻さない仕組み

**HSTS（HTTP Strict Transport Security）**は、一度HTTPSで接続したドメインに対して、以後のアクセスを自動的にHTTPSにアップグレードする仕組み。

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

| ディレクティブ | 意味 |
|--------------|------|
| `max-age=63072000` | 2年間（秒数）、このドメインにはHTTPSのみで接続する |
| `includeSubDomains` | すべてのサブドメインにも適用 |
| `preload` | ブラウザのプリロードリストに登録を申請する |

**なぜHSTSが必要か:**

HTTPからHTTPSへの301リダイレクトだけでは、最初のHTTPリクエストが平文で送られる。この一瞬を狙って中間者攻撃（SSLストリッピング）が可能になる。HSTSにより、ブラウザは最初からHTTPSでアクセスするため、この攻撃を防げる。

```mermaid
sequenceDiagram
    participant B as ブラウザ
    participant A as 攻撃者
    participant S as サーバー

    rect rgb(255, 235, 238)
        Note over B,S: HSTS なし — SSLストリッピング攻撃が可能
        B->>A: http://bank.example.com（平文）
        A->>S: https://bank.example.com
        S-->>A: 200 OK（暗号化）
        A-->>B: 200 OK（平文 — 改ざん可能）
    end

    rect rgb(232, 245, 233)
        Note over B,S: HSTS あり — ブラウザが自動でHTTPS化
        B->>S: https://bank.example.com（直接HTTPS）
        Note over B: ブラウザ内部で http → https に変換（307 Internal Redirect）
    end
```

## 証明書の種類と取得

| 種類                              | 検証レベル      | コスト               | 用途                       |
| ------------------------------- | ---------- | ----------------- | ------------------------ |
| **DV（Domain Validation）**       | ドメインの管理権のみ | 無料（Let's Encrypt） | 一般的なWebサイト・API           |
| **OV（Organization Validation）** | 組織の実在性     | 有料                | 企業サイト                    |
| **EV（Extended Validation）**     | 厳格な組織審査    | 高額                | 金融機関等（ただし近年はDVで十分とされる傾向） |

**Let's Encrypt** の普及により、DV証明書は無料で自動更新できるようになった。手動で証明書を管理する理由はほとんどない。

```bash
# certbot による Let's Encrypt 証明書の取得と自動更新（Nginx）
sudo certbot --nginx -d example.com -d www.example.com

# 証明書の有効期限確認
echo | openssl s_client -connect example.com:443 2>/dev/null | \
  openssl x509 -noout -dates
# notBefore=Mar 28 00:00:00 2026 GMT
# notAfter=Jun 26 00:00:00 2026 GMT

# 証明書チェーンの検証
openssl s_client -connect example.com:443 -showcerts </dev/null 2>/dev/null
```

## コード例

### Node.js — HTTPSサーバーの作成

```javascript
import { createServer } from 'node:https';
import { readFileSync } from 'node:fs';

// 本番: Let's Encrypt + リバースプロキシが一般的
// 開発: mkcert でローカル証明書を発行
const server = createServer(
  {
    key: readFileSync('./certs/key.pem'),
    cert: readFileSync('./certs/cert.pem'),
  },
  (req, res) => {
    // HTTPSであることを前提としたセキュリティヘッダ
    res.setHeader('Strict-Transport-Security', 'max-age=63072000; includeSubDomains');
    res.writeHead(200, { 'Content-Type': 'text/plain' });
    res.end('Hello over HTTPS\n');
  }
);

server.listen(443);
```

```bash
# mkcert でローカル開発用の信頼済み証明書を発行
mkcert -install                        # ルートCAをシステムの信頼ストアに追加（初回のみ）
mkcert localhost 127.0.0.1 ::1         # localhost 用の証明書を生成
# → localhost+2.pem / localhost+2-key.pem が生成される
```

### Python — HTTP→HTTPSリダイレクトの検証

```python
import requests

# リダイレクトを追わずに確認（allow_redirects=False）
response = requests.get('http://example.com', allow_redirects=False, timeout=5)
print(f"ステータス: {response.status_code}")       # 301
print(f"リダイレクト先: {response.headers.get('Location')}")  # https://example.com/

# HSTSヘッダの確認
response = requests.get('https://example.com', timeout=5)
hsts = response.headers.get('Strict-Transport-Security', 'なし')
print(f"HSTS: {hsts}")
```

### Go — HTTP→HTTPSの自動リダイレクトサーバー

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	// HTTPリクエストをHTTPSにリダイレクト
	go func() {
		redirect := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			target := "https://" + r.Host + r.URL.RequestURI()
			http.Redirect(w, r, target, http.StatusMovedPermanently)
		})
		log.Fatal(http.ListenAndServe(":80", redirect))
	}()

	// HTTPSサーバー
	mux := http.NewServeMux()
	mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Strict-Transport-Security", "max-age=63072000; includeSubDomains")
		w.Write([]byte("Hello over HTTPS"))
	})

	log.Fatal(http.ListenAndServeTLS(":443", "cert.pem", "key.pem", mux))
}
```

## よくある落とし穴

### 1. HTTPSを「導入しただけ」でHTTPを残す

HTTPSを有効にしても、HTTPポート（80）で引き続きコンテンツを配信していると、ユーザーが `http://` でアクセスした場合に平文通信になる。**HTTP→HTTPSの301リダイレクト** と **HSTS** の両方を設定する必要がある。

### 2. 開発環境でHTTPSを使わず、本番でのみ有効にする

以下の機能がHTTPSでないと動作しないため、開発環境との差異でバグを見逃す:
- `Secure`属性付きCookieが送信されない
- Service Worker が登録できない
- Geolocation APIなどの機能が制限される
- Mixed Contentの警告が出ない

`mkcert` を使えばローカル開発用の信頼済み証明書を簡単に発行できる。

### 3. 証明書の有効期限切れに気づかない

Let's Encryptの証明書は90日間有効。自動更新（certbot renew）を設定していないと、期限切れでサイト全体がアクセス不能になる。モニタリングで証明書の有効期限を監視すべき。

### 4. リバースプロキシの背後でプロトコルを判定できない

Nginx等のリバースプロキシがTLSを終端する構成では、アプリケーションサーバーにはHTTPで接続される。元のリクエストがHTTPSだったかは `X-Forwarded-Proto` ヘッダで判断する。

```
クライアント --HTTPS--> Nginx（TLS終端） --HTTP--> アプリケーション
                                                  ↑ X-Forwarded-Proto: https
```

## 関連トピック

- [[HTTP-HTTPS]] — 親トピック。HTTPプロトコルの全体像
- [[TLS-SSL]] — TLSハンドシェイクの詳細、暗号スイート、証明書チェーンの仕組み
- [[TCP-IP]] — TCPハンドシェイクとTLSハンドシェイクの重ね合わせによる接続コスト
- [[DNS]] — CAAレコードによる証明書発行制限、DNS-01チャレンジによるドメイン検証
