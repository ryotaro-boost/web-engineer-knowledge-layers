---
layer: 7
parent: "[[SOLID原則]]"
type: detail
created: 2026-03-30
---

# コンポジション over 継承（Composition over Inheritance）

> **一言で言うと:** 機能の再利用には継承よりもコンポジション（委譲）を優先せよ。継承は最も強い結合であり、親クラスの変更が全ての子クラスに波及する。

## 概念

GoF（Gang of Four）の *Design Patterns*（1994年）で提唱された原則:

> Favor object composition over class inheritance.

**継承（Inheritance）** は「is-a」関係を表現し、親クラスの実装をそのまま引き継ぐ。**コンポジション（Composition）** は「has-a」関係を表現し、別のオブジェクトに処理を委譲する。

## なぜ継承が問題になるのか

### 1. 強い結合

継承は最も強い結合形態。子クラスは親クラスの public/protected メンバー全てに依存する。親クラスの内部実装を変更するだけで、全ての子クラスが影響を受ける可能性がある。

### 2. 脆い基底クラス問題（Fragile Base Class Problem）

親クラスに安全に見える変更を加えても、子クラスがオーバーライドしたメソッドとの相互作用で予期しない挙動が発生する。

```typescript
// 親クラス
class Collection {
  protected items: string[] = [];

  add(item: string) {
    this.items.push(item);
  }

  addAll(items: string[]) {
    for (const item of items) {
      this.add(item); // 内部で add() を呼んでいる
    }
  }
}

// 子クラス: 追加回数をカウントしたい
class CountingCollection extends Collection {
  count = 0;

  add(item: string) {
    this.count++;
    super.add(item);
  }

  addAll(items: string[]) {
    this.count += items.length;
    super.addAll(items); // ← 内部で this.add() が呼ばれ、count が二重加算される！
  }
}

const c = new CountingCollection();
c.addAll(['a', 'b', 'c']);
console.log(c.count); // 期待: 3, 実際: 6
```

この問題は親クラスの実装詳細（`addAll` が内部で `add` を呼ぶ）に子クラスが依存していることで起きる。親クラスの実装を変更すると子クラスが壊れるため、[[SOLID原則]]のリスコフの置換原則（LSP）違反にもつながる。

### 3. 多重継承の不在

多くの言語（Java, PHP, TypeScript, C#）は多重継承をサポートしない。「ログ機能」と「キャッシュ機能」の両方を継承で取り込むことはできない。コンポジションならば任意の数の機能を組み合わせられる。

## 具体例

### 継承 → コンポジションへのリファクタリング

```typescript
// ❌ 継承: 「ログ付きリポジトリ」を継承で実現
class UserRepository {
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

class LoggingUserRepository extends UserRepository {
  async findById(id: string): Promise<User | null> {
    console.log(`Finding user: ${id}`);
    const user = await super.findById(id);
    console.log(`Found: ${user ? 'yes' : 'no'}`);
    return user;
  }
}

// UserRepository の実装変更が LoggingUserRepository を壊す可能性あり
// CachingUserRepository も作りたい → 多重継承が必要になる

// ✅ コンポジション: 機能を独立したオブジェクトとして組み合わせる
interface UserRepository {
  findById(id: string): Promise<User | null>;
}

class PostgresUserRepository implements UserRepository {
  async findById(id: string): Promise<User | null> {
    return db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

class LoggingRepository implements UserRepository {
  constructor(private inner: UserRepository) {}

  async findById(id: string): Promise<User | null> {
    console.log(`Finding user: ${id}`);
    const user = await this.inner.findById(id);
    console.log(`Found: ${user ? 'yes' : 'no'}`);
    return user;
  }
}

class CachingRepository implements UserRepository {
  private cache = new Map<string, User>();
  constructor(private inner: UserRepository) {}

  async findById(id: string): Promise<User | null> {
    if (this.cache.has(id)) return this.cache.get(id)!;
    const user = await this.inner.findById(id);
    if (user) this.cache.set(id, user);
    return user;
  }
}

// 自由に組み合わせ可能（デコレータパターン）
const repo = new LoggingRepository(
  new CachingRepository(
    new PostgresUserRepository()
  )
);
```

### React における実践

React ではクラスコンポーネントの継承よりも、コンポジションが公式に推奨されている。

```tsx
// ❌ 継承: React では非推奨
class SpecialButton extends Button {
  render() {
    return <button className="special">{super.render()}</button>;
  }
}

// ✅ コンポジション: children や props で合成
function Button({ variant, children, ...props }: ButtonProps) {
  return (
    <button className={`btn btn-${variant}`} {...props}>
      {children}
    </button>
  );
}

// 使用側で自由に合成
<Button variant="primary">
  <Icon name="save" /> 保存
</Button>
```

### PHP/Laravel での実践: Trait によるコンポジション

```php
// PHP では Trait がコンポジションの軽量な実現手段
trait HasTimestamps {
    public function getCreatedAt(): DateTimeInterface { /* ... */ }
    public function touch(): void { /* ... */ }
}

trait SoftDeletes {
    public function softDelete(): void { /* ... */ }
    public function restore(): void { /* ... */ }
}

// 継承なしで複数の機能を合成
class User extends Model {
    use HasTimestamps, SoftDeletes;
}
```

## 継承が適切な場面

コンポジションを優先すべきだが、継承が正当化される場面もある:

1. **フレームワークが要求する場合** — `React.Component`、Laravel の `Controller extends BaseController` など。フレームワークの設計に従う
2. **真の is-a 関係** — `HttpError extends Error` のように、振る舞いレベルで完全に互換性がある場合
3. **テンプレートメソッドパターン** — アルゴリズムの骨格を親クラスで定義し、一部のステップだけ子クラスでカスタマイズする。ただし深い継承階層は避ける（2階層まで）

## 判断のフローチャート

```mermaid
flowchart TD
    Q1{"コードの再利用が<br/>必要？"} -->|"Yes"| Q2{"is-a 関係が<br/>成立する？"}
    Q1 -->|"No"| NONE["再利用不要"]
    Q2 -->|"No"| COMP["コンポジション"]
    Q2 -->|"Yes"| Q3{"親クラスの全メソッドが<br/>子クラスで意味を持つ？<br/>（LSP）"}
    Q3 -->|"No"| COMP
    Q3 -->|"Yes"| Q4{"継承階層は<br/>2階層以内？"}
    Q4 -->|"No"| COMP
    Q4 -->|"Yes"| INHERIT["継承も検討可"]

    style COMP fill:#dfd,stroke:#333
    style INHERIT fill:#ffd,stroke:#333
```

## 落とし穴

### 1. コンポジションの過剰な適用

「継承は全て悪」と解釈して、フレームワークの規約に反してまで回避するのは行き過ぎ。また、`Error` や `Event` のような薄い継承（フィールド追加程度）まで避ける必要はない。

### 2. 委譲コードのボイラープレート

コンポジションでは委譲のためのメソッドを書く必要がある。TypeScript のようにインターフェースの自動実装がない言語では、ラッパーメソッドが増えることがある。これはトレードオフとして受け入れるか、Proxy パターンで軽減する。

## 参考リソース

- *Design Patterns* — GoF（「継承よりコンポジション」の原典）
- *Effective Java* (3rd Edition) — Joshua Bloch（Item 18: "Favor composition over inheritance" の詳細な解説）
- React 公式ドキュメント "Composition vs Inheritance" — react.dev
