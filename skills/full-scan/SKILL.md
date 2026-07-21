---
description: "リリース前の全ファイル静的解析スキル。プロジェクト構造を自動検出し、ソースモジュールごとのサブエージェントと依存関係スキャンを並列実行してコードベース全体の脆弱性を検出する。.security-owners.json が存在する場合は GitHub イシューを自動作成してチームメンバーにアサインする。手動呼び出し専用: /full-scan"
disable-model-invocation: true
---

# full-scan

**役割**: リリース前・全ファイル静的解析スキル（言語・フレームワーク非依存）

プロジェクト構造を自動検出し、以下を並列実行する：
- **ソースモジュールサブエージェント**（検出されたモジュールごとに1つ）: 全ソースファイルの静的解析
- **パッケージサブエージェント**: 依存関係の既知 CVE スキャン + シークレット漏洩チェック

> **コンテキスト上限への注意**: リポジトリが大規模な場合、全ファイルがコンテキストウィンドウに収まらない可能性がある。**スキップが発生した場合はサイレントに無視せず、必ずスキップしたファイル一覧とともに PARTIAL スキャンとして報告する。**

---

## 引数

| 引数 | 必須 | 説明 |
|------|------|------|
| `対象パス` | 任意 | 明示的にスキャンしたいディレクトリ（省略時は自動検出） |
| `--confidence N` | 任意 | 報告する最低信頼度（デフォルト: 8、範囲: 1〜10） |
| `--no-issues` | 任意 | GitHub イシューの自動作成をスキップする |

---

## 手順

### Step 1: プロジェクト構造の自動検出

以下の順序で構造を把握する：

1. **ソースディレクトリの特定**
   - `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` / `Gemfile` / `pom.xml` / `build.gradle` を `node_modules/` / `.git/` / `dist/` / `build/` を除外して検索する
   - 各マニフェストファイルの場所からソースディレクトリを推定する（例: `package.json` がある場所の `src/` / `lib/` / `app/` 等）
   - 引数でパスが指定された場合はそれを優先する

2. **言語・フレームワークの検出**
   - マニフェストファイルの内容（dependencies / devDependencies 等）からフレームワークを特定する
   - 例: `next` → Next.js、`express` → Express、`fastapi` → FastAPI、`rails` → Rails 等
   - **Next.js App Router の検出**: `next` が依存関係にあり、かつ `src/app/api/` または `app/api/` ディレクトリが存在する場合、App Router プロジェクトと判定する。この場合 `app/api/` 配下のルートグループを機能ドメイン（`auth/`、`boards/`、`chat/`、`cron/` 等）ごとにサブエージェントへ分割することを推奨する。

3. **`.security-owners.json` の読み込み（存在する場合）**
   - プロジェクトルートに `.security-owners.json` があれば読み込む
   - このファイルは発見をチームメンバーへ自動アサインするための責任マッピングを定義する
   - フォーマット: `templates/security-owners.example.json` を参照
   - ファイルが存在しない場合はスキップし、Step 6 のイシュー作成では `owner` フィールドを空にする

4. **モジュール分割の決定**
   - ソースディレクトリが複数ある場合（モノレポ等）: モジュールごとにサブエージェントを割り当てる
   - ソースディレクトリが1つの場合: サブエージェント1つで全体をカバーする
   - モジュールが4つ以上ある場合: 関連するモジュールをグループ化する（コンテキストコスト削減）

5. **スキャン前にファイル一覧を確定する（必須）**:
   - 各モジュールのソースファイルを `find` コマンドで列挙し、**総ファイル数を記録する**
   - この時点で確定した総ファイル数が、カバレッジ計算の分母になる
   - 「このスキャンは以下のモジュールを対象とします: [モジュール一覧] / 総ファイル数: [N]件」と明示する

### Step 2: サブエージェントを並列実行

以下を**同時に**起動する（並列実行）：

---

#### ソースモジュールサブエージェント（モジュールごと）

検出された各ソースモジュールのファイルを静的解析する。

**解析手順（必須）:**

1. **解析開始前に全ファイルを列挙する**: `find <src_dir> -type f \( -name "*.ts" -o -name "*.tsx" -o -name "*.js" -o -name "*.py" 等 \)` で対象ファイルの完全なリストを取得し、**総数 N を記録する**
2. **全ファイルを順番に読んで解析する**: 以下の優先度順に読み進める
3. **コンテキスト上限に達した場合**: 解析を即座に停止し、「読めなかったファイル一覧」を記録する。**未解析ファイルをスキップしたままスキャン完了と報告してはならない**

**読み込み優先度（コンテキスト上限時のみ意味を持つ）:**

| 優先度 | ファイル種別 | 汎用パターン例 |
|--------|------------|--------------|
| 1 | HTTPエントリーポイント・ルーター | `routes/`, `controllers/`, `api/`, `handlers/`, `endpoints/` |
| 2 | ミドルウェア・フィルター | `middleware/`, `filters/`, `interceptors/` |
| 3 | 認証・認可 | `auth/`, `security/`, `permissions/`, `guards/` |
| 4 | 外部入力を受け取る層 | リクエストボディを直接扱うファイル |
| 5 | データアクセス層 | `repository/`, `infra/`, `models/`, `db/`, `store/` |
| 6 | 外部サービス連携 | `clients/`, `adapters/`, `services/`, `integrations/` |
| 7 | ユーティリティ・ヘルパー | `utils/`, `lib/`, `helpers/`, `shared/` |

**報告義務（必須）:**
- 解析完了後に必ず `FILES_ANALYZED: <解析済み数> of <総数>` を報告する
- 未解析ファイルがある場合は **ファイルパスを全件列挙する**
- `解析済み数 < 総数` の場合、結果を **PARTIAL SCAN** として明示する

**検出対象（`--confidence` で指定した閾値以上のみ報告、デフォルト 8/10）:**
- **インジェクション**: SQL / NoSQL / コマンド / テンプレートインジェクション、パストラバーサル
- **認証・認可の欠陥**: セッション管理・JWT 誤実装・認可バイパス・権限昇格
- **ハードコードされた認証情報・API キー**: シークレット・接続文字列の平文埋め込み
- **危険なメソッドによる XSS**: `dangerouslySetInnerHTML` / `innerHTML` / `bypassSecurityTrustHtml` / `eval` / `document.write` 等
- **PII・トークン・パスワードのログ出力**
- **SSRF**: ホスト・プロトコルをユーザー入力で制御できる場合のみ

**除外ルール（報告しない）:**
- DoS・リソース枯渇・レート制限
- React / Angular 通常コンポーネントの XSS（危険なメソッド不使用の場合）
- クライアントサイドの認証チェック欠如（サーバーサイドが責任を持つ）
- AI システムプロンプトへのユーザー入力混入
- 環境変数・CLI フラグ経由の攻撃（信頼済み）
- Markdown・ドキュメントファイル
- ログスプーフィング
- パスのみ制御できる SSRF
- UUID は推測不能と仮定

**偽陽性フィルタリング:**
- 具体的な攻撃経路があるか（理論上ではなく実際に悪用可能か）
- 既存のフレームワーク・ライブラリが安全に処理している場合は除外する
- ログへの非 PII データ出力は脆弱性としない

---

#### パッケージサブエージェント

**依存関係スキャン:**

検出されたマニフェストファイルに応じて対応するコマンドを実行する：

| マニフェスト | コマンド | 抽出対象 |
|------------|---------|---------|
| `package.json` | `npm audit --json` または `yarn audit --json` | High / Critical |
| `requirements.txt` / `Pipfile` | `pip-audit` または `safety check` | High / Critical |
| `Gemfile` | `bundle audit check` | High / Critical |
| `go.mod` | `govulncheck ./...` | High / Critical |
| `Cargo.toml` | `cargo audit` | High / Critical |
| `pom.xml` / `build.gradle` | `dependency-check` | High / Critical |

ツールが存在しない場合はスキップして記録する。
`--force` や破壊的変更が必要な修正は別途明示する。

**シークレット漏洩チェック:**
- `gitleaks detect --source . --report-format json` を実行する
- `trufflehog filesystem .` を代替として試みる
- どちらもない場合はスキップして記録し、以下のインストール方法を案内する：
  ```
  # gitleaks
  brew install gitleaks        # macOS
  # または: https://github.com/gitleaks/gitleaks/releases

  # trufflehog
  brew install trufflehog      # macOS
  pip install trufflehog       # Python
  ```

---

### Step 3: 結果の統合・重複排除

1. 同一の脆弱性が複数サブエージェントから報告された場合は1件にまとめる
2. 深刻度別にソートする（Critical → High → Medium → Low）
3. フレームワーク固有の修正方法を具体的に記載する

**Critical を発見したら**: 即座に報告し、残りの処理を続けながらも即時対応を促す。

### Step 4: カバレッジ集計

```yaml
scan_date: <ISO8601>
scan_type: full_static

project:
  detected_modules: <検出されたモジュール名リスト>
  languages: <検出された言語リスト>
  frameworks: <検出されたフレームワークリスト>

coverage:
  modules:
    - name: <モジュール名>
      files_total: <総ファイル数>
      files_analyzed: <解析済み数>
      context_limit_reached: <true/false>
  skipped_files:
    - path: <スキップしたファイル>
      reason: <理由（コンテキスト上限・バイナリ等）>

packages:
  scanned: <スキャン済みマニフェスト数>
  tools_unavailable: <利用不可だったツールリスト>
  vulns_high_critical: <件数>

findings:
  critical: <件数>
  high: <件数>
  medium: <件数>
  low: <件数>

not_covered_by_this_scan:
  - デプロイ後のランタイム挙動 → /security-scan を使用
  - PR差分の即時レビュー → /security-review を使用
  - ビジネスロジックの欠陥 → ペネトレーションテスト（人手）が必要
  - インフラ・クラウド設定 → インフラ担当者レビューが必要
  - ゼロデイ脆弱性 → CVEデータベース外のため検出不可
  - マルチステップ攻撃チェーン → ペネトレーションテストが必要

# coverage_score = files_analyzed / files_total × 100（全モジュール合算）
# files_analyzed < files_total の場合は scan_status: PARTIAL
coverage_score: <files_analyzed合計 / files_total合計 × 100>%
scan_status: <COMPLETE | PARTIAL>
ci_result: <pass/fail>
```

### Step 5: レポート生成・履歴インデックス更新

1. **日付付きファイル名でレポートを保存する**
   - レポート: `./security-reports/YYYY-MM-DD-full-scan-report.md`
   - カバレッジ: `./security-reports/YYYY-MM-DD-full-scan-coverage.yml`
   - 日付は実行時の ISO 8601 形式（例: `2026-05-01`）
   - ディレクトリが存在しない場合は作成する

2. 深刻度別ソート・修正優先順位・具体的な修正コマンドを含める

3. **`./security-reports/index.md` に1行追記する**
   - ファイルが存在しない場合はヘッダーから作成する
   - 追記フォーマット:
   ```
   | YYYY-MM-DD | full-scan | 全ソース (<総ファイル数>files / <scan_status>) | <Critical件数> | <High件数> | <Medium件数> | <CI結果> | [レポート](YYYY-MM-DD-full-scan-report.md) |
   ```
   - CI結果: High 以上の発見が 0 件なら `✅ pass`、1件以上なら `❌ fail`

4. High 以上の発見がある場合、終了コード 1 を報告する

### Step 6: GitHub イシュー自動作成（`--no-issues` 未指定かつ gh CLI が利用可能な場合）

> `--no-issues` が指定されている場合、または `gh` コマンドが見つからない場合はこの Step をスキップする。

1. **`gh` CLI と認証の確認**
   ```bash
   gh auth status 2>/dev/null || echo "gh not authenticated"
   ```
   未認証の場合はスキップしてレポートにその旨を記載する。

2. **`.security-owners.json` の解釈**
   - Step 1 で読み込んだオーナーマッピングを使用する
   - 各発見のファイルパスを `paths` パターンと照合し、`github_username` を特定する
   - マッチしない場合は `default_owner` を使用する（未設定なら `assignee` なしでイシューを作成）

3. **発見ごとに GitHub イシューを作成する**
   - `severity:critical` / `severity:high` / `severity:medium` / `severity:low` ラベルが存在しない場合は自動作成する
   - `security` ラベルも作成する
   - 各発見を以下のフォーマットでイシューとして登録する：

   ```
   タイトル: [Security][<SEVERITY>] <発見のタイトル>

   本文:
   ## 概要
   <Descriptionの内容>

   ## 影響を受けるファイル
   - `<file>:<line>`

   ## 攻撃シナリオ
   <Attackの内容>

   ## 修正方法
   <Fixの内容>
   ```

   ```bash
   gh issue create \
     --title "[Security][HIGH] ..." \
     --body "..." \
     --label "security,severity:high" \
     --assignee "<github_username>"
   ```

4. **作成したイシューの URL 一覧をレポートに追記する**

---

## 3スキルの住み分け

| タイミング | スキル | 対象 |
|-----------|--------|------|
| 開発中・PR 単位 | `/security-review` | git diff のみ・静的解析 |
| リリース前 | `/full-scan` | 全ソースファイル + 依存関係（言語非依存） |
| ステージングデプロイ後 | `/security-scan` | ランタイム挙動・HTTP 動的テスト |

---

## このスキルでカバーできない領域

| 領域 | 推奨手段 |
|------|---------|
| デプロイ後の HTTP ヘッダー・動的挙動 | `/security-scan` |
| ビジネスロジックの欠陥（仕様知識が必要） | ペネトレーションテスト |
| インフラ・クラウド設定（IAM・ネットワーク ACL） | インフラ担当者レビュー |
| ゼロデイ脆弱性 | CVE データベース外のため検出不可 |
| マルチステップ攻撃チェーン | ペネトレーションテスト |

---

## 完了条件

- [ ] `./security-reports/YYYY-MM-DD-full-scan-report.md` が生成されている
- [ ] `./security-reports/YYYY-MM-DD-full-scan-coverage.yml` が生成されている（`scan_type: full_static` 明記）
- [ ] 検出されたモジュール・言語・フレームワークが明示されている
- [ ] 解析済みファイル数とスキップしたファイルが明示されている
- [ ] カバーできない領域が明示されている
- [ ] `severity_gate` 基準に基づく CI 結果（pass / fail）が明示されている
- [ ] `./security-reports/index.md` に1行追記されている
- [ ] `--no-issues` 未指定かつ `gh` 認証済みの場合、各発見の GitHub イシューが作成されている
