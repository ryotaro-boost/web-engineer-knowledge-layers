---
layer: 0
topic: 文字コード（UTF-8）
status: 🔴 未着手
created: 2026-03-27
---

# 文字コード（UTF-8）

> **一言で言うと:** Webは世界中の文字を扱う。エンコーディングの誤解は文字化けとセキュリティホールの両方を生む。

## なぜ必要か

コンピュータは内部的にすべてのデータをバイト列（0と1の並び）として扱う。文字コードとは「どのバイト列がどの文字に対応するか」を定義する取り決めであり、これがなければ人間が読める文字を保存・送受信できない。

文字コードの知識がないと以下の問題が発生する:
- **文字化け** — 書いたはずの文字が `ï¿½` や `æ–‡å­—` のように壊れて表示される
- **データ破損** — データベースに保存した文字列が読み出し時に壊れる
- **セキュリティ脆弱性** — エンコーディングの不一致を悪用した攻撃（例: UTF-7 XSS、[[ホモグラフ攻撃と正規化|ホモグラフ攻撃]]）
- **文字列操作のバグ** — 絵文字や日本語のような多バイト文字の長さ計算・切り出しが壊れる

## どの問題を解決するか

### 問題1: 文字集合の統一 — Unicode の登場

かつて各国・各ベンダーが独自の文字コードを使っていた（Shift_JIS, EUC-JP, ISO-8859-1, GB2312 など）。[[ASCII]] はその中で最も基本的な文字集合であり、UTF-8 の設計に大きな影響を与えた。異なる文字コード間の変換は不完全で、国際的なデータ交換は困難だった。

**Unicode** はこの問題を「世界中のすべての文字に一意の番号（コードポイント）を割り当てる」ことで解決した。例えば「あ」は常に U+3042 である。

### 問題2: 効率的なバイト表現 — UTF-8 の設計

Unicode のコードポイントをそのままバイト列にする方法（UTF-32）では、ASCII 文字1つに4バイトも使い、既存の ASCII テキストとの互換性もない。

**UTF-8** は可変長エンコーディングによりこれを解決する:

| コードポイント範囲 | バイト数 | 先頭バイトパターン |
|---|---|---|
| U+0000 〜 U+007F | 1バイト | `0xxxxxxx` |
| U+0080 〜 U+07FF | 2バイト | `110xxxxx 10xxxxxx` |
| U+0800 〜 U+FFFF | 3バイト | `1110xxxx 10xxxxxx 10xxxxxx` |
| U+10000 〜 U+10FFFF | 4バイト | `11110xxx 10xxxxxx 10xxxxxx 10xxxxxx` |

この設計の利点:
- **ASCII 互換** — 英数字は1バイトのまま。既存の ASCII ファイルはそのまま有効な UTF-8
- **自己同期** — バイト列の途中からでも文字の境界を特定できる（先頭バイトと継続バイトのパターンが異なる）
- **ソート可能** — バイト列の辞書順ソートがコードポイント順と一致する
- **NUL バイトなし** — ASCII の NUL（0x00）以外のコードポイントに 0x00 が現れないため、C言語の文字列関数と互換

### 問題3: Web における標準化

1990年代には Windows・Java・JavaScript が UTF-16 を内部表現に採用したが、Web では ASCII 互換性・NUL バイト非混入・バイト順非依存という特性から UTF-8 が選ばれた。2008年に Web 上で最多のエンコーディングとなり、現在は約98%のページが UTF-8 を使用している。詳細は [[UTF-8が標準になった理由]] を参照。

HTTP ヘッダー（`Content-Type: text/html; charset=utf-8`）と HTML メタタグ（`<meta charset="utf-8">`）でエンコーディングを宣言し、ブラウザに正しいデコード方法を伝える。WHATWG の HTML Standard は新規コンテンツに UTF-8 を推奨しており、事実上の唯一の選択肢となっている。

## 他の仕組みとどう関係するか

- **下位レイヤーとの関係:**
  - [[データ構造とアルゴリズム]] — 文字列は本質的にはバイト配列であり、[[ハッシュテーブル]]のキーとして使う際にエンコーディングの一貫性が必要
  - [[計算量-BigO]] — 多バイト文字を含む文字列操作は「文字数」と「バイト数」が異なるため計算量の見積もりに影響する

- **同レイヤーとの関係:**
  - [[並行性の基本概念]] — 複数プロセスが同じファイルを異なるエンコーディングで読み書きすると破損する

- **上位レイヤーとの関係:**
  - [[Layer1-OSインフラ/_index|Layer 1: OS・インフラ]] — ファイルシステムのファイル名エンコーディング（Linux は UTF-8、Windows は UTF-16）、ロケール設定
  - [[Layer3-データ永続化/_index|Layer 3: データ永続化]] — データベースの `CHARACTER SET` / `COLLATION` 設定。MySQL の `utf8` は3バイトまでしか扱えず、`utf8mb4` が真の UTF-8
  - [[Layer6-セキュリティ/_index|Layer 6: セキュリティ]] — エンコーディング不一致を悪用した XSS、[[ホモグラフ攻撃と正規化|ホモグラフ攻撃]]、ディレクトリトラバーサル

## 誤解されやすいポイント

1. **「Unicode = UTF-8」ではない**
   Unicode は文字集合（どの文字に何番を振るか）の規格であり、UTF-8 はその符号化方式（番号をどうバイト列にするか）の一つ。UTF-16 や UTF-32 も Unicode のエンコーディングである。JavaScript の内部表現は UTF-16 であり、Python 3.3+ は PEP 393 により文字列内容に応じて Latin-1 / UCS-2 / UCS-4 を動的に選択する。

2. **「文字列の長さ」は一意ではない**
   `"café"` の長さは、バイト数（5, UTF-8）、コードポイント数（4 or 5）、書記素クラスタ数（4）のどれを指すかで変わる。`é` は1つのコードポイント（U+00E9）にも、`e` + 結合アクセント（U+0065 U+0301）の2つにもなる。絵文字 `👨‍👩‍👧‍👦` は7つのコードポイントだが見た目は1文字。詳しくは [[サロゲートペア・結合文字・書記素クラスタ]] を参照。

3. **「UTF-8 は日本語に不利」は現在では誤り**
   日本語は UTF-8 で3バイト/文字、Shift_JIS で2バイト/文字なので「サイズが1.5倍」という指摘はある。しかし現代の Web では HTTP 圧縮（gzip/Brotli）が標準であり、圧縮後の差は無視できる。エンコーディング統一による開発・運用コスト削減のメリットが圧倒的に大きい。

4. **BOM（Byte Order Mark）の扱い**
   UTF-8 にバイト順の問題はないため BOM（U+FEFF, 3バイト: `EF BB BF`）は不要。しかし Windows のメモ帳などは BOM 付き UTF-8 で保存することがあり、シェルスクリプトやCSVの先頭に見えないゴミが混入してパースエラーの原因になる。

## 設計のベストプラクティス

### 推奨パターン

- **UTF-8 をデフォルトにする** — 新規プロジェクトではファイル、DB、API すべてで UTF-8 を統一する
- **HTTP レスポンスに charset を明示する** — `Content-Type: text/html; charset=utf-8`
- **HTML の `<head>` 冒頭で宣言する** — `<meta charset="utf-8">` は最初の1024バイト以内に置く
- **DB は `utf8mb4` を使う** — MySQL の `utf8` は3バイト文字までしか格納できない。絵文字（4バイト）を扱うには `utf8mb4` が必要
- **文字列操作は書記素クラスタ単位で行う** — ユーザー向けの切り出し・カウントには `Intl.Segmenter`（JavaScript）や `grapheme-splitter` ライブラリを使う
- **正規化（NFC/NFD）を意識する** — ユーザー入力の比較前に `String.prototype.normalize('NFC')` で正規化する

### アンチパターン

- バイト列を直接切り出して文字列を truncate する → マルチバイト文字の途中で切れて壊れる
- エンコーディング指定なしでファイルを読み書きする → 環境依存の文字化け
- `encodeURIComponent()` を使わずに URL に日本語を含める → URL エンコーディング不備
- 「全角半角変換」を自前実装する → Unicode 正規化（NFKC）を使うべき

## AIによる実装のアンチパターン

| アンチパターン | なぜ問題か | 対策 |
|---|---|---|
| バイト長と文字数を混同したバリデーション | `maxlength=100` をバイト数でチェックすると日本語が33文字しか入らない | 要件に応じてコードポイント数 or 書記素クラスタ数でカウントする |
| エンコーディング変換の過剰なフォールバック | `try UTF-8 → try Shift_JIS → try EUC-JP` のような推測デコードを生成しがち | 入力のエンコーディングを仕様で決め、推測しない設計にする |
| サロゲートペアを考慮しない正規表現 | `.` が絵文字にマッチしない、`\u{1F600}` を使うべき場所で `\uD83D\uDE00` を生成 | ES2015 の `u` フラグ付き正規表現を使う |
| BOM の無条件付与/除去 | BOM が必要なケース（Excel の CSV 読み込み）と不要なケース（Web）を区別しない | 用途に応じて BOM の有無を制御する |

## 具体例

### UTF-8 のバイト表現を確認する（Python）

```python
text = "Aあ🍣"
encoded = text.encode("utf-8")
print(f"文字列: {text}")
print(f"バイト列: {encoded}")
print(f"バイト数: {len(encoded)}")       # 8 (1 + 3 + 4)
print(f"文字数: {len(text)}")             # 3

# 各文字のバイト表現
for char in text:
    b = char.encode("utf-8")
    hex_str = " ".join(f"{byte:02x}" for byte in b)
    print(f"  '{char}' U+{ord(char):04X} → [{hex_str}] ({len(b)}バイト)")
```

出力:
```
文字列: Aあ🍣
バイト列: b'A\xe3\x81\x82\xf0\x9f\x8d\xa3'
バイト数: 8
文字数: 3
  'A' U+0041 → [41] (1バイト)
  'あ' U+3042 → [e3 81 82] (3バイト)
  '🍣' U+1F363 → [f0 9f 8d a3] (4バイト)
```

### JavaScript での文字列長の罠

```javascript
const emoji = "👨‍👩‍👧‍👦";
console.log(emoji.length);           // 11 (UTF-16 コードユニット数)
console.log([...emoji].length);      // 7  (コードポイント数)

// 書記素クラスタ数（= 見た目の文字数）を正しく数える
const segmenter = new Intl.Segmenter("ja", { granularity: "grapheme" });
const segments = [...segmenter.segment(emoji)];
console.log(segments.length);        // 1  (見た目どおり)
```

### MySQL での正しい文字セット設定

```sql
-- NG: utf8 は3バイトまで（絵文字が保存できない）
CREATE TABLE bad_table (
  name VARCHAR(100) CHARACTER SET utf8
);

-- OK: utf8mb4 が真の UTF-8
CREATE TABLE good_table (
  name VARCHAR(100) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
);
```

### Node.js での BOM 除去

```javascript
import { readFileSync } from "fs";

function readUtf8(path) {
  let text = readFileSync(path, "utf-8");
  // BOM (U+FEFF) が先頭にあれば除去
  if (text.charCodeAt(0) === 0xfeff) {
    text = text.slice(1);
  }
  return text;
}
```

## 参考リソース

- [The Absolute Minimum Every Software Developer Absolutely, Positively Must Know About Unicode and Character Sets (Joel on Software)](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)
- [UTF-8 Everywhere Manifesto](http://utf8everywhere.org/)
- [Unicode 公式サイト](https://home.unicode.org/)
- 書籍: 『プログラマのための文字コード技術入門』（矢野啓介）

## 学習メモ

- ~~UTF-8 が Web の標準になった経緯と、なぜ UTF-16（Windows/Java/JavaScript 内部）ではなく UTF-8 が選ばれたかを理解する~~ → 「問題3」セクションに記載済み
- サロゲートペア・結合文字・書記素クラスタの3つは、文字列処理のバグの温床なので重点的に学ぶ
- セキュリティ観点: ホモグラフ攻撃（見た目が同じ異なるコードポイント）と正規化の関係を深掘りする
