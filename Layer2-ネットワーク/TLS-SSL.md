---
layer: 2
topic: TLS/SSL
status: 🔴 未着手
created: 2026-03-28
prerequisites: ["[[TCP-IP]]", "[[HTTP-HTTPS]]"]
next_steps: ["[[WebSocket]]", "[[Layer6-セキュリティ/_index|Layer 6: セキュリティ]]"]
difficulty: advanced
estimated_minutes: 40
ai_collaboration: partial
---

# TLS/SSL

> **一言で言うと:** TLS（Transport Layer Security）は通信経路を暗号化し、**機密性（盗聴防止）・完全性（改ざん検知）・真正性（なりすまし防止）** の3つを保証するプロトコル。SSL（Secure Sockets Layer）はその前身であり、現在はすべてTLSに移行済み。Webにおける「信頼の連鎖（Chain of Trust）」の基盤。

## 3分で全体像

- **何を解決する技術か:** インターネット経路上の任意地点で発生しうる「盗聴・改ざん・なりすまし」をトランスポート層で一括解決する
- **代表的な使用シーン:** すべての HTTPS 通信、API 通信、メール（SMTPS/IMAPS）、VPN、mTLS でのマイクロサービス間認証、IoT デバイスとサーバーの通信
- **これだけは覚える3つ:**
    1. **TLS は通信経路を守るだけ** — サーバー上のデータの暗号化、SQLi/XSS、認可ロジックは TLS の範囲外。「HTTPS = 安全」は誤解
    2. **TLS 1.3 / 1.2 のみ許可、レガシー暗号は無効化** — TLS 1.0/1.1 や RC4・SHA-1・CBC モードは既知の脆弱性（POODLE, BEAST, CRIME）に晒される
    3. **証明書は自動更新する** — 手動更新は人的ミスで期限切れ → サービス停止に直結する。Let's Encrypt + certbot / ACM で自動化が大原則
- **AIに任せやすいか:** **一部任せられる** — Nginx/Apache の TLS 推奨設定、`openssl s_client` のデバッグコマンド、certbot の自動更新設定は AI が高品質に書ける（Mozilla SSL Configuration Generator のような確立された雛形がある）。一方「DV/OV/EV の選定」「mTLS を使うか JWT で済ますか」「0-RTT を有効化するか」「鍵の保管場所（HSM/KMS/ファイル）」は脅威モデルとビジネス要件次第で人間が判断
- **詰まったらここを読む:** [[TCP-IP]] / [[HTTP-HTTPS]] / [[Layer6-セキュリティ/_index|Layer 6: セキュリティ]]

## なぜ必要か

インターネット上の通信は、送信元から宛先までの間に多数のルーター・スイッチ・[[ISP]]を経由する。TLSがなければ、この経路上の**任意の地点で通信内容を盗聴・改ざんできる**。

具体的に何が起きるか:

- **盗聴**: パスワード・クレジットカード情報・個人データが平文で流れ、Wi-Fiの傍受やISPレベルでの監視によって読み取られる
- **改ざん**: レスポンスにマルウェアを注入される（実際に一部ISPが広告を挿入していた事例がある）
- **なりすまし**: 偽のサーバーが本物のふりをして、ユーザーの認証情報を窃取する（中間者攻撃 / Man-in-the-Middle Attack）

TLSはこれら3つの脅威を**トランスポート層で一括して解決**する。アプリケーション開発者が暗号化の詳細を意識せずとも、TLSレイヤーを挟むだけで通信が保護される。

## どの問題を解決するか

### 課題1: 盗聴防止 — 共通鍵暗号による通信の暗号化

TLSは通信データを[[暗号アルゴリズム]]で暗号化し、第三者が読めないようにする。ただし、暗号化に使う鍵をどうやって安全に共有するかが問題になる。

```mermaid
sequenceDiagram
    participant C as クライアント
    participant S as サーバー

    Note over C,S: 公開鍵暗号で安全に鍵を交換
    C->>S: ClientHello（対応する暗号スイート一覧）
    S-->>C: ServerHello（選択した暗号スイート + 証明書）
    Note over C,S: 鍵交換（ECDHE等）で共通鍵を生成
    C->>S: 暗号化された通信開始
    S-->>C: 暗号化された通信開始
    Note over C,S: 以降は共通鍵暗号（AES等）で高速に暗号化
```

**ハイブリッド暗号方式**: TLSは公開鍵暗号（非対称暗号）と共通鍵暗号（対称暗号）を組み合わせる。公開鍵暗号は計算コストが高いため鍵交換のみに使い、実データの暗号化には高速な共通鍵暗号（AES-GCM等）を使う。

### 課題2: 改ざん検知 — MAC（メッセージ認証コード）

TLSの各レコードにはMAC（Message Authentication Code）が付与される。受信側はMACを検証し、データが経路上で1ビットでも変更されていれば検知して接続を切断する。TLS 1.3ではAEAD（Authenticated Encryption with Associated Data）として暗号化と認証が一体化されている（AES-GCM、ChaCha20-Poly1305）。

### 課題3: なりすまし防止 — 証明書と信頼の連鎖

「通信先が本当に正しいサーバーか」を検証するのがデジタル証明書（X.509証明書）の役割であり、TLSの信頼モデルの中核を成す。

```mermaid
graph TD
    Root["ルートCA<br/>（OSやブラウザに事前インストール）"]
    Inter["中間CA<br/>（ルートCAが署名）"]
    Leaf["サーバー証明書<br/>（中間CAが署名）"]
    Server["example.com"]

    Root -->|署名| Inter
    Inter -->|署名| Leaf
    Leaf -->|証明| Server

    style Root fill:#c8e6c9
    style Inter fill:#fff9c4
    style Leaf fill:#bbdefb
```

**信頼の連鎖（Chain of Trust）**:
1. OSやブラウザには信頼されたルートCA（Certificate Authority）の証明書がプリインストールされている
2. ルートCAは中間CAの証明書に署名する
3. 中間CAはサーバー証明書に署名する
4. クライアントはこの連鎖を逆にたどり、ルートCAまで検証できれば「信頼できる」と判断する

**ルートCAの秘密鍵が漏洩すると、信頼の連鎖全体が崩壊する**。そのためルートCAの秘密鍵はオフラインのHSM（Hardware Security Module）に保管され、日常的な証明書発行には中間CAを使う。

### 課題4: 証明書の種類と取得

| 種類 | 検証レベル | 用途 | 費用 |
|------|-----------|------|------|
| **DV（Domain Validation）** | ドメインの所有権のみ | 一般的なWebサイト | 無料〜安価（Let's Encrypt） |
| **OV（Organization Validation）** | 組織の実在確認 | 企業サイト | 有料 |
| **EV（Extended Validation）** | 厳格な組織審査 | 金融・決済 | 高額 |

**Let's Encrypt**: 無料でDV証明書を自動発行するCA。ACMEプロトコルによる自動更新が特徴で、Webの HTTPS 普及を大きく加速させた。

### 課題5: TLSバージョンの進化

| バージョン | 年 | ハンドシェイクRTT | 特徴 | 状態 |
|-----------|-----|-------------------|------|------|
| SSL 3.0 | 1996 | 2 RTT | POODLEで致命的脆弱性 | 廃止 |
| TLS 1.0 | 1999 | 2 RTT | BEAST攻撃で脆弱 | 廃止 |
| TLS 1.1 | 2006 | 2 RTT | IV の改善 | 廃止 |
| TLS 1.2 | 2008 | 2 RTT | 長期間の標準。AEAD対応 | 現役（縮小中） |
| **TLS 1.3** | **2018** | **1 RTT（0-RTTも可能）** | **暗号スイート簡素化、ハンドシェイク高速化、前方秘匿性必須** | **推奨** |

TLS 1.3の主な改善:

- **ハンドシェイクの高速化**: 2 RTT → 1 RTT。0-RTT（Early Data）では再接続時にハンドシェイク完了前にデータを送信できる
- **前方秘匿性（Forward Secrecy）の必須化**: 鍵交換にECDHE（楕円曲線ディフィー・ヘルマン鍵共有）のみを使用。サーバーの秘密鍵が将来漏洩しても、過去の通信は復号できない
- **レガシー暗号の廃止**: RC4、SHA-1、CBC モードなど脆弱な暗号スイートを排除
- **ハンドシェイクの暗号化**: ServerHello以降が暗号化され、通信内容だけでなくハンドシェイクのメタデータも保護

```mermaid
sequenceDiagram
    participant C as クライアント
    participant S as サーバー

    Note over C,S: TLS 1.3 — 1-RTT ハンドシェイク
    C->>S: ClientHello + 鍵共有パラメータ + 対応暗号スイート
    S-->>C: ServerHello + 鍵共有パラメータ + 証明書 + Finished
    Note over C: 証明書の検証 + 共通鍵の導出
    C->>S: Finished
    Note over C,S: 暗号化通信開始（1 RTT で完了）
```

## 他の仕組みとどう関係するか

- **下位レイヤーとの関係:**
  - [[TCP-IP]] — TLSはTCP接続の確立後に動作する。TCPの3ウェイハンドシェイク（1 RTT）に加えてTLSハンドシェイク（TLS 1.3で1 RTT）が必要。HTTP/3ではQUIC内にTLS 1.3が統合され、TCPハンドシェイクとTLSハンドシェイクが同時に行われる（0-RTT接続も可能）
  - [[プロセスとスレッド]] — TLSの暗号化/復号はCPU負荷が高い。大量のHTTPS接続を捌くサーバーではAES-NIなどのハードウェアアクセラレーションが重要

- **同レイヤーとの関係:**
  - [[HTTP-HTTPS]] — HTTPSはHTTP + TLS。TLSハンドシェイクのコストがHTTPS接続のレイテンシに直結するため、TLS 1.3への移行やセッション再開（Session Resumption）が重要。[[HTTPとHTTPSの違い]]はTLSレイヤーの有無に集約される
  - [[DNS]] — DNSのCAAレコード（Certificate Authority Authorization）は、ドメインに対してどのCAが証明書を発行できるかを制限する。また、DNS over HTTPS（DoH）やDNS over TLS（DoT）はDNSクエリ自体をTLSで暗号化する
  - [[WebSocket]] — WebSocket over TLS（wss://）ではTLSハンドシェイク後にWebSocketハンドシェイクが行われる

- **上位レイヤーとの関係:**
  - [[認証と認可]] — TLSクライアント証明書を使ったmTLS（mutual TLS）は、APIやマイクロサービス間の認証に使われる。JWT等のトークンベース認証はTLSの上に構築される
  - [[CORS]] — CORSはHTTPS環境でのクロスオリジン通信を制御する。Mixed Content（HTTPSページからHTTPリソースの読み込み）はブラウザがブロックする
  - [[CDN]] — CDNエッジサーバーがTLS終端（TLS Termination）を行い、オリジンサーバーへの通信負荷を軽減する

## 誤解されやすいポイント

### 1. 「SSLとTLSは同じもの」

SSLはNetscapeが開発した初期の暗号化プロトコルであり、SSL 3.0まで存在した。TLSはSSLの後継としてIETFが標準化したプロトコルで、TLS 1.0はSSL 3.1に相当する。**現在「SSL」と呼ばれているものは実際にはTLS**であり、SSL 3.0は2015年にRFC 7568で正式に廃止された。「SSL証明書」という呼称も慣習的に残っているだけで、実体はTLS用の証明書。

### 2. 「TLSは通信の全てを守る」

TLSが保護するのは**通信経路上のデータ**のみ。以下は守れない:
- **通信のメタデータ**: 接続先のIPアドレスやSNI（Server Name Indication）でのホスト名は平文で送信される（TLS 1.3 + ECH で改善中）
- **エンドポイントでのデータ**: サーバーに到達した後のデータの保存・処理は別問題。TLSで暗号化されていてもサーバー側のログに平文で記録される可能性がある
- **アプリケーション層の脆弱性**: SQLインジェクション、XSSなどはTLSでは防げない

### 3. 「証明書があれば安全なサイト」

DV証明書はドメインの所有権のみを証明する。**サイトの運営者が信頼できるかどうか**は証明しない。フィッシングサイトもDV証明書（Let's Encrypt等）を取得できる。ブラウザの鍵アイコンは「通信が暗号化されている」ことを示すだけで、「サイトが安全」を意味しない。

### 4. 「自己署名証明書は開発環境なら問題ない」

自己署名証明書は信頼の連鎖に属さないため、ブラウザが警告を出す。開発者がこの警告を無視する習慣がつくと、本番環境での証明書エラーも見逃すリスクがある。開発環境では**mkcert**を使ってローカルCAを作成し、そのCAをシステムの信頼ストアに登録する方が安全。

### 5. 「TLS 1.3の0-RTTは常に使うべき」

0-RTT（Early Data）は再接続時のレイテンシを削減するが、**リプレイ攻撃に脆弱**。攻撃者が0-RTTデータを再送すると、サーバーが同じリクエストを再度処理してしまう可能性がある。そのため0-RTTではべき等な操作（GET）のみを許可し、状態を変更する操作（POST）には使わないのが原則。

## 設計のベストプラクティス

### TLS設定

```
✅ 推奨: TLS 1.3を優先、TLS 1.2を最低ラインとする
   - TLS 1.1以下は無効化する（PCIコンプライアンス要件でもある）
   - 暗号スイートはAEADのみ許可（AES-256-GCM, ChaCha20-Poly1305）
   - 鍵交換はECDHE（前方秘匿性を確保）
   - RSA鍵交換は無効化（前方秘匿性がない）

❌ アンチパターン: 互換性のためにTLS 1.0/1.1やレガシー暗号を有効にする
   - 既知の脆弱性（BEAST, POODLE, CRIME）に晒される
   - ダウングレード攻撃の対象になる
```

### 証明書管理

```
✅ 推奨: 証明書の自動更新を設定する
   - Let's Encrypt + certbot/acme.sh で自動更新
   - 証明書の有効期限を監視する（期限切れはサービス停止に直結）
   - CAA DNSレコードで許可するCAを制限する
   - OCSP Staplingを有効にして証明書の失効チェックを高速化

❌ アンチパターン: 証明書を手動で更新する
   - 人的ミスで期限切れが発生し、サービスが停止する
   - 有名企業でも証明書の期限切れ事故は頻繁に発生している
```

### HSTS（HTTP Strict Transport Security）

```
✅ 推奨: HSTSヘッダを設定し、HTTPS接続を強制する
   Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
   - ブラウザがHTTPリクエストを送信する前にHTTPSに変換する
   - 初回アクセス時のHTTP→HTTPSリダイレクトの隙を突く攻撃を防ぐ
   - preloadリストに登録すれば初回アクセスからHTTPS強制

❌ アンチパターン: HTTPとHTTPSの両方でサービスを提供し続ける
   - SSLストリッピング攻撃の対象になる
```

## AIエージェントとの協働

> このトピックでAIコーディングエージェントと協働するための観点。「AIに何をどこまで任せ、人間は何を判断するか」を整理する。

### AIに任せられる部分 / 人間が判断すべき部分

| タスク種類 | 任せ方（実装/レビュー） | 人間の関与 |
|---|---|---|
| Nginx/Apache の TLS 推奨設定（バージョン制限、暗号スイート、HSTS、OCSP Stapling） | サーバー種別と互換性要件を渡して任せる（Mozilla SSL Config Generator 相当） | 互換性要件（古いブラウザサポート可否）の確定 |
| Let's Encrypt + certbot / ACM の自動更新設定 | 環境（Nginx, ALB, ECS 等）を渡して任せる | 更新失敗時の通知先、ロールバック手順 |
| `openssl s_client` / `curl --tls1_3` でのデバッグコマンド組み立て | 調査目的（証明書チェーン確認、対応暗号確認）を渡して任せる | 結果の解釈、本番環境での実行可否 |
| **TLS バージョン・暗号スイートのレビュー** | `/review-ai-code` でレビューさせる | 古いクライアントとの互換性とのトレードオフ判断 |
| HSTS / CSP 等のセキュリティヘッダ初版 | アプリの構成を渡して任せる | preload リスト登録の判断、HSTS の段階導入計画 |
| mTLS の構成（証明書発行・配布・検証） | アーキテクチャを渡して任せる | 「mTLS を使うか JWT で済ますか」の選定、CA の運用体制 |
| 0-RTT 有効化の判断 | べき等な操作のみ許可する設定を任せる | そもそも 0-RTT を有効化するか（リプレイ攻撃許容範囲） |
| 鍵の保管場所（KMS / HSM / ファイル / Vault） | 候補のトレードオフを AI に出させる | コンプライアンス要件、運用コスト、災害復旧プランで決定 |

### AI生成コードのレビュー観点（このトピック固有）

AI生成物を受け取ったとき、最低限ここを見る。

1. **証明書検証を無効化していないか:** `verify=False`、`rejectUnauthorized: false`、`InsecureSkipVerify: true` がコードに混入していないか。これは TLS の存在意義を消す致命的なバグ。「動かないから」で AI が安易に追加することがある
2. **TLS バージョンと暗号スイートが現代的か:** TLS 1.2/1.3 のみ許可されているか、AEAD 暗号（AES-GCM, ChaCha20-Poly1305）が使われているか、RC4/3DES/CBC モード/SHA-1 が無効化されているか
3. **秘密鍵がソース/イメージに混入していないか:** `*.pem`, `*.key` がリポジトリにコミットされていないか、Dockerfile の `COPY` で含めていないか、環境変数 / シークレットマネージャー / ボリュームマウントで分離されているか

### 効くプロンプトの型

このトピックに関する実装をAIに依頼するとき、コンテキストとして渡すべき情報・制約・成功基準のテンプレ。

```
# 前提
- サーバー: Nginx 1.27 / Apache 2.4 / Caddy / ALB / CloudFront
- 証明書: Let's Encrypt / AWS ACM / 内部 CA / 自社 CA
- 互換性要件: モダンブラウザのみ / 古い Android / IoT デバイス対応必要
- コンプライアンス: PCI DSS / HIPAA / 社内ポリシー
- HTTP バージョン: HTTP/1.1 / HTTP/2 / HTTP/3
- 鍵の保管: KMS / HSM / ファイル / Vault

# やってほしいこと
- 〜の TLS 設定 / 自動更新スクリプト / mTLS 構成を作成

# 守ってほしい制約（このトピック固有）
- TLS 1.2 以上のみ許可、AEAD 暗号スイートのみ
- 鍵交換は ECDHE（前方秘匿性）。RSA 鍵交換は無効化
- HSTS ヘッダ（max-age=63072000; includeSubDomains; preload）
- OCSP Stapling 有効
- CAA DNS レコードで許可 CA を制限
- 自動更新（cron / systemd timer）と更新失敗時のアラート
- 証明書と秘密鍵はソース / イメージに含めない

# 完了の判断基準
- SSL Labs（ssllabs.com/ssltest）で A+ 評価
- TLS 1.0/1.1 で接続できないことを openssl s_client で確認
- 証明書の自動更新がドライランで成功
- HSTS preload 登録条件を満たす
```

### AI実装のアンチパターン

LLM生成コードで頻出する過剰設計・誤用パターン。レビュー時の照合表として使う。

| アンチパターン | なぜ問題か | レビュー観点 / 対策 |
|---|---|---|
| **TLS 証明書の検証を無効化（`verify=False`, `rejectUnauthorized: false`, `InsecureSkipVerify: true`）** | 中間者攻撃（MITM）が可能になり、TLS の存在意義が消える。「動かないから」と AI が安易に追加しがち。本番に残ると致命的 | 正しい CA 証明書を設定。社内 CA なら CA 証明書を信頼ストアに追加。開発環境では mkcert |
| **古い TLS バージョンやレガシー暗号スイートを許可** | POODLE（SSL 3.0）、BEAST（TLS 1.0）、CRIME 等の既知攻撃に脆弱。PCI DSS 等のコンプライアンス違反 | TLS 1.2 以上のみ。AEAD 暗号スイート（AES-GCM, ChaCha20-Poly1305）のみ |
| **秘密鍵をソースコード / Docker イメージに含める** | `git log` / `docker history` で全履歴に残り、削除しても完全には消えない。鍵の漏洩は全通信の危殆化（過去のキャプチャまで復号可能になる） | 環境変数 / シークレットマネージャー（AWS Secrets Manager, Vault）/ ボリュームマウント。ローテーション手順も用意 |
| **HTTP→HTTPS リダイレクトだけで HSTS なし** | 初回アクセスの HTTP→HTTPS リダイレクト中に SSL Stripping 攻撃が可能。中間者が HTTP を維持してパスワードを盗める | HSTS ヘッダ + preload リスト登録。HTTPS でしかアクセスできないと初回からブラウザに記憶させる |
| **0-RTT で POST リクエストを受け付ける** | TLS 1.3 の 0-RTT（Early Data）はリプレイ攻撃に脆弱。攻撃者が 0-RTT データを再送すると、サーバーが同じ POST を再処理して二重決済等が発生 | 0-RTT は冪等な操作（GET）のみ許可。サーバー側でのリプレイ検知（nonce / 時刻チェック）も併用 |
| **手動で証明書を更新** | 更新忘れで期限切れ → サービス全停止。有名サービスでも頻繁に発生する | certbot / ACM / cert-manager で自動更新。期限監視を CloudWatch / Datadog に組む |
| **自己署名証明書を本番で使い回す** | 信頼の連鎖に属さないためクライアント側で警告。警告を無視する習慣がつくと本番事故も見逃す | 開発環境は mkcert（ローカル CA を作って信頼ストアに登録）、本番は Let's Encrypt / ACM |
| **`Allow Insecure` のような環境変数で本番でも検証バイパス可能にする** | 設定ミスで本番が無防備に。`if (env === 'dev')` のような条件をくぐり抜ける事故が起きる | 検証バイパスはコード上から除去。CA 証明書の差し替えで対応する |

→ レイヤー横断のアンチパターン索引: [[_anti-patterns/_index|AIアンチパターン索引]]

## 具体例

### Nginx — TLS 1.3 推奨設定

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name example.com;

    # 証明書と秘密鍵
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # TLSバージョン: 1.2以上のみ
    ssl_protocols TLSv1.2 TLSv1.3;

    # 暗号スイート: TLS 1.2はAEADのみ、TLS 1.3はデフォルトで安全
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;  # TLS 1.3ではクライアント側の選択を尊重

    # ECDHE鍵交換の曲線
    ssl_ecdh_curve X25519:secp384r1;

    # セッション再開（TLS 1.2）
    ssl_session_cache shared:TLS:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;  # 前方秘匿性を完全に保つ場合はoff

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}
```

### Node.js — TLSサーバーの基本設定

```javascript
import { createServer } from 'https';
import { readFileSync } from 'fs';

const server = createServer({
  key: readFileSync('/path/to/privkey.pem'),
  cert: readFileSync('/path/to/fullchain.pem'),

  // TLS 1.2以上のみ許可
  minVersion: 'TLSv1.2',

  // 暗号スイートの制限（TLS 1.2用）
  ciphers: [
    'ECDHE-ECDSA-AES256-GCM-SHA384',
    'ECDHE-RSA-AES256-GCM-SHA384',
    'ECDHE-ECDSA-CHACHA20-POLY1305',
    'ECDHE-RSA-CHACHA20-POLY1305',
  ].join(':'),
}, (req, res) => {
  // HSTS ヘッダ
  res.setHeader('Strict-Transport-Security', 'max-age=63072000; includeSubDomains; preload');
  res.end('Hello, TLS!');
});

server.listen(443);
```

### Python — 証明書の検証とTLS設定

```python
import ssl
import urllib.request

# 安全なTLSコンテキストの作成
ctx = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
ctx.minimum_version = ssl.TLSVersion.TLSv1_2

# システムのCA証明書のデフォルトパスを設定する
ctx.set_default_verify_paths()

# 証明書の検証を有効にする（デフォルトで有効だが明示的に）
ctx.check_hostname = True
ctx.verify_mode = ssl.CERT_REQUIRED

response = urllib.request.urlopen("https://example.com", context=ctx)
print(response.read().decode())
```

### openssl — 証明書の確認とデバッグ

```bash
# サーバーの証明書チェーンを確認
openssl s_client -connect example.com:443 -servername example.com

# 証明書の詳細を表示
openssl s_client -connect example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -text

# 証明書の有効期限を確認
openssl s_client -connect example.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -dates

# 対応TLSバージョンの確認
openssl s_client -connect example.com:443 -tls1_3

# 証明書チェーンの検証
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt cert.pem
```

### certbot — Let's Encrypt 証明書の自動取得・更新

```bash
# 証明書の取得（Nginx用）
sudo certbot --nginx -d example.com -d www.example.com

# 証明書の更新テスト
sudo certbot renew --dry-run

# 自動更新はsystemd timerまたはcronで設定
# certbotインストール時に自動設定されることが多い
systemctl list-timers | grep certbot
```

## 参考リソース

- **書籍**: 『プロフェッショナルSSL/TLS』（Ivan Ristic） — TLSの実装・運用・セキュリティを網羅した決定版
- **書籍**: 『暗号技術入門 第3版』（結城浩） — 公開鍵暗号・証明書・TLSの基礎を分かりやすく解説
- **Web**: Mozilla SSL Configuration Generator — https://ssl-config.mozilla.org/ — Nginx/Apache/各種サーバーの推奨TLS設定を自動生成
- **Web**: SSL Labs Server Test — https://www.ssllabs.com/ssltest/ — サーバーのTLS設定を評価・診断するツール
- **RFC 8446**: TLS 1.3 — https://datatracker.ietf.org/doc/html/rfc8446
- **Web**: Let's Encrypt Documentation — https://letsencrypt.org/docs/
- **Web**: High Performance Browser Networking — TLS章（Ilya Grigorik） — TLSのパフォーマンス影響の解説

## 理解度セルフチェック

> 答えられなければ本文に戻る。答えはこのファイル内に必ずある。

1. 「HTTPS にすれば安全」という説明が不正確である理由を、TLS が守る範囲と守らない範囲の観点で30秒で説明できるか
2. 前方秘匿性（Forward Secrecy）とは何か、なぜ TLS 1.3 で必須化されたのかを説明せよ
3. 次のAI生成 Python コードはこのトピックの観点で何が問題か。修正方針を述べよ:

```python
import requests
import os

# 内部APIから機密データを取得する関数
def fetch_internal_data(item_id):
    cert_pem = """
-----BEGIN CERTIFICATE-----
MIIDazCCAlOgAwIBAgI... (省略)
-----END CERTIFICATE-----
-----BEGIN PRIVATE KEY-----
MIIEvgIBADANBgkqhki... (省略)
-----END PRIVATE KEY-----
"""
    # SSL証明書のエラーが出ることがあるので無効化
    response = requests.get(
        f"https://internal-api.example.com/items/{item_id}",
        verify=False,
        cert=cert_pem,
        timeout=10,
    )
    return response.json()
```

> [!info] 用語ミニ辞典（解答を読む前に）
> - **TLS / SSL** — 通信経路の暗号化プロトコル。SSL 3.0 までは Netscape の SSL、TLS 1.0 以降は IETF 標準。現在は TLS 1.2/1.3 が現役で、SSL 3.0 と TLS 1.0/1.1 は廃止
> - **AEAD（Authenticated Encryption with Associated Data）** — 暗号化と認証（改ざん検知）を一体化した暗号方式。AES-GCM、ChaCha20-Poly1305 が代表。CBC モード + HMAC のような分離型より安全
> - **前方秘匿性（Forward Secrecy）** — サーバーの長期秘密鍵が将来漏洩しても、過去の通信は復号できない性質。鍵交換ごとに使い捨ての鍵（ephemeral key）を生成することで実現。ECDHE が代表
> - **MITM（Man-in-the-Middle / 中間者攻撃）** — 攻撃者が通信経路の途中に入り、両端のふりをして通信内容を盗聴・改ざんする攻撃
> - **HSTS（HTTP Strict Transport Security）** — 「以後このドメインには HTTPS でしか接続するな」とブラウザに記憶させるヘッダ。SSL Stripping 攻撃を防ぐ
> - **OCSP Stapling** — サーバーが事前に取得した「自分の証明書の有効性証明（OCSP レスポンス）」をハンドシェイク中に提示する仕組み。クライアントから CA への問い合わせを不要にしてプライバシーと速度を向上
> - **mTLS（mutual TLS）** — クライアントもサーバーも証明書で互いを認証する TLS。マイクロサービス間や企業 VPN で使う

> [!note]- 解答の指針
> **問1: 「HTTPS = 安全」が不正確な理由**
>
> TLS が保証するのは **通信経路上の3つ（機密性・完全性・真正性）** だけで、それ以外は守らない。
>
> **TLS が守る範囲:**
>
> - 経路上のルーター・ISP・公衆 Wi-Fi の傍受からの保護（盗聴防止）
> - 経路上での内容書き換えからの保護（改ざん検知）
> - 接続先サーバーが本物かの検証（なりすまし防止）
>
> **TLS が守らない範囲:**
>
> - **接続先メタデータ:** 接続先 IP アドレスと SNI でのホスト名は平文で送信される（TLS 1.3 + ECH で改善中だが普及途上）
> - **エンドポイントでのデータ:** サーバー到達後のログ・DB の暗号化、メモリ上のデータは別問題
> - **アプリ層の脆弱性:** SQL インジェクション、XSS、CSRF、認可不備、IDOR 等は TLS で防げない
> - **サイトの信頼性自体:** DV 証明書はドメインの所有権だけ証明する。フィッシングサイトも Let's Encrypt で証明書を取得できる
>
> ブラウザの鍵アイコンは「通信が暗号化されている」だけを意味し、「サイトが信頼できる」を意味しない。HTTPS は **必要条件であって十分条件ではない**。アプリ層のセキュリティ（入力バリデーション、認可、CSRF対策）は別途必要。
>
> **問2: 前方秘匿性とは / なぜ TLS 1.3 で必須化されたか**
>
> 前方秘匿性（Forward Secrecy）とは、「**今、サーバーの秘密鍵が漏洩しても、過去に行われた通信を遡って復号できない**」という性質。
>
> 旧来の RSA 鍵交換では、クライアントがサーバーの公開鍵で暗号化した「セッション鍵」を送る。これを攻撃者が記録しておくと、将来サーバーの秘密鍵が漏洩した時点で過去の全通信を復号できる。実際に Snowden 事件で「政府機関が大量の TLS 通信を保存し、後日復号する」運用が行われていることが明らかになった。
>
> ECDHE（楕円曲線ディフィー・ヘルマン鍵共有 / ephemeral）では、鍵交換ごとに使い捨ての一時鍵を生成し、通信終了後にその鍵を破棄する。長期秘密鍵が後から漏洩しても、その鍵では過去の通信を復号できない。
>
> TLS 1.3 では:
>
> - RSA 鍵交換を完全に削除（前方秘匿性のない方式を排除）
> - ECDHE / DHE のみが鍵交換手段
> - 暗号スイートの選択肢を5つに絞り、すべて AEAD かつ前方秘匿性あり
>
> 結果、TLS 1.3 を選んだ時点で前方秘匿性が自動的に保証される。これが「TLS 1.3 移行の最大のセキュリティ改善」の一つ。
>
> **問3: AI生成 Python コードの問題点**
>
> このコードは、TLS のアンチパターンを3つ重ねた典型例。
>
> **(a) 秘密鍵をソースにハードコード**
>
> `cert_pem` に証明書とともに `PRIVATE KEY` がベタ書きされている。これは:
>
> - `git log` に永久に残る（過去のコミットからも消えない）
> - リポジトリにアクセスできる全員に鍵が露出
> - 鍵のローテーションがコードのデプロイと連動してしまう
>
> 修正: 環境変数 / シークレットマネージャー（AWS Secrets Manager, HashiCorp Vault）/ ファイルパスで分離。
>
> **(b) `verify=False` で証明書検証を無効化**
>
> 「SSL 証明書のエラーが出ることがあるので」という典型的な AI のコメント。これは中間者攻撃を全面的に許可する設定で、機密データを扱うコードに混入したら致命的。
>
> 「エラーが出る」のは多くの場合:
>
> - 内部 CA の証明書がシステムの信頼ストアに登録されていない → CA 証明書を `verify="/path/to/ca.pem"` で渡す
> - 証明書の有効期限切れ → 更新する
> - SNI の不一致 → サーバー側の設定を直す
>
> どれも `verify=False` で隠蔽せず、根本原因を直す。
>
> **(c) `cert` パラメータに鍵を文字列で渡している**
>
> `requests.get(cert=...)` はファイルパスを期待する（mTLS 用のクライアント証明書）。文字列で渡すと動かないか、想定外の挙動になる。
>
> **修正版（最小構成）:**
>
> ```python
> import os
> import requests
>
> # モジュールレベルでセッションを構築（コネクション再利用 + 設定の一元化）
> session = requests.Session()
>
> # mTLS 用クライアント証明書とキーはファイルパスで指定
> session.cert = (
>     os.environ['CLIENT_CERT_PATH'],   # /run/secrets/client.crt
>     os.environ['CLIENT_KEY_PATH'],    # /run/secrets/client.key
> )
>
> # 内部 CA を信頼ストアに追加（システムの信頼ストアと併用も可）
> session.verify = os.environ['INTERNAL_CA_PATH']  # /etc/ssl/certs/internal-ca.pem
>
> def fetch_internal_data(item_id):
>     response = session.get(
>         f"https://internal-api.example.com/items/{item_id}",
>         timeout=(3, 10),  # (接続, 読み取り)
>     )
>     response.raise_for_status()
>     return response.json()
> ```
>
> 鍵は Docker volume / Kubernetes Secret / Secrets Manager 経由で `/run/secrets/` 等にマウントする。`verify=False` は本番コードから完全に除去する（コードレビューで弾く）。

## 学習メモ

- mTLS（mutual TLS）によるマイクロサービス間認証は[[認証と認可]]で深掘り候補
- Certificate Transparency（CT）ログは証明書の不正発行を検知する仕組みとして重要
- ECH（Encrypted Client Hello）はSNIを暗号化してプライバシーを向上させるTLS 1.3拡張 — 標準化が進行中
- QUIC内のTLS 1.3統合について、[[HTTP-HTTPS]]のHTTP/3セクションと合わせて理解する
