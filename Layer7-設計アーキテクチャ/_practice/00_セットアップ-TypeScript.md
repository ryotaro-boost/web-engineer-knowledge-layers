---
layer: 7
type: practice-setup
language: TypeScript
created: 2026-04-29
---

# 演習セットアップ — TypeScript

> **目的:** [[Layer7-設計アーキテクチャ/_practice/_index|実践演習]]を TypeScript で取り組むための、最小かつ確実に動く scratch project の構築手順。一度作れば 01〜03 の全演習で再利用できる。

## 前提

- Node.js 20 以降がインストール済み（`node --version`）
- Docker が動く環境（Postgres 起動用）
- 任意のエディタ（VS Code 推奨）

## 1. プロジェクト初期化

各演習ごとに独立したディレクトリで作業する想定。最初の演習用ディレクトリを作る:

```bash
mkdir practice-01-user-registration && cd practice-01-user-registration
npm init -y
```

## 2. 依存パッケージのインストール

```bash
# ランタイム依存
npm i express pg bcrypt

# 開発依存（TypeScript 実行・型定義）
npm i -D typescript tsx @types/express @types/pg @types/bcrypt @types/node
```

各パッケージの役割:

| パッケージ | 役割 |
|----------|------|
| `express` | Web フレームワーク。HTTP ハンドラを書くため |
| `pg` | Postgres クライアント |
| `bcrypt` | パスワードハッシュ化 |
| `tsx` | `.ts` ファイルを直接実行する（TS のコンパイル + 実行を1コマンドで） |
| `@types/*` | TypeScript の型定義 |

> **`tsx` を選ぶ理由:** `ts-node` より起動が速く、`esbuild` ベースで ESM/CJS を意識せず動く。本番ではコンパイルしてから `node` で動かすが、練習中は `tsx` で十分。

## 3. TypeScript 設定

```bash
npx tsc --init --target es2022 --module nodenext --moduleResolution nodenext --strict
```

生成された `tsconfig.json` はそのまま使える。`strict: true` が入っていることだけ確認する（型の緩さを避けるため）。

## 4. Postgres を Docker で起動

```bash
docker run -d --name practice-pg \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=practice \
  postgres:16
```

起動確認:

```bash
docker ps
# practice-pg が Up になっていれば OK
```

## 5. テーブル作成

各演習に必要なテーブルは演習ファイル側に記載されている。**演習 01 用のスキーマは下記:**

```bash
docker exec -i practice-pg psql -U postgres -d practice <<'EOF'
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  name TEXT NOT NULL
);
EOF
```

> **演習 02 以降のスキーマ:** 各演習ファイルの「お題」セクションに `CREATE TABLE` を記載する。新しい演習に取り組むときは新しいテーブルを追加するか、別データベースに切り替える。

## 6. 環境変数

`DATABASE_URL` を渡す方法は 2 通り:

### 方法 A: 起動時にインライン指定（手軽）

```bash
DATABASE_URL=postgres://postgres:test@localhost:5432/practice npx tsx src/server.ts
```

### 方法 B: `.env` ファイル + dotenv（複数の変数があるとき）

```bash
npm i -D dotenv
```

`.env`:

```
DATABASE_URL=postgres://postgres:test@localhost:5432/practice
```

`src/server.ts` の冒頭に追加:

```typescript
import "dotenv/config";
```

`.gitignore` に `.env` を追加するのを忘れない。

## 7. 推奨ディレクトリ構成（refactor 後）

scratch project のルート構成:

```
practice-01-user-registration/
├── package.json
├── tsconfig.json
├── .env                      # 任意
├── src/
│   ├── server.ts             # bad example のスタート地点
│   ├── Domain/               # Onion で refactor したら作る
│   ├── Application/
│   ├── Infrastructure/
│   └── ...
└── node_modules/
```

最初は `src/server.ts` 1 ファイルだけ。Onion / Clean に refactor していく過程でディレクトリが分化していく。

## 8. Git で段階管理

refactor の各段階を branch に保存しておくと、後から差分で見比べられる:

```bash
git init
git add .
git commit -m "bad-example: all-in-one handler"
git branch bad-example

# Onion で refactor したら
git checkout -b onion
# ... refactor 作業 ...
git commit -am "onion: extract Domain / Application / Infrastructure"

# Clean で書き直すために bad-example から分岐
git checkout bad-example
git checkout -b clean
# ... refactor 作業 ...
git commit -am "clean: extract UseCases / adapters / domain"
```

これで `git diff onion clean` で2つのアーキテクチャの構造差を直接観察できる。

## 9. 動作確認の典型コマンド

各演習で使う `curl` テンプレート:

```bash
# 成功系（演習 01 の例）
curl -i -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"email":"a@b.c","password":"12345678","name":"A"}'

# JSON のレスポンスを整形して見たい場合
curl -s -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{...}' | jq
```

## 10. 後片付け

演習が一段落したら Postgres を停止:

```bash
docker stop practice-pg && docker rm practice-pg
```

データを保持したい場合は `docker stop` だけ実行し、次回は `docker start practice-pg` で再開できる。

## トラブルシューティング

### `tsx: command not found`

`npm i -D tsx` が成功していない可能性。`node_modules/.bin/tsx` の存在を確認するか、`npx tsx` で実行する。

### `ECONNREFUSED 127.0.0.1:5432`

Postgres が起動していない。`docker ps` で確認し、停止していたら `docker start practice-pg`。

### `relation "users" does not exist`

テーブル作成（手順 5）を飛ばしている可能性。再度 `psql` で `\dt` を実行して確認する。

### bcrypt のネイティブビルドエラー（Apple Silicon Mac 等）

bcrypt は C++ ビルドが必要なため、環境によって失敗することがある。代替として純 JS 実装の `bcryptjs` を使ってもよい:

```bash
npm uninstall bcrypt @types/bcrypt
npm i bcryptjs
npm i -D @types/bcryptjs
```

import を `import * as bcrypt from "bcryptjs"` に変更すれば API は互換。

## 関連リソース

- [[Layer7-設計アーキテクチャ/_practice/_index|演習一覧と進め方]]
- [[Dockerイメージ]] — Postgres コンテナの仕組みを理解したいとき
- [[コネクションプール]] — 演習中に「なぜ Pool を使うのか」が気になったら
