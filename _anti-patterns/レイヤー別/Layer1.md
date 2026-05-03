# Layer 1: OSインフラ — AI実装アンチパターン集

> Layer 1（[[プロセスとスレッド]] / [[ファイルシステムとIO]] / [[メモリ管理]] / [[Docker]] / [[Linux基本操作]] / [[クラウドサービスモデル]]）の改修で集約された、AIコーディングエージェントが頻発させるアンチパターンの索引。

## このレイヤーで頻出する「AIの癖」

OSインフラ層のコードは「**動いてしまう**」が、長期運用や障害時に致命的な問題を起こすコードを AI が量産しやすい。Layer 1 のアンチパターンには次の共通する根がある:

- **「とりあえず最大値・最大権限」で設定する** — Lambda の `timeout: 15分`、IAM の `Resource: "*"`、`chmod 777`、`memorySize: 上限値`
- **「同期API・全読み込み・即時実行」を選ぶ** — `readFileSync`、同期Crypto API、`process.exit()`、`fsync` 毎回
- **「リソース解放のフック」を書き忘れる** — ファイルディスクリプタ、setInterval、AbortController、WebSocket close、Graceful Shutdown
- **「キャッシュ・キュー・ログ」を上限なく成長させる** — モジュールスコープの `Map / Array`、Error オブジェクト蓄積、accessLog 無制限

これらは多くがレイヤー横断のパターン（**過剰なフォールバック / 既存機能の再発明 / 不要な抽象化 / リソース管理忘れ**）に紐付く。

## トピック別アンチパターン索引

### [[プロセスとスレッド]]

| アンチパターン | レビュー観点 |
|---|---|
| CPUバウンド処理をイベントループ内で実行（同期Crypto、大JSON.parse） | Worker Threads / 別プロセスへ分離 |
| 全リクエストに `fork` / `spawn` | プロセスプール、イベント駆動、接続プールへ |
| `process.exit()` の即時呼び出し | `server.close()` のコールバックで終了する Graceful Shutdown |
| sleep/busy-wait でプロセス間同期 | IPC / 外部ストア（Redis 等）で同期 |
| `os.cpus().length` でワーカー数決定 | コンテナの CPU 制限を読む（環境変数 / cgroup） |

### [[ファイルシステムとIO]]

| アンチパターン | レビュー観点 |
|---|---|
| 全ファイルを `readFileSync` / `readFile` で一括読み込み | ストリームAPI（`createReadStream` / `readline` / `io.Reader`） |
| 書き込みごとに `fsync` を呼ぶ | コミット点だけに限定（バッチ） |
| I/Oエラーを `try-catch` で握りつぶす | エラーは上位伝播、`errno` までログ |
| 存在チェック→作成の TOCTOU | `O_CREAT \| O_EXCL` で原子的に作成 |
| 一時ファイルを `/tmp/固定名` で作る | `mkstemp` / `mkdtemp` で一意名生成 |
| 温メモリキャッシュ前提の `readFileSync` 多用 | 明示的なキャッシュレイヤー（Redis 等）に置く |

### [[メモリ管理]]

| アンチパターン | レビュー観点 |
|---|---|
| 大きな JSON を `JSON.parse()` で一括パース | ストリーミングJSONパーサー（`stream-json` / `json.Decoder`） |
| 配列に結果を `.push()` し続けるバッチ処理 | ページネーション/カーソル/ジェネレータで逐次出力 |
| `Buffer.concat()` をループ内で繰り返し | `pipeline` / 配列に貯めて最後に1回だけ concat |
| Error オブジェクトを保持・蓄積 | `err.message` だけ残してログに書き出す |
| モジュールスコープの `Map / Array` を上限なし | LRU / TTL で必ず上限、`WeakMap` 検討 |
| `setInterval` の `clearInterval` 忘れ | `useEffect` cleanup / `AbortController` で一括解除 |

### [[Docker]]

| アンチパターン | レビュー観点 |
|---|---|
| `RUN apt-get update` と `install` を別 RUN に分割 | 同一 RUN に結合 + `rm -rf /var/lib/apt/lists/*` |
| `set -e` / `pipefail` なしの RUN | `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` を Dockerfile 冒頭に |
| 「念のため」のパッケージ大量インストール | 必要最小限に。デバッグ用は別イメージ |
| `COPY . .` を `RUN npm ci` の前に配置 | 依存定義 → install → アプリ COPY の順に |
| `ENV` でシークレット直書き | `--mount=type=secret` / ランタイム環境変数 |
| `USER root` のまま実行 | `USER` で非 root ユーザーに切り替え |
| `FROM xxx:latest` | 明示バージョン（`@sha256:` digest 固定も検討） |

### [[Linux基本操作]]

| アンチパターン | レビュー観点 |
|---|---|
| `chmod -R 777` を Dockerfile やデプロイで使う | 644/755/750/600 等の最小権限 |
| シェルスクリプトで `\|\|` による握りつぶし | `set -euo pipefail` で一括管理 |
| `set -euo pipefail` なしのスクリプト | スクリプト冒頭に必ず書く |
| 変数を `$VAR` のままクォートせず使用 | `"$VAR"`、`rm` 前に `[ -n "$VAR" ]` ガード |
| `curl \| bash` でスクリプトをインストール | パッケージマネージャ / 署名検証 |
| `sudo` を毎行に付けて実行 | systemd の `User=` で実行ユーザー切り替え |
| `nohup command &` でデーモン化 | systemd サービスに登録、`Restart=on-failure` |

### [[クラウドサービスモデル]]

| アンチパターン | レビュー観点 |
|---|---|
| 全サービスを Lambda で提案する | ワークロード特性で選定。長時間処理は ECS / Batch |
| ベンダー固有 SDK をビジネスロジック層に散らす | インフラ層に閉じ込める（ポート&アダプター） |
| `*` ワイルドカードの IAM ポリシー | `Action` / `Resource` を最小限に明示 |
| 最初からマルチクラウド戦略 | まず1クラウドに習熟。必要が出てから検討 |
| コスト見積もりなしのクラウド採用 | PoC で月額試算、Egress も含める |
| コア機能の SaaS への深い依存 | 抽象化レイヤー or 移行計画を持つ |

## レイヤー横断の共通テーマ

Layer 1 のアンチパターンは、より上位の **「パターン別」** カテゴリに属するものが多い:

- **過剰なフォールバック** — `|| true`、`|| echo "failed"`、try-catch の握りつぶし、`fsync` 毎回呼び
- **防御的すぎるエラーハンドリング** — 内部関数への `null` チェック、`if (!users)` ガードの連鎖（Layer 0 のレビュー観点と共通）
- **既存機能の再発明** — 既存の systemd / logrotate / cron / `mkstemp` を使わず自前実装
- **リソース管理の責任放棄** — fd / setInterval / AbortController / Graceful Shutdown を書かない

→ 横断索引: [[_anti-patterns/_index|AIアンチパターン索引]]

## 整備状況

- 2026-05-04: Week 3 改修で 6 トピック分のアンチパターンを集約。Layer 0 と合わせ、`/review-ai-code` のナレッジソースとして利用可能
- 今後: Week 4 以降の Layer 2-7 改修と同期し、レイヤーをまたぐパターン（特にセキュリティ・パフォーマンス領域）を **パターン別** ファイルへ集約予定
