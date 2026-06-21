# MCPコード知識ベース 設計プランニング

> 位置づけの再定義: 本システムは「事前学習」ではなく、**出典・依存・検証環境つきの再利用可能なコード知識ベース（RAG / コード検索・評価・提示基盤）** である。MCPサーバーが実行時にコード候補を検索・評価・提示する以上、実態は pretraining ではなく検索基盤であり、評価指標とデータ設計はこの前提に揃える。

---

## 1. スコープ

**対象**
- ファインチューニング/事前学習向けデータセット（コード、トレース等）を、コーディングエージェントが実行時に呼び出して「ベストプラクティスコード」を取得できる検索・評価基盤として整備する。
- コーディングエージェントがライブラリ使用時に呼び出し、引数（ライブラリ・バージョン・文脈）から高品質コードを提示する。

**当面スコープ外（将来フェーズ）**
- エージェントからの自己提出・自己成長ループ。書き込み経路は分離した構造にしておき、将来 quarantine フロー経由で追加する。

---

## 2. 設計原則

1. **コード片DBではなく知識ベース** — コードだけでなく出典・ライセンス・依存・検証環境をまとめて「実行可能単位」で持つ。
2. **閾値未満でも除外しない** — 機械検証/LLM judge で低評価でもコーパスから削除せず、スコア履歴と適用条件を残す。返答は「透明に根拠付きで提示」する。
3. **LLM judge を最終判断役にしすぎない** — まず機械的に検証できるもの（build/test/API互換/deprecated/脆弱依存）を通し、それを通過した候補のみ LLM judge に渡す。
4. **検索は絞りすぎない** — タグの完全一致 AND で落とすと検索漏れする。must（必須）と should（加点）を分け、semantic search と rerank を組み合わせた hybrid 構成にする。
5. **ストレージは抽象化する** — sqlite-vec は MVP に最適だが pre-v1 で破壊的変更リスクがある。中核を抽象化し、将来 LanceDB / Qdrant / pgvector に逃がせるようにする。
6. **検証は再現可能に** — コード片だけでは build/test は再現できない。toolchain・lockfile・feature flags 等のスナップショットを保存する。

---

## 3. 全体アーキテクチャ

```
[オフライン: 取り込みバッチ]
データセット
  → 正規化（実行可能単位 code_artifacts へ分解）
  → ライセンス/秘匿情報スキャン
  → タグ付け（8軸ファセット）+ 依存関係の構造化抽出
  → embedding 生成
  → 機械検証を1回実行（build/test/deprecated/脆弱性）、環境スナップショットとともにキャッシュ
  → ストレージへ格納

[オンライン: MCPサーバー（検索専用）]
エージェントの呼び出し引数
  → ① hybrid 初期検索（must条件で絞り + should加点 + semantic）
  → ② 抽出/rerank（embedding類似度 → 必要なら軽量reranker）
  → ③-a 機械検証の参照（キャッシュ結果 + リクエストバージョン整合性チェックのみ軽量実行）
  → ③-b LLM judge（③-a通過候補のみ、質的評価）
  → ④ 返答（最有力候補をコード + 出典 + ライセンス + 適用条件 + 注意点 + スコア根拠 + タグで返却）
```

書き込み経路（自己成長）はこの図には含めず、将来 §10 のフローで追加する。

---

## 4. データモデル

### 4.1 中核: 出典つきエントリ

```sql
CREATE TABLE code_entries (
    id                   TEXT PRIMARY KEY,
    summary              TEXT,            -- 用途・意図の要約（semantic検索の補助）
    -- 出典・権利メタデータ（コードを返すアプリでは必須）
    source_dataset       TEXT,            -- 由来データセット
    dataset_version      TEXT,
    repo_url             TEXT,
    commit_sha           TEXT,
    file_path            TEXT,
    license              TEXT,            -- SPDX識別子（例: MIT, Apache-2.0, unknown）
    redistribution_policy TEXT,           -- allow / restricted / forbidden / unknown
    provenance           TEXT,            -- human_oss / generated / mixed / unknown
    secret_scan_status   TEXT,            -- clean / flagged / unscanned
    created_at           TEXT NOT NULL
);
```

### 4.2 実行可能単位（コード片ではなく）

```sql
CREATE TABLE code_artifacts (
    id         TEXT PRIMARY KEY,
    entry_id   TEXT NOT NULL REFERENCES code_entries(id),
    kind       TEXT NOT NULL,   -- snippet / file / project / test / manifest / fixture
    path       TEXT,
    content    TEXT NOT NULL
);
CREATE INDEX idx_artifacts_entry ON code_artifacts(entry_id, kind);
```

> manifest（Cargo.toml 等）・lockfile・fixture・test を artifact として保持することで、機械検証を再現可能にする。

### 4.3 タグ（8軸ファセット、依存は別テーブルへ分離）

```sql
CREATE TABLE tags (
    entry_id    TEXT NOT NULL REFERENCES code_entries(id),
    axis        TEXT NOT NULL,   -- language / library / runtime / sync_mode /
                                  -- error_handling / security_requirement / granularity
    value       TEXT NOT NULL,
    min_version TEXT,            -- axis=library の場合のみ
    max_version TEXT,
    PRIMARY KEY (entry_id, axis, value)
);
CREATE INDEX idx_tags_axis_value ON tags(axis, value);
```

依存関係は `tags` に混ぜず別テーブル化（OSV照会・互換チェックで構造化が効く）:

```sql
CREATE TABLE dependencies (
    entry_id         TEXT NOT NULL REFERENCES code_entries(id),
    ecosystem        TEXT NOT NULL,   -- cargo / npm / pypi / maven
    name             TEXT NOT NULL,
    version_req      TEXT,            -- 宣言された要求（例: ">=1.28,<2.0"）
    resolved_version TEXT,            -- lockfileの確定バージョン
    features         TEXT,            -- JSON配列
    optional         INTEGER DEFAULT 0
);
CREATE INDEX idx_deps_pkg ON dependencies(ecosystem, name);
```

### 4.4 評価履歴（append-only、再現性スナップショット込み）

```sql
CREATE TABLE score_history (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    entry_id            TEXT NOT NULL REFERENCES code_entries(id),
    eval_type           TEXT NOT NULL,   -- mechanical_build / mechanical_test /
                                          -- mechanical_api_compat / mechanical_deprecated_api /
                                          -- mechanical_vuln_scan / llm_judge
    result              TEXT NOT NULL,   -- pass / fail / 数値スコア（JSON可）
    detail              TEXT,            -- JSON: エラー内容・判定根拠
    context_snapshot    TEXT,            -- 評価対象リクエスト/バージョン（取り込み時はNULL）
    toolchain_snapshot  TEXT,            -- JSON: compiler/toolchain version, OS, feature flags
    dependency_snapshot TEXT,            -- JSON: lockfile相当の確定依存
    evaluator_version   TEXT,            -- 使用モデル/ツールのバージョン
    created_at          TEXT NOT NULL
);
CREATE INDEX idx_score_code ON score_history(entry_id, eval_type, created_at);
```

> **閾値未満でも除外しない** を構造として保証: `score_history` は INSERT 専用。UPDATE/DELETE はアプリ層で禁止し、トリガーでもガードする。低評価は「事実の記録」として蓄積し、`applicability_conditions` の導出材料にする。

### 4.5 promotion_status（自己成長を見据えた将来用カラム）

当面は全エントリ `promoted` 固定でよいが、将来の quarantine フローに備えてカラムだけ用意:

```sql
ALTER TABLE code_entries ADD COLUMN promotion_status TEXT DEFAULT 'promoted';
-- quarantined / verified / rejected / promoted / deprecated
```

### 4.6 埋め込み

`sqlite-vec` 拡張で別テーブル保持。ただし §7 のストレージ抽象化越しにアクセスし、直接依存を最小化する。

---

## 5. タグ設計（8軸）と must / should 分類

| 軸 | 例 | 抽出方法 | 検索での扱い |
|---|---|---|---|
| 言語 | rust, python, ts | tree-sitter のパーサ種別 | **must** |
| ライブラリ/バージョン | tokio `>=1.28,<2.0` | import文 + manifest、範囲で保持 | **must**（メジャー一致） |
| 実行環境 | linux, wasm32, no_std | target/feature, import先 | should |
| 非同期/同期 | async, sync | `async fn`/`.await`/spawn検出 | should |
| エラー処理 | result伝播 / panic前提 / unwrap多用 | AST上の `?`/`unwrap`/`panic!` 分類 | should |
| セキュリティ要件 | unsafe使用 / 入力検証あり / 暗号処理 | unsafe検出 + 危険パターン静的検出 + 取り込み時LLM補助 | should |
| 依存関係 | （dependencies テーブル参照） | manifest/lockfile | must（ecosystem一致） |
| コード粒度 | snippet / function / module / full_app | 行数・トップレベル定義数 | should |

- **must**: language / ecosystem / メジャーバージョン → AND で必須フィルタ
- **should**: library / runtime / sync_mode / error pattern → 加点（落とさない）
- **semantic**: 用途・意図・API名 → embedding検索
- **rerank**: 軽量scorer または LLM judge

エラー処理・セキュリティは完全に機械判定し切れないため、取り込み時（バッチ）に LLM 補助タグ付けを併用。リクエスト時には使わない。

---

## 6. 検索パイプライン（hybrid）

**① 初期検索（must で絞り、should で加点）**
- エージェント引数を取り込み時と同じ `tagging` ロジックでファセット化（クエリ側と格納側でロジック共有 → 整合性確保）
- must 条件（language / ecosystem / major version）で AND 絞り込み
- should 条件は除外条件にせず加点に回す → 候補を数十〜百件（再現率優先）

**② 抽出 / rerank**
- 候補 embedding とクエリ embedding のコサイン類似度で上位 5〜10 件
- 必要なら軽量 cross-encoder（`ort` / `candle` でローカル推論）で再順位付け

**③-a 機械検証の参照（リクエスト時は軽量）**
- build/test/deprecated/脆弱性は取り込み時に1回実行済み、`score_history` にキャッシュ
- リクエスト時はキャッシュ参照のみ。例外として「ライブラリバージョン互換」だけは、検証済みバージョン範囲とリクエストバージョンの重なりをその場で軽量チェック（再ビルドはしない）
- ビルド不可など機械検証 fail の候補は④の「推奨」から外すが、削除はせず記録を残す

**③-b LLM judge**
- ③-a 通過候補のみを質的評価（コンテキスト適合・イディオム適合・設計上の懸念）
- 閾値未満でも除外せず `score_history` に記録。提示順位と注意点表示に反映するのみ

**④ 返答**
- 最有力候補を根拠込みで返す。良い候補が無い場合も空応答にせず、最近傍を注意点付きで返す

---

## 7. ストレージ抽象化

中核ロジックは具体DBに直接依存させず、trait 越しにアクセスする。

```rust
trait VectorStore {
    fn upsert(&self, id: &str, embedding: &[f32], meta: &Meta) -> Result<()>;
    fn search(&self, query: &[f32], filter: &MustFilter, k: usize) -> Result<Vec<Hit>>;
}

trait MetadataStore {
    fn find_by_tags(&self, must: &[TagFilter], should: &[TagFilter]) -> Result<Vec<EntryId>>;
    fn append_score(&self, record: &ScoreRecord) -> Result<()>;   // INSERT only
    fn fetch_entry(&self, id: &EntryId) -> Result<EntryView>;
}
```

- MVP実装: `SqliteMetadataStore` + `SqliteVecStore`
- 将来: `LanceDbStore` / `QdrantStore` / `PgVectorStore` を同 trait で差し替え

---

## 8. MCPインターフェース設計（tools / resources / prompts に分割）

すべてを1つの巨大 tool に押し込まず、エージェントが扱いやすい単位に分割する（rmcp は tools/resources/prompts に対応）。

| 種別 | 名前 | 用途 | 導入フェーズ |
|---|---|---|---|
| tool | `search_code_patterns` | 条件から候補検索（①②③参照） | Phase 2 |
| tool | `judge_code_fit` | 候補と現在の実装文脈の適合評価 | Phase 4 |
| tool | `submit_code_example` | 将来の自己成長用（当面は無効/未公開） | Phase 5 |
| resource | `code://entry/{id}` | コード本体（artifacts）+ メタ取得 | Phase 2 |
| resource | `score://entry/{id}` | 評価履歴の取得 | Phase 4 |
| prompt | `explain_applicability` | 返却コードの適用条件・使い方説明テンプレート | Phase 4 |

- transport: ローカルエージェント用途は `stdio`、常駐/複数クライアントは streamable HTTP
- `#[tool]` の JSON Schema description を手厚く書く（引数の質が①の検索精度に直結）

---

## 9. 返答スキーマ

```json
{
  "entry_id": "…",
  "code": "…",
  "provenance": {
    "source": "…",
    "repo_url": "…",
    "commit_sha": "…",
    "license": "MIT",
    "redistribution_policy": "allow",
    "origin": "human_oss"
  },
  "tags": {
    "language": "rust",
    "library": "tokio",
    "library_version_range": ">=1.28,<2.0",
    "runtime": "linux",
    "sync_mode": "async",
    "error_handling": "result_propagation",
    "security_requirement": "none_detected",
    "granularity": "function"
  },
  "dependencies": [
    { "ecosystem": "cargo", "name": "tokio", "version_req": ">=1.28", "features": ["rt-multi-thread"] }
  ],
  "applicability_conditions": [
    "tokio 1.28以上が必要（spawn_blocking APIに依存）",
    "no_std環境では不可"
  ],
  "caveats": [
    "unwrap()を1箇所使用。本番ではエラーハンドリング追加を推奨"
  ],
  "score_rationale": {
    "verified_with": {
      "toolchain": "rustc 1.82 / x86_64-linux",
      "build": "pass",
      "test": "pass (3/3)",
      "deprecated_api": "none detected",
      "vuln_scan": "no known CVE (OSV, checked 2026-05-10)"
    },
    "llm_judge": {
      "score": 0.86,
      "rationale": "意図したエラー処理パターンに合致。命名はやや非慣用的。",
      "model": "…",
      "evaluated_at": "2026-06-10"
    },
    "history_note": "tokio<1.25環境向けリクエストでは過去に非互換と判定"
  }
}
```

`provenance`（source/license）、`verified_with`、`caveats` は MVP から必ず含める。

---

## 10. 自己成長ループ（将来フェーズ、設計のみ）

エージェント提出をいきなり本番インデックスに入れない。`promotion_status` で状態管理する。

```
Agent submission
  → quarantine（隔離インデックス）
  → license & secret scan
  → mechanical verification（build/test/vuln）
  → human or trusted judge review
  → promoted index
```

- 各段階で `promotion_status` を遷移（quarantined → verified → promoted、不適合は rejected/deprecated）
- 自己評価をそのまま信用せず、§6 の機械検証ゲート → judge を必ず通す
- スコア履歴は append-only のまま流用できる

---

## 11. 実装スタック・モジュール構成

| 用途 | crate |
|---|---|
| MCPサーバー | rmcp |
| 非同期runtime | tokio |
| メタデータDB | rusqlite（+ sqlite-vec拡張） |
| 構文解析/タグ抽出 | tree-sitter + 言語別grammar |
| LLM judge / embedding API | reqwest + serde_json |
| 脆弱性照会 | OSV API（reqwest） |
| ID生成 | uuid |
| ログ | tracing |

```
crates/
  mcp-server/    # rmcp の tools/resources/prompts、検索パイプラインのオーケストレーション
  tagging/       # tree-sitter ベースのタグ抽出（取り込み/クエリで共有）
  storage/       # VectorStore / MetadataStore trait と SQLite 実装
  verification/  # 機械検証（サンドボックス build/test、OSV照会）
  judge/         # LLM judge / embedding 呼び出しラッパー
  ingestion/     # 取り込みバッチ用バイナリ（tagging/verification/storage を再利用）
```

`tagging` を取り込み側とサーバー側で共有することで、格納時とクエリ時のタグ付けの整合性を担保する。

---

## 12. フェーズ計画

### Phase 0 — 基盤整備
- cargo ワークスペース、モジュール骨格
- `VectorStore` / `MetadataStore` trait 定義（具体DB非依存）
- SQLite スキーマ（§4 全テーブル、provenance・snapshot カラム込み）を最初から用意
- tracing 導入、INSERT専用トリガーで `score_history` をガード
- **完了条件**: 空DBに対し trait 経由で読み書きが通る

### Phase 1 — 取り込みMVP（書き込み経路）
- データセット正規化 → `code_artifacts` への分解（snippet/file/manifest/test）
- ライセンス/秘匿情報スキャン → `license` / `secret_scan_status` 記録
- tree-sitter で language / library / API symbol 抽出 → `tags`
- 依存関係の構造化抽出 → `dependencies`（ecosystem/name/version）
- embedding 生成・格納
- **完了条件**: 実データセットを投入し、出典・タグ・依存・embedding が揃って格納される

### Phase 2 — 検索MVP（MCP tool 1本）
- tool: `search_code_patterns`（hybrid: must絞り + should加点 + semantic）
- resource: `code://entry/{id}`
- rmcp（stdio）で実エージェントから呼び出し
- 返答に `provenance(source/license)` / `caveats` を必ず含める（この段階では verified_with は未充足でも可）
- **完了条件**: エージェントが引数を投げて候補コード+出典を受け取れる

### Phase 3 — 機械検証（LLMより先）
- Cargo 系から着手: サンドボックスで `cargo check` / test 実行
- `toolchain_snapshot` / `dependency_snapshot` を `score_history` に保存（再現性確保）
- deprecated API（コンパイラ警告ベース）
- OSV 照会（ecosystem/name/version/commit で既知CVE確認）
- 結果を `verified_with` として返答に反映、fail は推奨から除外（記録は残す）
- **完了条件**: 取り込み時に検証が走り、返答が「検証済みかどうか」を提示できる

### Phase 4 — LLM judge + resources/prompts
- tool: `judge_code_fit`（③-a 通過候補のみ評価）
- resource: `score://entry/{id}`、prompt: `explain_applicability`
- 閾値未満は除外せず `score_history` に記録、提示順位・注意点に反映
- `applicability_conditions` を `score_history` 集計から半自動導出
- **完了条件**: 機械検証 → LLM judge の2段評価が通り、根拠付き返答が完成

### Phase 5 — 自己成長ループ（将来、必要時）
- tool: `submit_code_example`（初期は無効/限定公開）
- quarantine フロー（§10）+ `promotion_status` 遷移
- license/secret scan → mechanical verification → review → promoted index
- **完了条件**: 提出が隔離 → 検証 → 昇格の経路を通り、本番インデックスを汚染しない

---

## 13. 残課題・リスク

- **サンドボックス実行方式**: Docker（`bollard`）か制限付き subprocess か。再現性とレイテンシのトレードオフ。
- **検証の陳腐化**: toolchain/依存は時間で古くなる。再検証のバッチ運用（定期再実行 + snapshot 差分検知）が必要。
- **LLM judge コスト**: 候補数 × リクエスト数で増える。③-a の絞り込み精度と候補数上限の設計が鍵。
- **sqlite-vec の pre-v1 リスク**: §7 の抽象化で吸収。大量データ/高QPS時は LanceDB / Qdrant / pgvector へ移行判断。
- **ライセンス整合**: redistribution_policy が forbidden/unknown のコードを「提案」として返してよいかのポリシー策定（返却時フィルタ or 警告）。
- **applicability_conditions の粒度**: `score_history` からどの単位でルール化するか（バージョン帯・環境別）。