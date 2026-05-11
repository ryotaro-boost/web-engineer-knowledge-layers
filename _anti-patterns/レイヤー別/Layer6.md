# Layer 6: セキュリティ — AI実装アンチパターン集

> Layer 6（[[XSS]] / [[CSRF]] / [[SQLインジェクション]] / [[CORS]] / [[DoS攻撃とDDoS攻撃]] / [[サプライチェーンセキュリティ]] / [[最小権限の原則]]）の改修で集約された、AIコーディングエージェントが頻発させるアンチパターンの索引。

## このレイヤーで頻出する「AIの癖」

Layer 6 は **「動くこと」と「安全であること」のトレードオフを AI が常に動作優先側へ倒す** レイヤーで、`*` の全開放、文字列結合での SQL/HTML 構築、検証なしの除外設定 が頻繁に混入する。Layer 6 アンチパターンには次の 7 つの根がある。

- **「データとコードの混同」誤認** — `${userInput}` でテンプレートリテラル / f-string で SQL や HTML を組み立てる。プリペアドステートメント（`$1`, `?`, `%s`）やテンプレートエンジンの自動エスケープに乗らず、**インジェクション系（XSS / SQLi）の根本原因をそのまま温存**する
- **「全開放のデフォルト値」誤認** — `Allow-Origin: *`、`GRANT ALL PRIVILEGES`、`Action: "*"`、`USER root`、`Allow-Methods: *` を AI は「動かすため」に生成する。allowlist 設計が原則だが、**AI は denylist / 全許可スタイルに自然に流れる**
- **「除外ルートの過剰適用」誤認** — Webhook 1 本のために CSRF 保護を全 API から外す、開発時の便宜で全 API のレート制限を外す、テスト用のバイパスが本番に漏れる。**「特定ケースの除外を全体に波及させる」**典型
- **「保護の意味の取り違え」誤認** — CORS で CSRF を防いだつもり、`HttpOnly` Cookie で XSS の被害なしと思う、`SameSite=Lax` だけで CSRF 完全防御と判断、`npm audit` クリーンで依存安全と判断。**それぞれが守る範囲を取り違える**
- **「侵害前提の発想欠落」誤認** — `GRANT ALL` のまま本番、`USER root` のまま、Admin ロール 1 つで全員、長命 API キー、Dockerfile に `.env` をそのまま COPY。**Assume Breach（侵害は起きる）**の発想に立てておらず、爆発半径が無制限
- **「動かすための便宜」漏出** — 開発時の `Allow-Origin: *`、テスト時の CSRF disable、デバッグ用エラースタック露出、認証ミドルウェアの一時バイパスが**本番ビルドに残る**。AI は環境分岐を書くが分岐の片側を消し忘れる
- **「過剰防御 = UX 破壊」誤認** — 全エンドポイントに CAPTCHA、過度なバリデーション、攻撃面の少ない箇所への WAF 強化。**正規ユーザーを締め出すが攻撃者は迂回**する非対称な防御

これらは多くがレイヤー横断のパターン（**過剰なフォールバック / 防御的すぎるエラーハンドリング / 不要な抽象化 / 既存ユーティリティの再発明**）に紐付く。Layer 6 は特に **「動作確認の便宜」と「本番デフォルト」の境界** を厳密に管理する必要がある。

## トピック別アンチパターン索引

### [[XSS]]

| アンチパターン | レビュー観点 |
|---|---|
| `innerHTML` / `dangerouslySetInnerHTML` / `v-html` でユーザー入力を描画 | `textContent` / フレームワークの値補間に置き換え、必要なら DOMPurify |
| XSS 対策をフロントのみで実施 | 攻撃者は API 直接叩きでバイパス、サーバー側で出力時エスケープ |
| テンプレートリテラルで HTML 構築 | テンプレートエンジン（EJS / Jinja2 / html/template）か DOM API |
| CSP だけに依存してエスケープ省略 | エスケープが主防御、CSP は多層防御の追加層 |
| 入力時に HTML エスケープして DB 保存 | 出力コンテキストで二重エスケープ事故、生データを保存し出力時エスケープ |
| 正規表現で自作 HTML サニタイザー | `<svg onload>` など見逃す、DOMPurify / sanitize-html を使う |
| `href={url}` にユーザー入力直接渡し | `javascript:` スキーム XSS、`https?:` allowlist で検証 |
| `JSON.stringify(data)` を `<script>` 直接埋め込み | `</script>` 含むデータで script タグ閉じ、専用エスケープか `<script type="application/json">` |
| CSP に `'unsafe-inline'` / `'unsafe-eval'` を許可 | nonce / hash ベースで設計し、`report-uri` で違反検知 |

### [[CSRF]]

| アンチパターン | レビュー観点 |
|---|---|
| API フレームワーク（FastAPI 等）で CSRF 保護なし + Cookie 認証 | Cookie 認証時は明示的に CSRF ミドルウェア導入 |
| CSRF トークン検証から全 API ルートを除外 | 除外はルート単位で明示、Webhook には HMAC 署名 |
| `SameSite=None` を考えずに設定 | 必要なケースのみ `None`、CSRF トークンで補完 |
| テスト環境で CSRF 無効化 → 本番漏れ | 環境変数分岐ではなくテストクライアントでトークン取得 |
| GET リクエストで状態変更（DB 書き込み・課金） | POST/PUT/DELETE のみ、GET は冪等・副作用なし |
| 全ユーザー共通の固定 CSRF トークン | セッション / ユーザー単位で発行 |
| CORS の `Allow-Origin` 制限を CSRF 対策とみなす | CORS はレスポンス読み取り制限、CSRF トークン + SameSite で対策 |
| Bearer Token を HttpOnly Cookie に保存しつつ CSRF 対策なし | Cookie 保存にする時点で Cookie 認証同等の CSRF 対策 |

### [[SQLインジェクション]]

| アンチパターン | レビュー観点 |
|---|---|
| テンプレートリテラル / f-string で SQL 構築 | プレースホルダ（`$1`, `?`, `%s`）を使用 |
| `$queryRawUnsafe` / `DB::raw` / `text(f"...")` の使用 | 通常 ORM API か バインドパラメータ付き raw クエリ |
| 動的テーブル名・カラム名のユーザー入力受け入れ | プレースホルダで渡せないので allowlist 検証 |
| IN 句でスプレッド構文 (`IN (${ids.join(',')})`) | `ANY($1::int[])` や ORM の `in` 演算子 |
| LIKE 句でユーザー入力にワイルドカード結合 | プレースホルダで値を渡し、値側で `%` を付加 |
| 数値型パラメータの型チェック省略 | バリデーション + プレースホルダの多層防御 |
| DB エラー詳細をクライアントに返す | 本番では汎用メッセージ、詳細はサーバーログ |
| 認証バイパス可能な「動作確認用」ロジック残置 | コードレビュー + git grep で削除確認 |
| ストアドプロシージャ内での動的 SQL 文字列結合 | プロシージャ内でもパラメータ化 |

### [[CORS]]

| アンチパターン | レビュー観点 |
|---|---|
| `Allow-Origin: *` をデフォルトで生成 | 環境変数から具体的オリジンを取得、Credentials と併用不可 |
| `Allow-Methods: *` / `Allow-Headers: *` | 実際に使うメソッド・ヘッダのみ列挙 |
| CORS ミドルウェアと手動 OPTIONS ハンドラの併設 | ミドルウェア一元管理 |
| `Max-Age` を設定しない | `Max-Age: 7200`（Chromium 上限）でキャッシュ |
| `Origin` を `String.includes` で部分一致検証 | 完全一致 (`===`) または `URL` パースで host 確認 |
| 動的オリジンを返しているのに `Vary: Origin` 抜け | CDN キャッシュ汚染、必ず `Vary: Origin` |
| CORS で CSRF 対策をしたつもり | CORS はリクエスト送信を止めない、CSRF トークン + SameSite |
| `Allow-Credentials: true` を無条件に設定 | Cookie 認証時のみ true、Bearer 構成では不要 |
| ローカル開発の `*` 設定を本番へ持ち込む | 環境変数で完全分離、CI で本番設定チェック |

### [[DoS攻撃とDDoS攻撃]]

| アンチパターン | レビュー観点 |
|---|---|
| IP ブロックリストをハードコード | CGNAT / IPv6 で誤爆、ASN 評価や Bot 検知サービスに委譲 |
| 全エンドポイントに CAPTCHA | UX 破壊、リスクの高いエンドポイントに限定 + Invisible CAPTCHA |
| WAF ルールを LLM が網羅的に生成 | log only から開始、統計後に enforce |
| 「DoS 対策 = レート制限」と短絡 | 帯域型 DoS / Slowloris に無力、CDN/WAF/タイムアウト併用 |
| エラー応答に詳細スタックトレース | 情報漏洩 + コスト増、本番は汎用化 |
| クライアント retry を exponential backoff なしで生成 | jitter + 最大試行回数 + `Retry-After` 尊重 |
| `/health` で全テーブル `SELECT COUNT(*)` | 攻撃時に DB が死ぬ、軽量化 + 詳細は別エンドポイント |
| アプリサーバーで全 DDoS を捌こうとする | CDN / 上流で吸収する前提アーキテクチャ |
| 攻撃検知時に手動で設定変更する運用 | 自動緩和（Under Attack Mode）+ Runbook |
| ヘルスチェックを高負荷時に止める | LB が「全サーバー死亡」判定、軽量・優先処理に |

### [[サプライチェーンセキュリティ]]

| アンチパターン | レビュー観点 |
|---|---|
| `npm install パッケージ名` でバージョン未指定 | `@バージョン` 明示、`--save-exact` で完全固定 |
| `npx 未知のCLI` をバージョン未指定で実行 | `devDependencies` 追加、または `npx foo@version` |
| lockfile を `.gitignore` | lockfile は必ずコミット |
| `postinstall` の無批判実行 | `--ignore-scripts` + 必要なものだけ明示許可 |
| 依存の過剰追加 | 標準ライブラリ / 既存依存で代替可能か検討 |
| Docker で `npm install` を `npm ci` 不使用 | `npm ci --ignore-scripts` をマルチステージで |
| `.npmignore` の denylist 方式 | `package.json` の `files` allowlist |
| `npm audit` だけで安全判断 | Socket.dev / Snyk の行動分析を併用 |
| メンテナの 2FA 未有効化（公開側） | `npm profile enable-2fa auth-and-writes`、Granular Token + IP 制限 |
| AI 生成の「存在しないパッケージ名」をそのまま install | パッケージ幻覚悪用のスクワッティング、`npm view` で実在確認 |

### [[最小権限の原則]]

| アンチパターン | レビュー観点 |
|---|---|
| `GRANT ALL PRIVILEGES` を DB 初期化スクリプトに | 必要操作のみ GRANT、テーブル単位で限定 |
| Dockerfile で `USER root` のまま | マルチステージ + 非特権ユーザー、`COPY --chown` |
| IAM ポリシーに `Action: "*"` / `Resource: "*"` | 必要 Action と Resource ARN を限定 |
| API トークンにスコープを設定しない | エンドポイントごとにスコープ定義、`requireScope()` |
| 環境変数に全シークレット注入 | 各サービスに必要なシークレットのみ |
| 開発・本番で同じ認証情報 | 環境別、開発者の本番直接アクセス禁止 |
| 長命 API キー（有効期限なし） | 短命トークン + リフレッシュ、IAM Role / OIDC |
| Admin ロール 1 つで全員に付与 | 責任範囲別にロール分離、緊急時は PIM/JIT で昇格 |
| 一度設定した権限を見直さない | 四半期棚卸し、IAM Access Analyzer 活用 |
| Kubernetes で `cluster-admin` の RoleBinding | namespace 単位の `Role` + `ServiceAccount` 分離 |

## レビュー時のチェック観点（横断）

Layer 6 のコードをAIコードレビュー観点でレビューする時、以下の観点を最低限見る:

### 1. 「データとコードの混同」を起こしていないか
- HTML 文字列を `+` / `${}` / f-string で結合していないか
- SQL を文字列結合で組み立てていないか（`$queryRawUnsafe` / `text()` + 変数 / `DB::raw`）
- `href` / `src` 属性にユーザー入力を直接渡していないか
- → 構造的分離（プリペアドステートメント・テンプレートエンジン自動エスケープ）に置き換え

### 2. デフォルト値が allowlist になっているか
- `Allow-Origin` / `Allow-Methods` / `Allow-Headers` に `*` が混入していないか
- `GRANT ALL PRIVILEGES` / IAM `Action: "*"` / `Resource: "*"` がないか
- Dockerfile に `USER root`（明示・暗黙含む）がないか
- → 「全拒否から始めて必要なものだけ許可」の Default Deny 原則

### 3. 保護の意味を取り違えていないか
- CORS で CSRF を防いだつもり / `HttpOnly` で XSS の被害なしと判断 / `SameSite=Lax` のみで CSRF 完全防御と判断
- `npm audit` クリーンで依存安全と判断 / レート制限のみで DoS 対策完了と判断
- → 各防御策の射程と限界を明示する。多層防御（Defense in Depth）の発想

### 4. 除外・バイパスが必要最小限か
- CSRF 除外、CORS 除外、認証ミドルウェアバイパス、レート制限除外がルート単位で明示されているか
- 開発時の便宜（`Allow-Origin: *`、CSRF disable、debug stack 露出）が本番ビルドに混入していないか
- → 環境分岐より「テスト用 HTTP クライアントで本番設定を再現」の方が安全

### 5. Assume Breach の発想に立っているか
- DB 接続ユーザーが `DROP` 権限を持っていないか（SQLi 成功時の被害限定）
- IAM Role が必要最小限の Action / Resource に絞られているか
- API キー / トークンが短命でローテーション可能か
- Docker コンテナが非 root で動いているか
- → 「侵害は起きる」前提で爆発半径を最小化

## 関連

- [[_anti-patterns/_index]]
- [[_starter/03_AIコーディング時代の学び方]]
- [[_starter/04_AI協働の基本動作]]
- 各トピックの「AIエージェントとの協働」章
