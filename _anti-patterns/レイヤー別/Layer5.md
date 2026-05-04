# Layer 5: パフォーマンス・信頼性 — AI実装アンチパターン集

> Layer 5（[[CoreWebVitals]] / [[CDN]] / [[ロードバランシング]] / [[非同期処理とメッセージキュー]] / [[モニタリング]] / [[パフォーマンス最適化]]）の改修で集約された、AIコーディングエージェントが頻発させるアンチパターンの索引。

## このレイヤーで頻出する「AIの癖」

Layer 5 のコードは「**動く / 速いように見える / 監視できているように見える**」が、実は**事故の地雷を抱えた**コードを AI が量産しやすい。Layer 5 アンチパターンには次の6つの根がある（細かい派生はトピック別索引を参照）:

- **「予防的・横並び最適化」誤認** — `React.memo` / `useMemo` / 全カラムインデックス / `Promise.all` 多用 / 全関数 `async` 化 を**計測根拠なし**で適用。Donald Knuth の格言「Premature optimization is the root of all evil」をそのまま無視するパターン
- **「ステートレス前提を破る」誤認** — `express-session` のメモリストア、`new Map()` のローカルキャッシュ、`/var/uploads/` へのファイル保存。**LB 配下では別サーバーに振り分けられる前提**を AI が忘れ、Sticky Session で誤魔化す
- **「キャッシュの誤用と情報漏洩」誤認** — 認証付きレスポンスに `Cache-Control: public`、キャッシュキーに `user_id` 抜け、TTL 未設定、HTML に長期キャッシュ、`Vary` ヘッダーの取りこぼし。**ユーザー A のデータが B に返る致命的事故**は AI が最も見落としやすい
- **「冪等性の欠落と握りつぶし」誤認** — メッセージハンドラに `Idempotency-Key` / 重複チェックなしで外部 API を呼ぶ、`try-catch` で全例外を空 catch して ACK。**at-least-once 配信前提**を忘れて、重複送金 / 重複メール / 障害透明化を引き起こす
- **「観測の片手落ち」誤認** — メトリクスラベルに `user_id` / 動的 `path` を入れてカーディナリティ爆発、`req.body` / `req.headers` 全体ダンプで PII 漏洩、`/health` の極端さ（重すぎ or 軽すぎ）、`console.log` 非構造化、5xx 1件で PagerDuty 発報。**「見えるようにしているつもり」が事故 / コスト爆発を生む**
- **「ライフサイクル無視」誤認** — `process.exit(0)` の即時終了、`server.close()` / `worker.close()` 不使用、`SIGTERM` ハンドリング不在、LCP 候補画像に `loading="lazy"`、サードパーティスクリプトを `<head>` に同期 `<script>`。**「いつ生まれていつ死ぬか」「いつ読み込まれていつ実行されるか」の制御責任を放棄**

これらは多くがレイヤー横断のパターン（**過剰なフォールバック / 予防的最適化 / 既存機能の再発明 / 自動防御の解除 / 計測根拠なき複雑化**）に紐付く。

## トピック別アンチパターン索引

### [[CoreWebVitals]]

| アンチパターン | レビュー観点 |
|---|---|
| 全画像に `loading="lazy"` を一律適用 | LCP 候補画像は `eager`（デフォルト） |
| LCP 候補に `loading="lazy"` + `fetchpriority="high"` 併用 | `lazy` が優先されて `fetchpriority` 無効化、`eager` に修正 |
| `useEffect` でデータフェッチ後にレイアウトサイズが変わる UI | スケルトン UI / `min-height` / `aspect-ratio` で領域確保 |
| `<img>` の `width` / `height` 未指定 | アスペクト比確定で CLS 防止 |
| バンドル分割なしの巨大 JS | ルートベースのコード分割（dynamic import） |
| `requestIdleCallback` / Web Worker をボトルネック未確認のまま予防適用 | DevTools Profiler で実測してから |
| Web フォントの `font-display` 未指定 | `font-display: swap` + `<link rel="preload">` |
| アニメーションに `top` / `left` を使用 | `transform: translate()` で合成レイヤー |
| Critical CSS のインライン化を CSP `unsafe-inline` で許可 | `nonce` / `hash` ベースの CSP |
| Lighthouse のスコアだけで完了判定 | `web-vitals` で本番フィールドデータ確認 |
| `<head>` に大量の同期 `<script>` | `defer` / `async`、可能なら body 末尾 |
| サードパーティスクリプト（広告・分析）を同期読み込み | `async` + Partytown / Facade パターン |

### [[CDN]]

| アンチパターン | レビュー観点 |
|---|---|
| 全リソースに `Cache-Control: no-store` | 静的アセットには `max-age=31536000, immutable` |
| 認証付きレスポンスに `Cache-Control: public` | `private` または `no-store`、`Vary: Authorization` |
| HTML に長期キャッシュ | HTML は `no-cache` / `max-age=0, must-revalidate` |
| クエリパラメータでキャッシュバスティング (`?v=123`) | ファイル名にハッシュを含める方式 |
| 動的 CORS で `Vary: Origin` 抜け | A サイトのレスポンスが B にキャッシュされて事故 |
| 圧縮レスポンスで `Vary: Accept-Encoding` 抜け | 非対応ブラウザに gzip が渡って表示崩れ |
| 多言語サイトで `Vary: Accept-Language` 抜け | 言語別 URL に分けるか `Vary` で対応 |
| 全レスポンスに `Vary: *` | キャッシュ無効化と等価 |
| 毎デプロイで全エッジパージ | Cache Busting に統一、パージは緊急時のみ |
| エッジ Worker でデータベース書き込み | 書き込みはオリジンに集約 |
| オリジン IP を DNS に直接公開 | CDN を回避した直接攻撃の足場、ファイアウォール必須 |
| `stale-while-revalidate` の `max-age` を長くしすぎ | `max-age` 短く、`swr` 長くで使い分け |
| CDN 設定をコンソール手動操作 | Terraform / Pulumi で IaC 化 |

### [[ロードバランシング]]

| アンチパターン | レビュー観点 |
|---|---|
| セッションをインメモリストアに保存 | Redis 等の外部ストアに移行、または Cookie ベース JWT |
| アップロードファイルをローカルディスクに保存 | S3 等のオブジェクトストレージ |
| ヘルスチェックで全テーブル `SELECT COUNT(*)` や全外部 API 確認 | 軽量化 + 詳細は別エンドポイント |
| ヘルスチェックが `res.send('ok')` のみ | 最低限の DB 疎通確認は含める |
| 全アクセスログを `/health` にも適用 | パスフィルタで除外 |
| `process.exit(0)` で即時終了 | `server.close()` + 既存処理完了待ち |
| Sticky Session で「ステートフルでも動く」と判断 | ステートを外部化、Sticky は移行期の暫定策 |
| クライアント IP ハッシュで Sticky | プロキシ経由で IP 集中、Cookie ベースに |
| LB を単一構成で運用 | Active-Standby / Active-Active |
| `Connection: keep-alive` の TTL がアップストリームより長い | アップストリームを長く、LB を短く |
| `X-Forwarded-For` を無条件で信頼 | `trust proxy` を信頼できる段数だけに制限 |
| Round Robin 一択 | Least Connections / Random with Two Choices を検討 |

### [[非同期処理とメッセージキュー]]

| アンチパターン | レビュー観点 |
|---|---|
| ハンドラに冪等性チェックなし | メッセージ ID + 重複チェック、`Idempotency-Key`、`ON CONFLICT DO NOTHING` |
| グローバル `try-catch` で全例外を握りつぶして ACK | エラー種別ごとに NACK / DLQ / リトライ分類 |
| エラー時に無限リトライ | `max_retries` 設定、超過は DLQ へ |
| DB 書き込み後に `await queue.add` を直接呼ぶ | Transactional Outbox パターン |
| メッセージ payload に巨大データ（画像バイナリ等） | S3 / DB に格納、メッセージには ID / URL のみ |
| Visibility Timeout が処理時間より短い | 処理時間最大値 + マージン、長時間ジョブは延長 API |
| `acks_late=False`（処理開始時 ACK） | `acks_late=True` + 冪等ハンドラ |
| ワーカーシャットダウンで `process.exit(0)` | `worker.close()` + in-flight 完了待ち |
| 順序保証なし環境で順序前提のロジック | FIFO / Kafka パーティションキー、または順序非依存設計 |
| すべてを非同期化して「高速化」 | 「ユーザーが結果を待つか」で判断 |
| 役割の異なるジョブを単一キューに混在 | 役割ごとにキューを分ける |
| メトリクス監視なし（キュー深さ・DLQ） | Prometheus / Datadog にメトリクス出力 |
| メッセージ ID にシーケンシャル DB ID やタイムスタンプ | UUID / ULID |

### [[モニタリング]]

| アンチパターン | レビュー観点 |
|---|---|
| `console.log` / `print` の非構造化ログ | pino / winston / structlog で JSON 構造化 |
| メトリクスラベルに `user_id` / 動的 path / クエリ | 固定列挙値のみ、動的値はログ / トレースへ |
| `path` ラベルに動的 URL 生値（`/users/123`） | テンプレ化（`/users/:id`） |
| `Histogram` のバケット未設定 | サービス特性に合わせたバケット指定 |
| 平均値だけのダッシュボード | p50 / p95 / p99 のパーセンタイル |
| `req.body` / `req.headers` 全体を `info` ログにダンプ | フィールド単位、機密項目は除外 |
| 全リクエストを DEBUG レベルでログ | 本番は INFO 以上、DEBUG はサンプリングか動的有効化 |
| `catch (err) {}` の空 catch | 構造化ログ + メトリクスのエラーカウンター + 必要に応じて再 throw |
| `console.error('failed')` のみ | エラーオブジェクト本体 + コンテキストを渡す |
| ヘルスチェックパスもアクセスログ記録 | パスフィルタで除外 |
| 5xx 1件で PagerDuty 発報 | 比率 + 持続時間で判定 |
| アラートに Runbook リンクなし | 各アラートに対応手順リンク必須 |
| メトリクス・ログ・トレースの ID 連動なし | 共通 `trace_id` / `request_id` を全シグナルに |
| OpenTelemetry のサンプリング 100% | head-based 1〜10% + tail-based でエラー / 高レイテンシ 100% |
| ビジネスメトリクス監視なし | 技術 + ビジネスメトリクスの両軸 |

### [[パフォーマンス最適化]]

| アンチパターン | レビュー観点 |
|---|---|
| 全関数にメモ化 / 全コンポーネントに `React.memo` | DevTools Profiler で計測してから |
| 単純な同期処理まで `async/await` | I/O バウンドのみ |
| 全カラムにインデックス追加 | `EXPLAIN ANALYZE` 結果に基づく |
| 早すぎるマイクロサービス分割 | モノリスで計測してから分離判断 |
| キャッシュで N+1 / O(n²) を隠蔽 | 根本原因を直してからキャッシュ |
| 並列化で外部 API レート制限超過 | 並列度を制限（`p-limit` / Semaphore） |
| 副作用順序依存を破壊する並列化 | 独立処理のみ並列、順序依存はシーケンシャル |
| マイクロ最適化（ループの書き換え等） | ボトルネック確認後にのみ採用 |
| Redis キャッシュ TTL 未設定 | 必ず TTL 設定 + `stale-while-revalidate` |
| キャッシュキーに `user_id` を含めない | ユーザー / テナントスコープ必須（情報漏洩防止） |
| ロードテストなしの本番投入 | k6 / Locust / Vegeta で負荷試験 |
| パフォーマンスバジェットなし | LCP / API レイテンシの数値目標 |

## 横断的な「AIの癖」と対策パターン

| パターン | 何が起きるか | レビュー観点 |
|---|---|---|
| **予防的最適化** | 計測根拠なしに `React.memo` / メモ化 / 並列化 / インデックスを乱発、複雑性とオーバーヘッドが利益を上回る | プロファイル / `EXPLAIN ANALYZE` / DevTools Profiler の実測値を要求する |
| **キャッシュ誤用** | 認証付きレスポンスを共有キャッシュ、TTL 未設定、`user_id` 抜けキー → 情報漏洩 / 永続古データ | キャッシュキー設計とスコープを必ずレビュー |
| **冪等性の欠落** | at-least-once 配信前提を忘れ、リトライ時に重複送金・重複メール | メッセージ ID 重複チェック、`Idempotency-Key`、`ON CONFLICT` |
| **エラー握りつぶし** | catch で何もせず、ログ・メトリクスにも残らず、障害が透明化 | catch では必ず構造化ログ + メトリクス更新 |
| **計測値なき自信** | 「速くなりました」「安定しました」が改善前後の数値根拠なし | 改善前後の p50 / p95 / p99 / RPS を必須化 |
| **ステートフル化の混入** | LB 配下でセッション・キャッシュ・ファイルがローカル | 外部ストア (Redis / S3) に外出し |
| **シャットダウンの即殺** | SIGTERM で `process.exit(0)`、進行中処理中断 | `server.close()` + ドレインモードへの遷移 |
| **メトリクス爆発** | カーディナリティ無界の値（user_id / 動的 path）をラベルに | 固定列挙値のみ、動的値はログへ |
| **PII 漏洩** | `req.body` / `req.headers` 全体ダンプで認証トークン・カード番号がログに | フィールド単位の出力、機密項目を明示除外 |
| **段階的劣化なき実装** | 部分障害で全停止（外部 API 1 つ落ちて自分も落ちる） | サーキットブレーカー / リトライ + バックオフ / フォールバック |

## 関連

- [[_starter/03_AIコーディング時代の学び方]]
- [[_starter/04_AI協働の基本動作]]
- [[_anti-patterns/_index|AIアンチパターン索引]]
- 各トピックの「AIエージェントとの協働」章
