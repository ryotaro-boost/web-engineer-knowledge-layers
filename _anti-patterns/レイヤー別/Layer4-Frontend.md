# Layer 4 Frontend: アプリケーション — AI実装アンチパターン集

> Layer 4 Frontend（[[HTML-CSS-JS]] / [[DOMと仮想DOM]] / [[コンポーネント設計]] / [[状態管理]] / [[アクセシビリティ]]）の改修で集約された、AIコーディングエージェントが頻発させるアンチパターンの索引。

## このレイヤーで頻出する「AIの癖」

Frontend のコードは「**ブラウザで動いて見栄えはいいが、キーボードユーザ・スクリーンリーダーユーザには使えない**」「**最初は動くが状態が増えると壊れる**」コードを AI が量産しやすい。Layer 4 Frontend のアンチパターンには次の共通する根がある:

- **「`<div>` で全部解決しがち」誤認** — `<div onClick>` をボタン代わり、`<div class="header">` を `<header>` 代わり、`<div tabindex="0">` をフォーカス可能要素代わり。ネイティブ要素なら無料で得られるアクセシビリティ・キーボード操作・SEO 効果を全て自前実装するハメになる
- **「予防的最適化」誤認** — `React.memo` を全コンポーネントに適用、`useMemo` / `useCallback` を予防的に大量配置。シンプルなコンポーネントでは比較コストのほうが利益を上回る。**計測してから最適化**が原則
- **「過剰分割」誤認** — 10行の JSX を別ファイルに切り出す、`<UserNameText>` `<UserEmailText>` のような単一 `<span>` ラッパーコンポーネント乱立。Rule of Three（3回目に出てから抽出）に違反、ファイル数増加だけが利益を上回る
- **「派生状態を別 state にする」誤認** — `items` と `itemCount`、`products` と `filteredProducts` を別 state に持ち、`useEffect` で同期。二重レンダリング・無限ループ・同期漏れの温床。**派生はレンダリング中に計算**するか `useMemo`
- **「サーバー状態とクライアント状態の混在」誤認** — API データをグローバルストアや `useState` + `useEffect` で自前管理。ローディング・エラー・キャッシュ・再取得を全部自前実装。サーバー状態は **TanStack Query / SWR** に分離
- **「key にインデックスや乱数」誤認** — `key={index}` `key={Math.random()}` `key={Date.now()}`。並べ替え・追加・削除でフォーカスや入力中の値が失われる。**データ固有の安定 ID** を使う
- **「`outline: none` でフォーカス削除」誤認** — フォーカスリングが「ダサい」という理由で消す。キーボードユーザが操作不能。`:focus-visible` でカスタマイズ
- **「ARIA を予防的に大量付与」誤認** — `<button role="button" aria-label="送信">送信</button>` のようなネイティブで足りるものに ARIA 重ね付け。「No ARIA is better than bad ARIA」
- **「セキュリティの自動防御を解除」誤認** — `dangerouslySetInnerHTML` をユーザー入力にそのまま使用。XSS の直接的な原因

これらは多くがレイヤー横断のパターン（**過剰なフォールバック / 予防的最適化 / 既存機能の再発明 / セマンティクスの放棄 / 自動防御の解除**）に紐付く。

## トピック別アンチパターン索引

### [[HTML-CSS-JS]]

| アンチパターン | レビュー観点 |
|---|---|
| 全て `<div>` で構築 | セマンティック要素 (`<nav>`, `<main>`, `<button>`) を使う |
| `<div onclick>` をボタンとして使用 | `<button>` を使う |
| インラインスタイル `style="..."` 多用 | クラスベースの CSS 設計 |
| インラインイベント `onclick="..."` | `addEventListener` で外部 JS から登録 |
| `document.getElementById` 多用の手続き的 DOM 操作 | 宣言的 UI フレームワークかイベントデリゲーション |
| `outline: none` でフォーカス表示削除 | `:focus-visible` でカスタムスタイル |
| `<img>` に `alt` 属性なし | 意味のある画像は説明、装飾は `alt=""` |
| `<script>` を `<head>` に `defer` なしで配置 | `defer` または `<body>` 末尾、`type="module"` はデフォルトで defer |
| `!important` の多用 | 詳細度の設計を見直す |
| ID セレクタでスタイリング | クラスセレクタを使う |
| CSS の `@import` でファイル分割 | バンドラの import またはビルド時連結 |
| メディアクエリで PC ファースト | `min-width` のモバイルファースト |
| `prefers-reduced-motion` 無視のアニメーション | `@media (prefers-reduced-motion: reduce)` で抑制 |

### [[DOMと仮想DOM]]

| アンチパターン | レビュー観点 |
|---|---|
| `key={index}` でリスト要素を識別 | データ固有の `item.id` を使う |
| `key={Math.random()}` `key={Date.now()}` | 安定 ID を使う、強制再マウントが必要なら状態キー (`<Comp key={mode}>`) |
| 全コンポーネントに `React.memo` 適用 | DevTools Profiler で計測、ボトルネックのみメモ化 |
| `useMemo` / `useCallback` の予防的乱用 | 高コスト計算 / 子のメモ化が前提のときのみ適用 |
| インラインオブジェクト・関数を props に渡しメモ化を破壊 | 親で `useMemo` / `useCallback` で参照を安定化 |
| `useEffect` 内で大量の DOM 直接操作 | `useRef` 経由で最小限に |
| `dangerouslySetInnerHTML` にサニタイズなしの入力 | DOMPurify でサニタイズ、または仮想DOM の自動エスケープに任せる |
| 状態をミュータブルに更新 (`state.items.push`) | `[...state.items, newItem]` イミュータブル、または Immer / Valtio |
| 巨大リストを `.map()` で全件レンダリング | TanStack Virtual / react-window で仮想スクロール |
| `useEffect` で状態を派生 | レンダリング中に計算するか `useMemo` |
| Server Component に状態フックを使う | `'use client'` で Client Component に分離 |

### [[コンポーネント設計]]

| アンチパターン | レビュー観点 |
|---|---|
| 1つの `<div>` のラッパーコンポーネント乱立 | 再利用性・独立した状態・テスト需要のいずれかが正当化条件 |
| 全コンポーネントに大量の props 設定オプション | 最初はシンプルに、必要になったら拡張 |
| `children` を使わず props で全 UI 注入 | 合成パターンや Compound Component (`<Card.Header>`) を優先 |
| `<Spacer>` `<FlexRow>` `<Center>` のレイアウト用 div ラッパー | CSS の `gap` / Flexbox / Grid を使う |
| `React.memo` を全コンポーネントに適用 | DevTools Profiler で計測、ボトルネックのみ |
| 1コンポーネント = 1ファイルの厳格適用 | 外部公開するもののみ独立ファイル |
| Container / Presentational の機械的分離 | カスタムHook でロジック抽出する方が現代的 |
| ユーザー情報全体を props で渡す（必要なのは name だけ） | 必要な値だけを props に |
| boolean props を否定形 (`isNotDisabled`) | 肯定形（`disabled={false}`）に統一 |
| イベントハンドラ命名の乱れ | `on + 動詞`（`onClick`、`onSubmit`）で統一 |
| Server Component に状態フックを使う | `'use client'` で Client Component に分離 |
| 巨大コンポーネント (500行+) の責務混在 | 200〜300行を超えたら分割の検討 |

### [[状態管理]]

| アンチパターン | レビュー観点 |
|---|---|
| 全ての状態をグローバルストアに入れる | ローカルで完結する状態は `useState` |
| サーバー状態を `useState` + `useEffect` で管理 | TanStack Query / SWR を使う |
| サーバー状態を Redux / Zustand に詰める | グローバルストアにはクライアント状態のみ |
| 派生可能な値を別 state にする | レンダリング中に計算、重ければ `useMemo` |
| `useEffect` で state を同期 | 派生はレンダリング中、イベント時の同時更新はハンドラ内で |
| ミュータブル更新（`state.items.push(...)`） | `[...state.items, newItem]` 等イミュータブル、または Immer / Valtio |
| Context に高頻度更新の state を入れる | 高頻度状態は Zustand / Jotai に分離 |
| Context を分割せず1つの Provider で巨大な値を共有 | 関心ごとに Provider を分ける |
| 最初から Redux 導入 | 段階的導入（useState → State Lifting → Context → 外部ストア） |
| 状態のネスト構造をそのまま保持 | 正規化（`Record<id, entity>`）に展開 |
| `useReducer` の state を mutate | 必ず新オブジェクトを返す、または Immer の `produce` |
| ローカルストレージへの保存を `useEffect` で都度実行 | persist ミドルウェアや debounce で頻度を抑える |

### [[アクセシビリティ]]

| アンチパターン | レビュー観点 |
|---|---|
| `<div onclick>` をボタンとして使用 | `<button>` を使う |
| `<div role="button" tabindex="0" onkeydown>` の自前実装 | `<button>` を使う |
| `outline: none` でフォーカス表示削除 | `:focus-visible` でカスタムスタイル |
| `tabindex="1"` 等の正の値 | `0` または `-1` のみ |
| `<img>` に `alt` 属性なし | 意味のある画像は説明、装飾は `alt=""` |
| `aria-label` と可視テキストの不一致 | 一致させる |
| フォーム `<input>` に `<label>` なし | `<label for="id">` または `aria-label` |
| ネイティブ `<dialog>` を使わず自前モーダル | `<dialog>` + `showModal()` |
| `prefers-reduced-motion` 無視のアニメーション | `@media (prefers-reduced-motion: reduce)` |
| 色だけでエラー・状態を表現 | アイコン・テキスト・パターン併用 |
| バリデーションエラーが視覚的にだけ表示 | `role="alert"` または `aria-live="assertive"` |
| 巨大なフォーカストラップ | `<dialog>` 使用、または focus-trap-react |
| 矢印キー未対応のタブ・メニュー UI | WAI-ARIA Authoring Practices Guide のパターンに従う |
| ライブリージョン未使用 | `aria-live="polite"` (通常) / `assertive` (緊急) |

## レイヤー横断の癖（Layer 4 Frontend で目立つもの）

### 「`<div>` 全部主義」誤認
- `<div class="header">` `<div class="nav">` 等で全部 `<div>`
- `<div onclick>` をボタン代わり
- `<div role="button" tabindex="0">` の自前実装
- → アクセシビリティ・SEO・キーボード操作・保守性が全方位で劣化

### 「予防的最適化」誤認
- `React.memo` を全コンポーネントに適用
- `useMemo` / `useCallback` を予防的に大量配置
- ロジック未確定の段階で「将来こうなるかも」と props を増やす
- → 計測してから最適化が原則。早すぎる抽象化は使われない設定だらけの API を生む

### 「派生状態を別 state にする」誤認
- `items` と `itemCount`、`products` と `filteredProducts` を別 state
- `useEffect` で「A の state が変わったら B を更新」の同期パターン
- → 二重レンダリング・無限ループ・同期漏れ。**派生はレンダリング中に計算**

### 「サーバー状態とクライアント状態の混在」誤認
- API データを `useState` + `useEffect` で自前管理
- グローバルストアに API データを詰め込む
- → ローディング・エラー・キャッシュ・再取得・楽観的更新を全部自前実装することに。**TanStack Query / SWR** が標準解

### 「`key` にインデックス・乱数」誤認
- `key={index}` で配列インデックス使用
- `key={Math.random()}` `key={Date.now()}` で毎回新 key
- → 並べ替え・追加・削除でフォーカス・入力中の値が失われる。**データ固有の安定 ID** を使う

### 「セキュリティ自動防御の解除」誤認
- `dangerouslySetInnerHTML` をユーザー入力にそのまま使用 → XSS
- React の自動エスケープを `innerHTML` で迂回
- → デフォルトの安全機能を活かす。生 HTML が必要なら DOMPurify 等でサニタイズ

### 「アクセシビリティ後付け可能」誤認
- `<div onclick>` を後で `<button>` に置き換える計画
- フォーカス管理・キーボード操作を後付けで実装
- → コンポーネント構造に深く関わるため、後付けは大規模リファクタリング。**設計初期にセマンティック HTML** で組む

## 関連

- [[_anti-patterns/_index|AIアンチパターン索引（トップ）]]
- [[_anti-patterns/レイヤー別/Layer4-Backend|Layer 4 Backend アンチパターン集]]
- [[_starter/04_AI協働の基本動作]]
