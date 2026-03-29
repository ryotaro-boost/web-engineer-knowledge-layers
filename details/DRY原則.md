---
layer: 7
parent: "[[関心の分離]]"
type: detail
created: 2026-03-30
---

# DRY原則（Don't Repeat Yourself）

> **一言で言うと:** 「すべての知識はシステム内で一意かつ明確な表現を持つべき」という原則。ただし「コードの重複を排除せよ」という意味ではない。

## 概念

DRY原則は Andy Hunt と Dave Thomas が *The Pragmatic Programmer*（1999年）で提唱した。原文は:

> Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

ここでの「知識（Knowledge）」とは、**ビジネスルール、アルゴリズム、データの意味**といった概念的な知識を指す。ソースコードの文字列の一致ではない。

## 「重複」の2つの種類

DRY を正しく適用するには、**本質的な重複**と**偶然の重複**を区別する必要がある。

### 本質的な重複（True Duplication）

同じ「知識」が複数箇所に存在している状態。一方を変更したら、もう一方も必ず同じように変更しなければならない。

```typescript
// ❌ 税率の知識が2箇所に重複
function calculateInvoiceTotal(amount: number): number {
  return amount * 1.10; // 税率10%
}

function calculateReceiptTotal(amount: number): number {
  return amount * 1.10; // 税率10% — 税率変更時に両方修正が必要
}

// ✅ 税率の知識を一箇所に集約
const TAX_RATE = 0.10;

function calculateTotalWithTax(amount: number): number {
  return amount * (1 + TAX_RATE);
}
```

この場合、税率という「知識」が2箇所にあるので、税率変更時に片方だけ修正して不整合が起きるリスクがある。これは排除すべき本質的な重複。

### 偶然の重複（Accidental Duplication）

コードの見た目が似ているだけで、背後にある「知識」は異なる状態。変更される理由が異なるため、無理に共通化すると後で問題になる。

```typescript
// ユーザー登録のバリデーション
function validateRegistration(data: RegistrationInput): string[] {
  const errors: string[] = [];
  if (!data.name || data.name.length < 2) errors.push('名前は2文字以上');
  if (!data.email.includes('@')) errors.push('メールアドレスが不正');
  return errors;
}

// プロフィール更新のバリデーション
function validateProfileUpdate(data: ProfileInput): string[] {
  const errors: string[] = [];
  if (!data.name || data.name.length < 2) errors.push('名前は2文字以上');
  if (!data.email.includes('@')) errors.push('メールアドレスが不正');
  return errors;
}
```

一見同じだが、登録とプロフィール更新は**異なるビジネスコンテキスト**であり、将来の要件変更は独立に起きる（登録時だけ招待コードが必要になる、プロフィール更新時だけ変更理由の入力が必要になる、等）。これを共通化すると、片方の要件変更でもう片方に影響が波及し、条件分岐が増殖する。

```typescript
// ❌ 無理な共通化の末路
function validateUser(data: UserInput, context: 'registration' | 'profile'): string[] {
  const errors: string[] = [];
  if (!data.name || data.name.length < 2) errors.push('名前は2文字以上');
  if (!data.email.includes('@')) errors.push('メールアドレスが不正');
  if (context === 'registration' && !data.inviteCode) errors.push('招待コードが必要');
  if (context === 'profile' && !data.changeReason) errors.push('変更理由が必要');
  // context分岐が増え続ける...
  return errors;
}
```

## DRYが適用される「知識」の例

| 知識の種類 | DRY違反の例 | 解決策 |
|-----------|-----------|--------|
| ビジネスルール | 税率計算が複数箇所にハードコード | 定数 or 関数に集約 |
| データの構造 | API レスポンスの型定義がフロント・バックで別々に手書き | [[OpenAPIとスキーマ駆動開発]]で自動生成 |
| 設定値 | ポート番号が Dockerfile・docker-compose・アプリ設定に散在 | 環境変数で一元管理 |
| アルゴリズム | ソート処理が複数ファイルにコピペ | 共通関数に抽出 |

## DRYとWET

DRY の対義語として **WET**（Write Everything Twice / We Enjoy Typing）がある。ただし「3回目の重複が現れるまでは共通化しない」という **Rule of Three** は、偶然の重複を誤って共通化するリスクを避けるための実践的な指針として広く支持されている。

## 落とし穴

### 1. DRY至上主義による過度な抽象化

重複を恐れるあまり、わずかな類似点で共通化を行うと、汎用的すぎて理解困難な抽象が生まれる。「この共通関数は何をしているのか」を理解するのに、全ての呼び出し元のコンテキストを把握しなければならなくなる。

### 2. レイヤーをまたぐDRYの危険

フロントエンドとバックエンドで[[バリデーション]]ロジックを共有ライブラリにまとめると、バリデーションの変更が両方のデプロイを強制する。フロントのバリデーションはUX目的、バックのバリデーションはセキュリティ目的であり、[[関心の分離]]の観点からは重複を許容すべき場合がある。

### 3. DRYはコードだけの話ではない

DRY の「知識」にはドキュメント、設定ファイル、データベーススキーマも含まれる。コードとドキュメントで同じ情報を二重管理していれば、それもDRY違反。コードから自動生成できるドキュメント（APIドキュメント等）は自動生成すべき。

## 参考リソース

- *The Pragmatic Programmer* — Andy Hunt, Dave Thomas（DRY原則の原典）
- *A Philosophy of Software Design* — John Ousterhout（「浅いモジュール」の問題として過度な共通化を批判）
- Sandi Metz "The Wrong Abstraction" — sandimetz.com（「重複は間違った抽象化より安い」という名言）
