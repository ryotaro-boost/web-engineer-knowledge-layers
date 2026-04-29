---
layer: 7
type: practice-setup
language: PHP
created: 2026-04-29
---

# 演習セットアップ — PHP（Slim 4）

> **目的:** [[Layer7-設計アーキテクチャ/_practice/_index|実践演習]]を PHP で取り組むための、最小かつ確実に動く scratch project の構築手順。

## なぜ Laravel ではなく Slim 4 を使うのか

[[クリーンアーキテクチャ]] の detail ドキュメントでは Laravel を例示しているが、**練習用のフレームワークとしては Slim 4 を推奨**する。理由:

| 観点 | Laravel | Slim 4 |
|------|---------|--------|
| Express との対応関係 | 弱い（Service Container, FormRequest, Eloquent 等の Laravel 流儀） | 強い（`$app->post('/users', fn($req, $res) => ...)` の構造が Express とそっくり） |
| 学習対象との比率 | フレームワーク 60% / アーキテクチャ 40% | フレームワーク 10% / アーキテクチャ 90% |
| Eloquent の Active Record 性 | Domain と Infra の境界が曖昧になりがち | PDO で素直に書け、境界を引きやすい |

Laravel は実プロダクトでは選ぶべき選択肢だが、**「アーキテクチャを書いて学ぶ」**という今回の目的では Slim の方が論点が見えやすい。慣れたら同じ演習を Laravel で書き直すのは良い stretch goal。

## 前提

- PHP 8.2 以降がインストール済み（`php --version`）
- Composer がインストール済み（`composer --version`）
- Docker が動く環境（Postgres 起動用）
- PHP の `pdo_pgsql` 拡張が有効（`php -m | grep pdo_pgsql`）

## 1. プロジェクト初期化

```bash
mkdir practice-01-user-registration && cd practice-01-user-registration
```

プロジェクトルートに **`composer.json`** を直接作成する（`composer init` の対話モードを通らずにテンプレートを書いた方が学習者には早い）:

```json
{
  "name": "practice/user-registration",
  "type": "project",
  "require": {},
  "autoload": {
    "psr-4": { "App\\": "src/" }
  }
}
```

**`autoload.psr-4` の意味:** `src/Domain/User.php` の中で `namespace App\Domain;` と書けば、`use App\Domain\User;` で読み込める仕組み（PSR-4: 名前空間の階層 → ディレクトリの階層をマップ）。次の `composer require` で `vendor/autoload.php` が生成され、これを `require` 1 行で全クラスが lazy load される。

## 2. 依存パッケージのインストール

```bash
composer require slim/slim:"^4.15" slim/psr7:"^1.7" ramsey/uuid:"^4.7" vlucas/phpdotenv:"^5.6"
```

各パッケージの役割:

| パッケージ | 役割 |
|----------|------|
| `slim/slim` | マイクロ Web フレームワーク本体 |
| `slim/psr7` | PSR-7 メッセージ実装（`Request` `Response`） |
| `ramsey/uuid` | UUID v4 生成 |
| `vlucas/phpdotenv` | `.env` ファイルから環境変数を読み込む |

> **PHP 標準で使えるもの:** `password_hash()` / `password_verify()` で bcrypt が使えるため、外部パッケージは不要。Postgres 接続も `pdo_pgsql`（標準拡張）で十分。

## 3. Postgres を Docker で起動

```bash
docker run -d --name practice-pg \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=practice \
  postgres:16
```

## 4. テーブル作成

演習 01 用のスキーマ:

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

## 5. 環境変数

`.env` をプロジェクトルートに作成:

```
DATABASE_URL=pgsql:host=localhost;port=5432;dbname=practice
DATABASE_USER=postgres
DATABASE_PASSWORD=test
```

`.gitignore` に `.env` を追加するのを忘れない。

エントリポイント側で読み込む:

```php
// public/index.php または src/server.php
require __DIR__ . '/../vendor/autoload.php';

$dotenv = Dotenv\Dotenv::createImmutable(__DIR__ . '/..');
$dotenv->load();

$pdo = new PDO(
    $_ENV['DATABASE_URL'],
    $_ENV['DATABASE_USER'],
    $_ENV['DATABASE_PASSWORD'],
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION]
);
```

## 6. 推奨ディレクトリ構成（refactor 後）

```
practice-01-user-registration/
├── composer.json
├── composer.lock
├── .env
├── public/
│   └── index.php           # Slim のエントリポイント（Composition Root もここ）
├── src/                    # PSR-4 で App\ にマップ
│   ├── Domain/             # Onion で refactor したら作る
│   ├── Application/
│   └── Infrastructure/
└── vendor/
```

最初は `public/index.php` 1 ファイルだけ。Onion / Clean に refactor していく過程で `src/` 配下が分化していく。

> **`public/` に置く理由:** PHP の慣習で「Web からアクセス可能なディレクトリ」を `public/` に分け、`composer.json` 等の設定ファイルや `.env` は外側に置くことで、誤って公開されるリスクを回避する。

## 7. 起動

PHP のビルトイン Web サーバーを使う（本番では Nginx + PHP-FPM などにするが、練習では十分）:

```bash
php -S localhost:3000 -t public
```

`-t public` でドキュメントルートを `public/` に指定。`public/index.php` が全リクエストのフロントコントローラになる。

## 8. Git で段階管理

```bash
git init
echo "vendor/" > .gitignore
echo ".env" >> .gitignore
git add .
git commit -m "bad-example: all-in-one handler"
git branch bad-example

git checkout -b onion
# refactor 作業 ...
git commit -am "onion: extract Domain / Application / Infrastructure"

git checkout bad-example
git checkout -b clean
# refactor 作業 ...
git commit -am "clean: extract UseCases / adapters / domain"
```

## 9. 動作確認の典型コマンド

TypeScript 版と完全に同じ:

```bash
# 成功（演習 01 の例）
curl -i -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"email":"a@b.c","password":"12345678","name":"A"}'
```

## 10. 後片付け

```bash
docker stop practice-pg && docker rm practice-pg
```

## トラブルシューティング

### `could not find driver` で PDO が失敗する

`pdo_pgsql` 拡張が無効になっている可能性。`php -m | grep pdo_pgsql` で確認し、出ない場合は `php.ini` で `extension=pdo_pgsql` のコメントアウトを外す。

macOS の Homebrew PHP なら標準で有効になっているはず。Docker で動かす場合は `php:8.2-cli` イメージに `docker-php-ext-install pdo pdo_pgsql` を追加する。

### Slim の Response 書き込みが反映されない

PSR-7 の `Response` は **immutable** なので `withStatus(201)` の戻り値を `return` し忘れると無効。下記のような書き方になる:

```php
// ❌ 書き込みは反映されるがステータスが変わらない
$res->getBody()->write(json_encode($data));
$res->withStatus(201);  // 戻り値を捨てている
return $res;

// ✅ 戻り値で上書きしながら return
$res->getBody()->write(json_encode($data));
return $res->withStatus(201)->withHeader('Content-Type', 'application/json');
```

### Composer のメモリ不足

`composer install` が `Allowed memory size exhausted` で落ちる場合:

```bash
COMPOSER_MEMORY_LIMIT=-1 composer install
```

### `password_hash()` の cost を bcrypt の TS / Go 例（cost=10）と揃えたい

`password_hash($plain, PASSWORD_BCRYPT)` のデフォルト cost は **PHP のバージョンに依存する**:

| PHP バージョン | デフォルト cost |
|--------------|---------------|
| PHP 8.3 以前 | **10** |
| PHP 8.4 以降 | **12** |

つまり:

- PHP 8.3 以前で動かしているなら何も指定しなくても TS / Go と同じ cost=10 になる
- PHP 8.4 以降では明示的に指定しないとずれる

```php
// PHP 8.4+ で TS/Go と揃えたい場合
$hash = password_hash($plain, PASSWORD_BCRYPT, ['cost' => 10]);
```

実プロダクトでは cost=12 以上が推奨される（PHP 8.4 が引き上げた背景もここ）。練習の動作確認上は cost を意識する必要は薄いが、3 言語間でハッシュ計算時間を揃えたい時の参考。

## 関連リソース

- [[Layer7-設計アーキテクチャ/_practice/_index|演習一覧と進め方]]
- [[Layer7-設計アーキテクチャ/_practice/00_セットアップ-TypeScript|TypeScript 版セットアップ]] — 比較対象
- [[Webサーバーとランタイムのリクエスト処理モデル]] — `php -S` のビルトインサーバーが本番運用に向かない理由を理解する補助
- [[コネクションプール]] — PDO の接続管理を考える際の参考
- [[トランザクション]] — 演習 02 以降で使う
