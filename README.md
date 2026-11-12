# openapi-lint

OpenAPI Linter（Spectral / Redocly CLI / vacuum ほか）の調査と、『Web API の設計』のチェックリストをどこまで Lint で自動化できるかの検証。

---

## 1. 結論（TL;DR）

| 観点 | 結論 |
| --- | --- |
| 自動化できる範囲 | 表13-2 の **約半分〜6 割は「デフォルトルール or カスタムルール」で機械検出できる**。残りは「ガイドラインとの整合」「既存要素との一貫性」「ニーズとの一致」など意味の判断で、人間のレビューが必要 |
| 表13-1/13-3/13-4 | ニーズ分析・プロバイダ/コンシューマ視点のレビュー。**ほぼ自動化不可**（OWASP 系ルールでセキュリティの一部だけ拾える） |
| 破壊的変更（表13-2「進化」） | Spectral / Redocly / vacuum では**検出できない**。**oasdiff** などの差分ツールを併用する |
| おすすめ構成 | **ルールは Spectral 形式で書き、vacuum（または Spectral）で実行 ＋ oasdiff で破壊的変更検出 ＋ 残りは人間のレビューチェックリスト** |

書籍も「リントで検証されるのは形式だけで、内容までは検証されない」と述べており、実測もその通りでした。

---

## 2. 主要ツール比較

実際にインストールして動かしたバージョン: Spectral 6.17.1 / Redocly CLI 2.77.0 / vacuum 0.40.0 / oasdiff (latest)

| | **Spectral** | **Redocly CLI** | **vacuum** |
| --- | --- | --- | --- |
| 開発元 | Stoplight（現 SmartBear） | Redocly | pb33f（Dave Shanley） |
| 実装 / 配布 | TypeScript / npm | TypeScript / npm | Go / 単一バイナリ・npm・brew・Docker |
| 対応仕様 | OAS 2.0 / 3.0 / 3.1, AsyncAPI, Arazzo | OAS 2.0 / 3.0 / 3.1, AsyncAPI, Arazzo | OAS 2.0 / 3.0 / 3.1 |
| ルール記述 | YAML/JSON ルールセット（JSONPath + 組込関数）＋ JS カスタム関数 | `redocly.yaml` の **configurable rules（assertions）** ＋ JS プラグイン | **Spectral 形式のルールセットをそのまま使える** ＋ JS カスタム関数 |
| 組込ルールセット | `spectral:oas`, `spectral:asyncapi`, `spectral:arazzo`（OWASP は別パッケージ） | `minimal` / `recommended` / `recommended-strict` / `spec` | `vacuum:oas`（recommended / all）, **`vacuum:owasp` 内蔵** |
| デフォルトの厳しさ | 緩め（ドキュメント系中心） | 中（security / 4xx もチェック） | 中〜厳（description・example・命名・DELETE ボディ等） |
| レポート | text/json/junit/sarif/html 他 | stylish/json/checkstyle/codeclimate/markdown 他 | コンソール、**HTML レポート、TUI ダッシュボード**、JSON、JUnit |
| 付加機能 | — | bundle / split / join / docs プレビュー / 統計 | 高速（大規模スペック向け）、品質スコア表示 |
| 破壊的変更検出 | × | × | ×（同作者の **openapi-changes** で可能） |
| 向いている用途 | 事実上の標準ルールセット形式。既存ルールセット資産が豊富 | ドキュメント生成・バンドルまで含めた一体運用 | Spectral 互換で速く、OWASP まで一発でかけたい場合 |

### 実測で分かった特徴

同じ「悪いスペック」（`examples/invalid.yaml`：書籍の `POST /send-email` 例＋各種違反を仕込んだもの）を各ツールのデフォルト設定で検査した結果:

| 仕込んだ違反 | Spectral `spectral:oas` | Redocly `recommended` | vacuum デフォルト | vacuum `all`+`owasp` |
| --- | :-: | :-: | :-: | :-: |
| パスパラメータ `{accountId}` 未定義 | ✅ | ✅ | ✅ | ✅ |
| operationId 重複 | ✅ | ✅ | ✅ | ✅ |
| 未使用スキーマ | ✅ | ✅ | ✅ | ✅ |
| **未使用セキュリティスキーム** | ❌ | ✅ | ✅ | ✅ |
| **security 未設定のオペレーション** | ❌ | ✅ | ❌ | ✅ |
| **4xx レスポンスなし** | ❌ | ✅ | ❌ | ✅ |
| tags なし / 未定義タグ | ✅ | ❌ | ✅ | ✅ |
| **DELETE にリクエストボディ** | ❌ | ❌ | ✅ | ✅ |
| **プロパティ名 `err_msg` が camelCase でない** | ❌ | ❌ | ✅ | ✅ |
| string に format/maxLength なし、integer に min/max なし | ❌ | ❌ | ❌ | ✅ |
| http（非 TLS）の server URL | ❌ | ❌ | ❌ | ✅ |
| **全プロパティが任意（required なし）** | ❌ | ❌ | ❌ | ❌ |
| **クエリパラメータが object** | ❌ | ❌ | ❌ | ❌ |
| **OAuth スコープの説明が空** | ❌ | ❌ | ❌ | ❌ |
| **パスに動詞（`/send-email`）** | ❌ | ❌ | ❌ | ❌ |
| `msg` が number（名前と型の不整合） | ❌ | ❌ | ❌ | ❌ |

下の 5 行のうち上 4 つは **カスタムルールを書けば 3 ツールとも検出できました**（`examples/custom.spectral.yaml`, `examples/redocly.yaml`）。  
同じ `custom.spectral.yaml` が **Spectral と vacuum の両方でそのまま動く** ことも確認済みです。  
最後の「`msg` が number」のような意味の不整合は、どのツールでも検出できません（人間のレビュー領域）。

---

## 3. 表13-2 API リントチェックリスト × 自動化可否

凡例:
- ◎ … デフォルト（または公開ルールセット）で検出できる
- ○ … カスタムルールで検出できる（実装例あり／容易）
- △ … 一部だけ（存在チェックや簡易ヒューリスティックまで）
- × … 機械では判定できない（人間のレビューが必要）

### 進化

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 破壊的変更の導入 | ◎（別ツール） | **oasdiff `breaking`**（実測: `email` が必須化、`msg` の型変更を error として検出）。pb33f openapi-changes も可。3 つのリンター単体では × |

### API のセキュリティ定義

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 使われている | ◎ | Redocly `no-unused-components`、vacuum `oas3-unused-component`（Spectral の同名ルールは securityScheme を未検出） |
| 有効である | ◎/○ | 構造の妥当性はスキーマ検証で ◎。「ガイドライン準拠」（例: OAuth2 必須、https の tokenUrl）はカスタムで ○ |
| 説明が有効 | ○/× | 「スコープ説明が空でない」はカスタムで ○（実測）。説明の適切さは × |

### API のゴールフロー

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 有効である | × | フロー（ユースケース）の妥当性は人間。Arazzo でワークフローを記述すれば構造リントは可能（Spectral/Redocly）だが意味は × |
| 説明が有効 | △ | description の存在のみ |

### モデル

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 使われている | ◎ | 3 ツールとも未使用コンポーネント検出 |
| 名前が有効 | ○/× | 命名規則（PascalCase、`DTO` 接尾辞禁止など）は `casing` / `pattern` で ○。「分かりやすい名前か」は × |
| データ構造が有効 | △ | ネストの深さ等は JS カスタム関数で可能。適切さは × |
| 説明が有効 | ◎/× | 存在は ◎（vacuum `component-description`、Spectral はカスタム）。正確さは × |

### モデルのプロパティ

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 名前が有効 | ○ | `casing: camel`（vacuum はデフォルト ◎）。`ts` `msg` のような略語は禁止語リスト（`pattern`）で △ |
| 必須ステータスが有効 | ○ | 「properties があるのに required がない」をカスタムで警告（実測、3 ツールとも可） |
| 型とフォーマットが有効 | △ | vacuum OWASP で「string に format/pattern/enum」「integer に format」を ◎。`*At`/`*Date` という名前なら `format: date-time` を要求、などのヒューリスティックは ○。名前と型の意味的整合は × |
| 説明が有効 | ◎/× | description 存在、minimum/maximum・maxLength の有無は ◎（vacuum OWASP）。正確さは × |

### ゴール（オペレーション）

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 分類されている | ◎ | `operation-tags` / `operation-tag-defined`（Spectral, vacuum） |
| パスが有効 | ◎/○ | パスパラメータ整合は ◎（3 ツール）。kebab-case・動詞禁止・複数形などのフォーマットはカスタムで ○（実測） |
| HTTP メソッドが有効 | △ | 「DELETE にボディ」は vacuum ◎。ゴールの内容にふさわしいかは × |
| 成功とエラーに対処している | ◎ | Redocly `operation-4xx-response`、vacuum OWASP（401/429/500 等）。Spectral は OWASP ルールセット追加で対応 |
| セキュア | ◎ | Redocly `security-defined`、vacuum OWASP `owasp-protection-global-unsafe` / `-safe` |

### ゴールのパラメータ

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| 名前が有効 | ○ | `casing` |
| 必須ステータスが有効 | ◎/○ | パスパラメータ必須はスキーマ検証で ◎。作成/置換ボディの全任意はカスタムで ○ |
| 型とフォーマットが有効 | ○ | 「クエリに object 禁止」はカスタムで ○（実測、3 ツールとも可） |
| 説明が有効 | ◎/× | description 存在は ◎（vacuum）、正確さは × |
| 位置が有効 | ◎/○ | DELETE/GET のボディは vacuum ◎。「センシティブ値をクエリに置かない」は OWASP ルールで一部 ◎ |
| 提供可能 | × | 「コンシューマがその値を知っているか／別ゴールから取れるか」はフロー分析が必要で × |

### ゴールのレスポンス

| チェック | 可否 | 手段 |
| --- | :-: | --- |
| HTTP ステータスが有効 | ○ | 許可するステータスコード一覧を `enum`/`pattern` で ○ |
| ボディが有効 | ○ | 「4XX/5XX は標準エラーモデル（例: `application/problem+json` / `#/components/schemas/Error`）を返す」をカスタムで ○ |
| ヘッダーが有効 | ○ | ヘッダー名の命名規則、レート制限ヘッダー必須（vacuum OWASP ◎）など |
| 使われている | × | 返すデータが実際に使われるかは × |
| 成功の説明が有効 | ◎/× | description・スキーマ存在は ◎、正確さは × |
| エラーの説明が有効 | △ | エラーレスポンスに description/スキーマがあるかまで |

### 集計（表13-2・全 31 項目、主たる判定で分類）

| 区分 | 件数の目安 |
| --- | --- |
| デフォルト or 公開ルールセットで検出（◎） | 約 10 |
| カスタムルールで検出（○） | 約 9 |
| 一部のみ（△） | 約 6 |
| 人間のみ（×） | 約 6 |

※ ◎/× のように「存在チェックは自動、内容は人間」に分かれる項目が多い点に注意。

---

## 4. 表13-1 / 13-3 / 13-4 について

| 表 | 自動化 | 補足 |
| --- | --- | --- |
| 13-1 ニーズの検証と分析 | × | 5つのなぜ、ゴールキャンバス等。スペック以前の話 |
| 13-3 プロバイダレビュー | ほぼ × | 「URL にセンシティブなデータを含めない」「API キーをクエリに置かない」「TLS 必須」などは **OWASP ルールセット**（vacuum 内蔵 / `@stoplight/spectral-owasp-ruleset`）で一部検出可。「ニーズと一致」「実装可能」「拡張可能」は × |
| 13-4 コンシューマレビュー | ほぼ × | 「UNIX タイムスタンプではなく ISO 8601」は `integer` かつ名前が `*At`/`*Time` を警告する等のヒューリスティックで △。「aDTO / ts ではなく分かりやすい名前」は禁止語リストで △ |

---

## 5. その他の関連ツール

| ツール | 種別 | 概要 / 状況 |
| --- | --- | --- |
| **oasdiff** | 破壊的変更検出 | Go 製 OSS（Apache-2.0）。`oasdiff breaking old.yaml new.yaml`。CI 向け。表13-2「進化」の自動化に最適（実測済み） |
| **pb33f openapi-changes** | 破壊的変更検出 | vacuum と同じ作者。変更履歴の可視化（HTML/TUI）も可能 |
| **IBM OpenAPI Validator** | Spectral ベースのリンター | `@ibm-cloud/openapi-ruleset` は `spectral:oas` を拡張し IBM Cloud API Handbook のルールを追加。**ガイドライン準拠ルールの実例として参考になる** |
| **Spectral OWASP ruleset** | ルールセット | `@stoplight/spectral-owasp-ruleset`。OWASP API Security Top 10 ベース。vacuum は同等ルールを内蔵 |
| **Zally**（Zalando） | ガイドラインベースのリンター | Zalando RESTful API Guidelines を実装。サーバー型。2025 年春以降目立った更新が見当たらず、新規採用は慎重に |
| **Optic** | 破壊的変更検出・リント | 2024 年に Atlassian が買収、**2026-01 にリポジトリがアーカイブ**。新規採用は非推奨（oasdiff 等へ移行） |
| 42Crunch API Audit | セキュリティ監査（商用） | セキュリティ観点のスコアリング。表13-3 のセキュリティ項目の補助に |
| Schemathesis | 実行時コントラクトテスト | リンターではなく、実装がスペック通りかをプロパティベースでテスト |

---

## 6. 推奨構成

```
                 ┌─ vacuum lint（Spectral 形式ルールセット）
PR の OpenAPI ───┤     extends: vacuum:oas(recommended) + vacuum:owasp
                 │     + 自組織ガイドライン（custom.spectral.yaml）
                 ├─ oasdiff breaking main..PR          ← 表13-2「進化」
                 └─ 人間レビュー（表13-2 の × 項目、表13-3、表13-4）
```

1. **ガイドラインは Spectral 形式で記述する**  
   Spectral・vacuum・IBM Validator・Stoplight 系ツールで共通に使え、資産を流用しやすい。
2. **実行エンジンは vacuum を第一候補**  
   デフォルトで命名・description・DELETE ボディ・未使用セキュリティを検出し、OWASP 内蔵、HTML レポートあり、高速。  
   Redocly でドキュメント生成・バンドルもしているなら Redocly に寄せるのも可（ただしルール記法は独自）。
3. **破壊的変更は oasdiff で CI ゲート化**
4. **自動化できない項目は PR テンプレートのチェックリストとして残す**  
   「既存要素との一貫性」「名前と型の意味的整合」「ゴールフロー」「提供可能」「ニーズとの一致」など。書籍の言う「レビュー担当者が人間にしかできないチェックに専念できる」状態を目指す。

---

## 7. インストール

### 7.1 このリポジトリで使う（推奨）

Spectral / Redocly / vacuum は npm パッケージとして提供されているので、**プロジェクトの devDependencies に入れてバージョンを固定**するのが一番扱いやすいです（`package.json` 同梱、動作確認済み）。

```bash
# 前提: Node.js 20 以上 + pnpm（確認環境は Node.js v24 / pnpm 11.7.0）
# pnpm が無ければ: corepack enable（package.json の packageManager に従って pnpm が使われる）
pnpm install          # package.json の devDependencies を入れる
pnpm lint             # examples/valid.yaml を 3 ツールで検査（全部通る）
pnpm lint:invalid     # examples/invalid.yaml を 3 ツールで検査（違反が出る）
```

> **pnpm の注意点:** pnpm は既定で依存パッケージのインストールスクリプトを実行しません。vacuum は postinstall でバイナリを取得するため、`pnpm-workspace.yaml` で許可しています（同梱済み）。
>
> ```yaml
> allowBuilds:
>   '@quobix/vacuum': true   # 必須。false だと vacuum コマンドが動かない
>   '@scarf/scarf': false    # 利用統計の送信（不要）
> ```
>
> 新規プロジェクトでは `pnpm approve-builds` で対話的に許可しても構いません。

新しいプロジェクトに入れる場合:

```bash
pnpm add -D @stoplight/spectral-cli @redocly/cli @quobix/vacuum
```

oasdiff は Go 製なので npm には入っていません（npm の `oasdiff` は無関係のプレースホルダー）。

```bash
# 前提: Go 1.22 以上（確認環境は 1.23）
go install github.com/oasdiff/oasdiff@latest
# $(go env GOPATH)/bin を PATH に通す
```

### 7.2 ツール別のインストール方法

✅ = この環境で実際に試した方法。それ以外は各公式ドキュメント記載の方法（未検証）。

| ツール | npm パッケージ（pnpm） | Homebrew | Docker | その他 |
| --- | --- | --- | --- | --- |
| Spectral | ✅ `pnpm add -D @stoplight/spectral-cli` | — | `docker run --rm -v "$PWD:/tmp" stoplight/spectral lint /tmp/openapi.yaml` | GitHub Releases のスタンドアロンバイナリ |
| Redocly CLI | ✅ `pnpm add -D @redocly/cli` | — | `docker run --rm -v "$PWD:/spec" redocly/cli lint openapi.yaml` | `pnpm dlx @redocly/cli lint` で単発実行 |
| vacuum | ✅ `pnpm add -D @quobix/vacuum` | `brew install daveshanley/vacuum/vacuum` | `docker run --rm -v "$PWD:/work:ro" dshanley/vacuum lint -d /work/openapi.yaml` | `go install github.com/daveshanley/vacuum@latest` |
| oasdiff | ✕（別物） | `brew install oasdiff` | `docker run --rm -v "$PWD:/specs:ro" tufin/oasdiff breaking /specs/old.yaml /specs/new.yaml` | ✅ `go install github.com/oasdiff/oasdiff@latest` |

### 7.3 確認したバージョン

| ツール | バージョン | 確認コマンド |
| --- | --- | --- |
| Spectral | 6.17.1 | `pnpm exec spectral --version` |
| Redocly CLI | 2.77.0 | `pnpm exec redocly --version` |
| vacuum | 0.40.0 | `pnpm exec vacuum version` |
| oasdiff | latest（2026-11-12 時点） | `oasdiff --version` |

---

## 8. valid / invalid の例

### 8.1 サンプルファイル

| ファイル | 内容 | 結果 |
| --- | --- | --- |
| `examples/invalid.yaml` | 書籍の `POST /send-email` 例をもとに、表13-2 の違反を仕込んだスペック | 3 ツールとも違反あり |
| `examples/valid.yaml` | 同じ題材をガイドラインに沿って直したスペック | **3 ツール（vacuum `all`+OWASP を含む）とも違反 0（hint まで 0）** |
| `examples/invalid-v2.yaml` | `invalid.yaml` に破壊的変更を入れた版 | oasdiff で error 2 件 |
| `examples/valid-v2.yaml` | `valid.yaml` に任意プロパティだけを追加した版 | oasdiff で breaking 0 件（info のみ） |

| ルールセット | 内容 |
| --- | --- |
| `examples/custom.spectral.yaml` | 自組織ガイドラインを想定したカスタムルール 5 つ（Spectral 形式。Spectral / vacuum 共用） |
| `examples/spectral.yaml` | `spectral:oas` ＋ カスタムルール |
| `examples/vacuum.yaml` | `vacuum:oas`(recommended) ＋ `vacuum:owasp`(all) ＋ カスタムルール |
| `examples/redocly.yaml` | Redocly `recommended` ＋ 同等のカスタムルール（configurable rules 記法） |

### 8.2 チェック項目ごとの invalid / valid

各例の「検出」列は、invalid 側を検出するツールとルール名です。valid 側は 3 ツールとも通ることを確認済みです。

#### ① パスパラメータが定義されていない（表13-2「パスが有効」）

```yaml
# ❌ invalid: {accountId} がパスにあるのに parameters に無い（名前も位置も違う）
/accounts/{accountId}:
  get:
    parameters:
      - name: account_id
        in: query
```

```yaml
# ✅ valid
/accounts/{accountId}:
  parameters:
    - name: accountId
      in: path
      required: true
      schema: { type: string, pattern: '^[0-9]{10}$', maxLength: 10 }
```

検出: Spectral `path-params` / Redocly `path-parameters-defined` / vacuum `path-params`（**3 ツールともデフォルト**）

#### ② セキュリティ定義が使われていない・ゴールがセキュアでない（表13-2「使われている」「セキュア」）

```yaml
# ❌ invalid: securitySchemes はあるが、どこからも参照されていない
components:
  securitySchemes:
    oauth: { type: oauth2, flows: { ... } }
# paths/ルートに security が無い
```

```yaml
# ✅ valid: ルートで既定を指定し、必要ならオペレーションで上書き
security:
  - oauth: [accounts:read]
paths:
  /accounts/{accountId}:
    delete:
      security:
        - oauth: [accounts:write]
```

検出: Redocly `security-defined`・`no-unused-components` / vacuum `oas3-unused-component`（デフォルト）、`owasp-protection-global-unsafe`・`owasp-protection-global-safe`（OWASP）。**Spectral デフォルトは未検出**

#### ③ セキュリティスコープの説明が空（表13-2「説明が有効」）

```yaml
# ❌ invalid
scopes:
  read: ''
```

```yaml
# ✅ valid
scopes:
  accounts:read: 口座情報の参照
  accounts:write: 口座の変更・解約
```

検出: カスタムルール `oauth-scope-description`（Spectral / vacuum）

#### ④ 成功とエラーに対処していない／エラーモデルが無い（表13-2「成功とエラーに対処している」「ボディが有効」）

```yaml
# ❌ invalid: 200 しか無い
responses:
  '200':
    description: OK
```

```yaml
# ✅ valid: 4xx/5xx を標準エラーモデル（RFC 9457 Problem Details）で返す
responses:
  '200': { ... }
  '400': { $ref: '#/components/responses/BadRequest' }
  '401': { $ref: '#/components/responses/Unauthorized' }
  '404': { $ref: '#/components/responses/NotFound' }
  '429': { $ref: '#/components/responses/TooManyRequests' }
  '500': { $ref: '#/components/responses/InternalServerError' }
# components.responses.* は application/problem+json で #/components/schemas/Problem を返す
```

検出: Redocly `operation-4xx-response` / vacuum OWASP `owasp-define-error-responses-401/429/500` など。**Spectral デフォルトは未検出**

#### ⑤ 全プロパティが任意（表13-2「必須ステータスが有効」）

```yaml
# ❌ invalid: required が無い
schema:
  type: object
  properties:
    email: { type: string }
    msg:   { type: number }
```

```yaml
# ✅ valid
Account:
  type: object
  required: [accountId, accountName, createdAt]
  properties: { ... }
```

検出: カスタムルール `schema-all-optional`（Spectral / vacuum）、`rule/all-optional`（Redocly）。**デフォルトでは 3 ツールとも未検出**

#### ⑥ クエリパラメータに object（表13-2 パラメータ「型とフォーマットが有効」）

```yaml
# ❌ invalid
- name: account_id
  in: query
  schema: { type: object }
```

```yaml
# ✅ valid
- name: limit
  in: query
  schema: { type: integer, format: int32, minimum: 1, maximum: 100, default: 20 }
```

検出: カスタムルール `query-param-no-object`（Spectral / vacuum）、`rule/query-no-object`（Redocly）

#### ⑦ パスに動詞（表13-2「パスが有効」＝リソース指向ガイドラインの場合）

```yaml
# ❌ invalid
/send-email:
  post: ...
```

```yaml
# ✅ valid: リソース名詞 + HTTP メソッドで表す
/accounts:
  get: ...
/accounts/{accountId}:
  get: ...
  delete: ...
```

検出: カスタムルール `path-no-verb`（Spectral / vacuum）、`rule/path-no-verb`（Redocly）

> ハマりどころ: パスのキーを検査するとき、Spectral で一般的な `given: $.paths` + `field: "@key"` は **vacuum では発火しません**。`given: "$.paths[*]~"`（`field` なし）なら両方で動きます。

> 書籍のとおり、関数ベースの API をガイドラインで認めている組織ならこのルールは不要です。ルールは**自組織のガイドラインを機械化したもの**であるべきです。

#### ⑧ プロパティ名が命名規則に沿っていない（表13-2 プロパティ「名前が有効」）

```yaml
# ❌ invalid
properties:
  ts:      { type: integer }
  err_msg: { type: string }
```

```yaml
# ✅ valid
properties:
  createdAt: { type: string, format: date-time }
```

検出: vacuum `camel-case-properties`（**デフォルト**）、カスタムルール `property-camel-case`（Spectral / vacuum）、`rule/prop-camel`（Redocly）。  
`ts` は camelCase としては正しいため**検出されない**。略語禁止は禁止語リストのルールを別途書く必要がある。

#### ⑨ DELETE にリクエストボディ（表13-2 パラメータ「位置が有効」）

```yaml
# ❌ invalid
delete:
  requestBody:
    content:
      application/json:
        schema: { type: object }
```

```yaml
# ✅ valid: 対象はパスで指定し、ボディは持たない
/accounts/{accountId}:
  delete:
    responses:
      '204': { description: 解約済み }
```

検出: vacuum `no-request-body`（**デフォルト**）。Spectral / Redocly のデフォルトは未検出

#### ⑩ 破壊的変更（表13-2「進化」）

```diff
# ❌ invalid → invalid-v2: 既存コンシューマが壊れる
   schema:
     type: object
+    required: [email]           # 任意 → 必須
     properties:
       email: { type: string }
-      msg: { type: number }
+      msg: { type: string }     # 型変更
```

```text
$ oasdiff breaking examples/invalid.yaml examples/invalid-v2.yaml
2 changes: 2 error, 0 warning, 0 info
error  [request-property-became-required]  the request property `email` became required
error  [request-property-type-changed]     the `msg` request property `type` changed from `number` to `string`
```

```diff
# ✅ valid → valid-v2: 任意プロパティの追加だけなので互換
     createdAt: { ... }
+    branchName:
+      type: string
+      description: 取扱支店名（v2 で追加・任意）
```

```text
$ oasdiff breaking examples/valid.yaml examples/valid-v2.yaml
No breaking changes to report, but the specs are different.
```

CI で止めるには `oasdiff breaking old.yaml new.yaml --fail-on ERR`（破壊的変更があると exit 1）。

#### ⑪ 【参考】リンターでは検出できない例

```yaml
# リンター的には「valid」だが、設計としては誤り
properties:
  message:
    type: number      # 「メッセージ」なのに数値
    description: 送信するメッセージ
    minimum: 0
    maximum: 100
```

名前・型・description の**意味が合っているか**はどのツールでも判定できません。書籍の「リントは形式だけを検証し、内容までは検証しない」の典型例で、人間のレビューで見るべき項目です。

### 8.3 実行結果のまとめ

| | `invalid.yaml` | `valid.yaml` |
| --- | --- | --- |
| Spectral（`spectral.yaml`） | 4 errors / 15 warnings | 0 |
| Redocly（`redocly.yaml`） | 9 errors / 15 warnings | 0（"Your API description is valid"） |
| vacuum（`vacuum.yaml`） | 19 errors / 45 warnings / 2 info | 0（"A perfect score!"） |
| oasdiff breaking（v1 → v2） | 2 errors | 0 |

---

## 9. 実行方法

```bash
# まとめて
pnpm lint               # valid.yaml → 全部通る
pnpm lint:invalid       # invalid.yaml → 違反が出る

# 個別
pnpm exec spectral lint examples/invalid.yaml -r examples/spectral.yaml
pnpm exec redocly  lint examples/invalid.yaml --config examples/redocly.yaml
pnpm exec vacuum   lint -d -r examples/vacuum.yaml examples/invalid.yaml

# vacuum の HTML レポート
pnpm exec vacuum html-report -r examples/vacuum.yaml examples/invalid.yaml report.html

# 各ツールのデフォルトルールだけで見る
echo 'extends: ["spectral:oas"]' > .spectral.yaml   # Spectral はプロセス置換 <(...) を読めないのでファイルにする
pnpm exec spectral lint examples/invalid.yaml
pnpm exec redocly  lint examples/invalid.yaml
pnpm exec vacuum   lint -d examples/invalid.yaml

# 破壊的変更
oasdiff breaking examples/invalid.yaml examples/invalid-v2.yaml --fail-on ERR
oasdiff breaking examples/valid.yaml   examples/valid-v2.yaml   --fail-on ERR
```

## 参考

- Optic のアーカイブ: <https://github.com/opticdev/optic>, <https://www.atlassian.com/blog/announcements/optic-acquisition>
- Zally: <https://github.com/zalando/zally>
- IBM OpenAPI Validator: <https://github.com/ibm/openapi-validator>
