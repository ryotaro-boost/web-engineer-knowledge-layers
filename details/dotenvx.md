---
layer: 7
parent: "[[CI-CD]]"
type: detail
created: 2026-06-30
---

# dotenvx（暗号化対応の .env ローダー）

> **一言で言うと:** `.env` ファイルを公開鍵で暗号化し、暗号化済みの状態のまま git にコミットできるようにするシークレット管理ツール。「秘密情報はリポジトリに入れない」という従来の鉄則を、「**暗号化されていれば入れてよい**」へと反転させ、復号できる秘密鍵（`DOTENV_PRIVATE_KEY`）だけを安全に配ればよい状態にする。

## 前提: そもそも .env とは何か

多くのアプリケーションは、データベース接続文字列や API キーといった**環境ごとに変わる設定値**を、ソースコードに直書きせず**環境変数（environment variable）**として外から注入する。これは [The Twelve-Factor App](https://12factor.net/config) が「設定はコードから分離せよ」と定めた原則で、[[CI-CD]] の「環境ごとの設定の散乱」を防ぐ基本でもある。

`.env` は、その環境変数を `KEY=VALUE` 形式で1ファイルにまとめたもの。`dotenv` というライブラリ（2013年 Node.js 向けに登場、現在はほぼ全言語に移植）がこのファイルを読み、プロセスの環境変数（`process.env` / `os.environ` 等）に流し込む。

```bash
# .env （平文）
DATABASE_URL="postgres://user:pass@localhost:5432/app"
STRIPE_SECRET_KEY="sk_live_51Abc..."
```

## 従来の平文 .env が抱える問題

平文 `.env` の運用は、次の構造的なジレンマを抱える。

```mermaid
flowchart TB
    SECRET["秘密情報を含む .env"]
    SECRET -->|"git に入れる"| RISK["❌ 履歴に永久に残る<br/>漏洩は確定的"]
    SECRET -->|".gitignore で除外"| GAP["⚠️ リポジトリに存在しない"]
    GAP --> Q1["新メンバーはどこから値を得る?"]
    GAP --> Q2["本番の値は誰がどう配る?"]
    GAP --> Q3[".env.example と本物がズレる"]
    Q1 & Q2 & Q3 --> MANUAL["Slack / 1Password / 口頭で<br/>手作業共有 → 属人化・ミス"]

    style RISK fill:#fdd,stroke:#c00
    style MANUAL fill:#ffd,stroke:#c80
```

- **コミットすれば漏洩する** — 一度 git 履歴に入った秘密情報は、後で削除しても履歴の書き換え（`git filter-repo` 等）が必要で、フォークやクローンには残り続ける。「漏洩は確定的」として扱うのが安全側。
- **`.gitignore` で除外すると共有できない** — リポジトリに存在しないため、新メンバーは値を入手する経路がなく、Slack や 1Password での手渡しに頼る。これが「環境ごとの設定の散乱」の温床になる。
- **`.env.example` と本物がズレる** — 「変数名だけ書いたサンプル」を別途維持する慣習があるが、本物の `.env` に変数を足したとき example の更新を忘れ、他メンバーの環境が壊れる。

dotenvx はこのジレンマを「**暗号化した .env なら git に入れてよい**」という発想で解消する。

## dotenvx の暗号化モデル

dotenvx は**公開鍵暗号（public-key cryptography）**を使う。鍵が2種類に分かれているのが核心である。

| 鍵 | 役割 | 置き場所 |
|---|---|---|
| `DOTENV_PUBLIC_KEY` | **暗号化用**。これで値を暗号化する。漏れても復号はできない | `.env` ファイル冒頭に平文で記録 → **git にコミットしてよい** |
| `DOTENV_PRIVATE_KEY` | **復号用**。実行時にこれで値を戻す。これが漏れると全て復号される | `.env.keys` ファイル（**git 管理外**）/ 本番では環境変数・シークレットストアに格納 |

```mermaid
sequenceDiagram
    participant Dev as 開発者
    participant Repo as git リポジトリ
    participant Prod as 本番サーバー

    Dev->>Dev: dotenvx set KEY value<br/>(PUBLIC_KEY で暗号化)
    Dev->>Repo: 暗号化済み .env を commit ✅
    Note over Repo: 値は encrypted:BNGD... の形<br/>漏れても復号不可
    Repo->>Prod: clone / pull
    Prod->>Prod: DOTENV_PRIVATE_KEY を環境変数で渡す
    Prod->>Prod: dotenvx run -- <app><br/>(PRIVATE_KEY で復号 → 注入)
```

暗号化方式は **ECIES（Elliptic Curve Integrated Encryption Scheme、楕円曲線 secp256k1 ベース）**。正確には、値ごとに使い捨ての **AES-256** 鍵（ephemeral key）を生成して値を暗号化し、その AES 鍵を secp256k1 の公開鍵で包む**ハイブリッド暗号（hybrid encryption）**である。これにより [[暗号アルゴリズム]] でいう共通鍵暗号の「速さ」と公開鍵暗号の「鍵配送のしやすさ（暗号化する側に秘密鍵を渡さなくてよい）」を両取りしている。値ごとに暗号化されるため、`.env` の中で「どの変数があるか（キー名）」は読めるが「値」は読めない。これにより、レビュアーは差分で「`STRIPE_SECRET_KEY` が追加された」とは分かるが、値そのものは見えない。

### 暗号化後の .env はこう見える

```bash
#/-------------------[DOTENV_PUBLIC_KEY]--------------------/
#/            public-key encryption for .env files          /
#/-------------------------------------------------/
DOTENV_PUBLIC_KEY="03eaf2ec07a04b8c3..."

# 値は暗号化される。キー名は平文のまま残る
DATABASE_URL="encrypted:BNGD5x9c8Hk2...（長い暗号文）"
STRIPE_SECRET_KEY="encrypted:A8f0Tz...（長い暗号文）"
```

## 基本操作

```bash
# 1. インストール（npm 不要のスタンドアロンバイナリも提供）
npm install -g @dotenvx/dotenvx
#   または curl -fsS https://dotenvx.sh | sh

# 2. 平文 .env を暗号化する（PUBLIC/PRIVATE 鍵ペアを自動生成）
#    → .env が暗号化され、.env.keys に PRIVATE_KEY が書き出される
dotenvx encrypt

# 3. 新しい秘密情報を「暗号化した状態で」追加する
dotenvx set STRIPE_SECRET_KEY "sk_live_xxx"

# 4. アプリ起動時に復号して環境変数を注入する
dotenvx run -- node server.js
dotenvx run -- npm start
```

ポイントは `dotenvx run -- <コマンド>`。**`<コマンド>` を起動する直前に `.env` を復号してそのプロセスの環境変数へ流し込む**ラッパーとして動く。アプリ側のコードは通常の `process.env.DATABASE_URL` を読むだけでよく、**ソースコードに dotenvx 依存を一切書かなくてよい**（言語非依存）。これが従来の `require('dotenv').config()` をコードに埋め込む方式との大きな違いである。

### 複数環境の使い分け

```bash
# 環境ごとにファイルを分け、それぞれ別の鍵で暗号化できる
dotenvx set DATABASE_URL "..." -f .env.production
dotenvx set DATABASE_URL "..." -f .env.staging

# 本番の値を本番鍵で復号して起動
dotenvx run -f .env.production -- node server.js
```

`.env.keys` には `DOTENV_PRIVATE_KEY_PRODUCTION`、`DOTENV_PRIVATE_KEY_STAGING` のように環境別の秘密鍵がまとまる。本番サーバーには**その環境の秘密鍵1本だけ**を環境変数として渡せばよく、他環境の値は復号できない（[[最小権限の原則]] の実践）。

## どこで使うべきか — Vault / Secrets Manager との使い分け

dotenvx は「専用シークレットストア」の**完全な代替ではなく、軽量な選択肢**である。判断軸は「秘密鍵をどこに置き、どう守るか」に集約される。

| 観点 | dotenvx | HashiCorp Vault / AWS Secrets Manager |
|---|---|---|
| **保管場所** | 暗号化された .env を git に同居 | 専用の集中ストアに保管 |
| **必要な信頼** | `DOTENV_PRIVATE_KEY` を配る経路の安全性 | ストアへのアクセス制御（IAM / ポリシー） |
| **動的な機能** | なし（静的な値） | **自動ローテーション・動的シークレット・失効・監査ログ** |
| **アクセス監査** | git 履歴のみ（誰が復号したかは追えない） | 「誰がいつどの秘密を読んだか」を記録 |
| **導入コスト** | ほぼゼロ（CLI 1本） | サーバー運用 / 料金 / IAM 設計が必要 |
| **向く規模** | 個人〜小中チーム、OSS、シンプルな本番 | 規模が大きい / コンプライアンス要件あり / 秘密の数が多い |

> [!info] 用語ミニ辞典
> - **シークレットローテーション（secret rotation）**: API キーやパスワードを定期的に新しい値へ入れ替えること。漏洩時の被害を「次の更新まで」に限定する。dotenvx は手動更新のみで、自動ローテーション機構を持たない。
> - **動的シークレット（dynamic secrets）**: Vault などが要求時に都度生成し短時間で失効させる使い捨ての資格情報。長命な静的キーを置かないことで漏洩リスクを下げる。

**判断のしかた:** 「秘密の数が少なく・ローテーション頻度が低く・監査要件がない」なら dotenvx で十分。「コンプライアンス監査が必要」「秘密が数百規模」「自動失効が要る」なら Vault / Secrets Manager を選ぶ。両者は排他ではなく、**`DOTENV_PRIVATE_KEY` 自体を Secrets Manager に置く**といった併用も成立する。

## よくある落とし穴

- **`.env.keys` をコミットしてしまう** — 最大の事故。秘密鍵が入った `.env.keys` を git に入れると、暗号化の意味が完全に消える。`dotenvx encrypt` は `.gitignore` に `.env.keys` を自動追記するが、既に追跡済みのファイルは ignore が効かないため `git rm --cached .env.keys` が要る。コミット前に `git status` で必ず確認する。
- **「キー名は平文」を忘れる** — 暗号化されるのは**値だけ**。変数名（`PAYMENT_GATEWAY_PASSWORD` 等）から秘密の存在が推測される点は変わらない。命名で機密内容そのものを露出させない配慮は依然必要。
- **本番に秘密鍵を安全に渡せていない** — dotenvx は「git に暗号文を置く」問題は解くが、「本番へ `DOTENV_PRIVATE_KEY` をどう届けるか」は別問題として残る。これを平文でデプロイスクリプトに書けば本末転倒。CI/CD のシークレット機能（GitHub Actions Secrets 等）経由で渡す。
- **暗号化前の平文をコミット済み** — `dotenvx encrypt` する**前**に平文 `.env` を1度でもコミットしていれば、その値は履歴に残っている。暗号化は「これ以降」を守るだけで、過去の漏洩は別途キーの失効・再発行で対処する。

## AI実装のアンチパターン

| アンチパターン | なぜ起きるか | あるべき姿 |
|---|---|---|
| 平文 `.env` への逆戻り | AI は学習データに多い「`dotenv.config()` で平文 .env を読む」定番コードを書きがち。dotenvx 導入後もこのコードを足すと暗号化が迂回される | アプリ側の `require('dotenv')` を消し、起動を `dotenvx run --` に寄せる |
| `.env.keys` をサンプルに含める | 「動く例」を優先するため、秘密鍵をコードブロックやコミット例に書いてしまう | 鍵は必ずプレースホルダ。`.gitignore` 追記と `git rm --cached` をセットで提示させる |
| 秘密鍵をデプロイ YAML に直書き | `DOTENV_PRIVATE_KEY: "値"` のようにパイプライン定義へリテラルで埋める | `${{ secrets.DOTENV_PRIVATE_KEY }}` 経由のみ。リテラル禁止を制約として明示 |
| Vault が要る場面でも dotenvx を推す | 「手軽さ」を最適化し、監査・ローテーション要件を見落とす | 規模・コンプライアンス要件を確認し、必要なら Secrets Manager を選ぶ判断を人間に残す |

→ [[_anti-patterns/_index|AIアンチパターン索引]]

## 関連トピック

- [[CI-CD]] — 「シークレット管理」「環境ごとの設定の散乱」の解決手段の一つ。本番への秘密鍵配布は CI/CD のシークレット機能と組み合わせる
- [[最小権限の原則]] — 環境ごとに秘密鍵を分け、本番には本番鍵だけを渡す設計の根拠
- [[暗号アルゴリズム]] — dotenvx が使う公開鍵暗号（ECIES / 楕円曲線）の基礎
- [[サプライチェーンセキュリティ]] — `.env` がパッケージに混入する情報漏洩リスクとも関連

## 参考リソース

- [dotenvx 公式サイト](https://dotenvx.com/)
- [dotenvx GitHub リポジトリ](https://github.com/dotenvx/dotenvx)
- [The Twelve-Factor App — Config](https://12factor.net/config)
