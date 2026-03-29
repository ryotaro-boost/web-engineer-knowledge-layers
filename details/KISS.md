---
layer: 7
parent: "[[関心の分離]]"
type: detail
created: 2026-03-30
---

# KISS（Keep It Simple, Stupid）

> **一言で言うと:** 設計・実装において最もシンプルな解決策を選べ。[[YAGNI]]が「いつ作るか」の指針なら、KISSは「どう作るか」の指針。

## 概念

KISS は米国海軍の航空技術者 Kelly Johnson が提唱したとされる設計原則で、元々は航空機の設計哲学だった。ソフトウェアにおいては:

> ほとんどのシステムは、複雑にするよりもシンプルに保った方がうまく動く。

シンプルさとは「手抜き」ではない。**必要な機能を最も少ないコード・最も分かりやすい構造で実現すること**がシンプルさの本質。複雑なコードを書くのは簡単で、シンプルなコードを書く方が難しい。

## なぜ複雑さが生まれるか

複雑さは意図的に追加されるのではなく、以下の力によって無意識に侵入してくる:

1. **「念のため」の防御** — 起きない可能性が高いケースへの対処コードを追加する
2. **抽象化の誘惑** — 「汎用的にしておけば将来楽だろう」と考えて過度に抽象化する
3. **技術的好奇心** — 新しいパターンやライブラリを使いたいという動機で不必要に導入する
4. **要件の曖昧さ** — 「もしかしたらこの機能も要るかも」と想像で機能を追加する

## 具体例

### 過剰な抽象化

```typescript
// ❌ 複雑: メール送信を「通知戦略」として汎用化
interface NotificationChannel {
  type: string;
  send(recipient: Recipient, content: Content): Promise<Result>;
}

interface Recipient {
  id: string;
  channels: ChannelPreference[];
}

interface Content {
  template: string;
  variables: Record<string, unknown>;
  locale?: string;
}

interface Result {
  success: boolean;
  channelType: string;
  timestamp: Date;
  retryable: boolean;
}

class NotificationOrchestrator {
  constructor(private channels: Map<string, NotificationChannel>) {}

  async notify(recipient: Recipient, content: Content): Promise<Result[]> {
    const results: Result[] = [];
    for (const pref of recipient.channels) {
      const channel = this.channels.get(pref.type);
      if (channel) {
        results.push(await channel.send(recipient, content));
      }
    }
    return results;
  }
}

// ✅ シンプル: 今必要なのはメール送信だけ
async function sendWelcomeEmail(email: string, name: string): Promise<void> {
  await mailer.send({
    to: email,
    subject: 'ようこそ',
    body: `${name}さん、登録ありがとうございます。`,
  });
}
```

### 過剰な型設計

```typescript
// ❌ 複雑: 汎用的な Result 型
type Result<T, E = Error> =
  | { success: true; data: T; metadata?: Record<string, unknown> }
  | { success: false; error: E; code?: string; retryable?: boolean };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) return { success: false, error: '0で割ることはできません' };
  return { success: true, data: a / b };
}

// ✅ シンプル: 言語の例外機構をそのまま使う
function divide(a: number, b: number): number {
  if (b === 0) throw new Error('0で割ることはできません');
  return a / b;
}
```

### 過剰なファイル分割

```
❌ 複雑: 1つの機能に6ファイル
src/
  users/
    types/
      UserInput.ts
      UserOutput.ts
    interfaces/
      IUserRepository.ts
      IUserService.ts
    UserService.ts
    UserRepository.ts

✅ シンプル: 1つの機能に必要最小限のファイル
src/
  users/
    UserService.ts        # 型定義・リポジトリも含む（分離の必要が出たら分ける）
```

## KISSと他の原則の関係

| 原則 | KISSとの関係 |
|------|-------------|
| [[YAGNI]] | YAGNI = 不要な機能を**作らない**。KISS = 必要な機能を**シンプルに作る**。補完関係 |
| [[DRY原則]] | DRYの過剰適用（偶然の重複の共通化）はKISS違反になる |
| [[SOLID原則]] | SOLIDの過剰適用（全てにインターフェース）はKISS違反。ただしSOLIDを無視した巨大クラスもKISS違反。バランスが重要 |
| [[関心の分離]] | 適切な分離はシンプルさを生む。過剰な分離は複雑さを生む |

## シンプルさの判断基準

コードのシンプルさを判断する実践的な問い:

1. **「これは何をしているか」を30秒で説明できるか？** — 説明に時間がかかるなら複雑すぎる
2. **新しいチームメンバーが理解できるか？** — コンテキストなしで読める方がシンプル
3. **このコードを消して書き直す場合、同じ構造にするか？** — Noなら過剰設計のサイン
4. **間接参照（インターフェース・ファクトリ・委譲）に明確な理由があるか？** — 「将来のため」は理由にならない

## 落とし穴

### 1. 「シンプル」と「簡単」の混同

シンプルとは「概念的な複雑さが低い」こと。簡単とは「慣れていて楽にできる」こと。ORM を使うのは「簡単」だが、発行される SQL を理解していなければ問題が「シンプル」になったわけではない。

### 2. 事前設計の放棄

KISS は「考えずに書け」ではない。複雑な問題をシンプルに解決するには、むしろ**深い理解と設計の時間**が必要。パスカルの「短い手紙を書く時間がなかったので、長い手紙を書きました」と同じ。

### 3. シンプルさの過激な追求

「1ファイルに全部書く」「変数名を1文字にする」はKISSではなく可読性の破壊。シンプルさは**コード量**ではなく**認知的負荷**で測る。

## 参考リソース

- *A Philosophy of Software Design* — John Ousterhout（「複雑さ」を正面から定義し、その管理方法を体系化）
- *Simple Made Easy* — Rich Hickey（StrangeLoop 2011 講演。「Simple」と「Easy」の違いを明確にした名講演）
- *The Art of Unix Programming* — Eric S. Raymond（Unix哲学のKISS原則をソフトウェア設計全般に適用）
