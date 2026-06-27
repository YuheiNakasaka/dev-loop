---
name: dev-loop
description: Reference for the dev-loop harness — the plan (dev-plan.md) + state ledger (dev-state.json) + /step loop protocol and its state schema. Load when working with dev-loop plans or the /step, /plan-init, /plan-status commands, or when designing a step-by-step, verified, resumable remediation/build plan for any project.
---

# dev-loop — plan + state + /step ハーネス

会話履歴ではなく **2 つのファイル** を正典にして、検証ゲートを挟みながら **1 ステップずつ** 開発を進める
loop-engineering ハーネス。状態を外部化しているので `/clear` で文脈を消しても再開できる。

- **仕様（何をやるか）**: `dev-plan.md`（既定 `.claude/dev-plan.md`）— 順序付きステップの正典。
- **状態台帳（何が起きたか・設定）**: `dev-state.json`（既定 `.claude/dev-state.json`）— `/step` が読み書きする。

設定値（ブランチ・ゲートコマンド・プロジェクト名・計画パス）は **すべて `dev-state.json` に置く**。
コマンドの prose にハードコードしないので、どのプロダクトでも同じコマンドが動く。

## コマンド

| コマンド | 役割 |
|---|---|
| `/plan-init [ゴール/仕様/レビュー文書]` | `dev-plan.md` + `dev-state.json` を生成。ゲート自動検出・ベースライン記録・レビュー設定。 |
| `/step [id]` | 1 ステップ実行（依存確認→実装→ゲート→**階層的レビューゲート**→state 更新→1 コミット）。空なら次の pending。 |
| `/step-review [id]` | レビューゲートだけを現作業ツリーに対して回す**読み取り専用**ビュー（コミット/前進しない）。 |
| `/plan-status` | 進捗の読み取り専用ビュー。 |

## 実行プロトコル（`/step` が毎回守る 12 + 1）

```
[開始前]
  1. dev-state.json を読み、設定(planDoc/branch/gateCommands/review/currentStep/steps/globalLearnings)を取得
  2. 対象ステップ確定（引数 > currentStep。in-progress があれば再開。mid-review 中断は verification.review/baseCommit を参照）
  3. branch があればそのブランチ上＆作業ツリー clean を確認（違えば停止。※mid-review 再開時の dirty は「階層的レビューゲート」節を参照）
  4. dependsOn が全て done＋検証緑か確認。未完/退行は先に是正
  5. globalLearnings で既知の地雷を確認
  6. planDoc で「ゴール/成功条件/検証方法」を読み、file:line を現物コードで再確認
[実行中]
  7. 当ステップを status=in-progress, startedAt に更新。review.enabled なら verification.review.baseCommit に現 HEAD を記録
  8. 実装 → 「検証方法」＋ gateCommands の各ゲートを実行（定義キーのみ）。退行ゼロを確認
[階層的レビューゲート（review.enabled のとき。round r = 1..maxRounds。詳細は次節）]
  9. Layer1(breadth): git diff <baseCommit> を対象に perspectives[] の各観点を dev-reviewer で並行起動。ラベル＋安定キー付き指摘を収集
 10. Layer2(meta): dev-review-meta を起動し、dedup・偽陽性反証・スコープ判定・成功条件突合・カバレッジ確認・requiresHuman 判定 → refined findings＋verdict
 11. 収束判定: blockOn 以上の未解決が無→収束。有＋autoFix＆全て autoFixable→修正し gateCommands 再実行→r++ で 9 へ。
     オシレーション(未解決キー集合が縮小しない)/maxRounds 超過/reviewer 失敗→収束失敗
[完了時]
 12. state をコミット前に最終形まで一括更新する（分割して currentStep/lastUpdated を後回しにしない＝コミット漏れの主因）。
     収束時: status=done(見送りは deferred＋理由), completedAt, verification(gates＋review), learnings, out-of-scope は steps[]＋planDoc に新規ステップ＋globalLearnings に追記, さらに currentStep を次へ・lastUpdated を更新。steps[].commit は自分のハッシュを埋め込めないため null のまま（次ステップ手順7で backfill）
     収束失敗時: status=blocked, verification.review を記録, currentStep は据え置き, コミットせず作業ツリー dirty のまま停止・報告
 13. 収束時のみ 1 ステップ=1 コミット（このステップの最後の操作）: git add -A で dev-state.json＋planDoc(dev-plan.md)＋実装を必ず一括ステージ（git commit -am や部分 add は使わない）→ commit → 直後に git status --porcelain で clean を確認（残差分はコミット漏れ＝追従コミットで取り込む）
```

> `review.enabled=false`（または `review` ブロック不在）なら 9〜11 を**丸ごとスキップ**し、従来の 10+1 として動く（後方互換）。

## 階層的レビューゲート（hierarchical review loop）

各ステップの**実装＋品質ゲートが緑になった後・done 記録/コミット/前進の前**に挟む、自己完結のレビュー機構。
「あらゆる観点のサブエージェントが階層的にレビューし、その結果をさらにレビューし、何度か反復して洗練し、
最終結果で次の処理を決める」を実現する。**外部プラグインに依存しない**（エンジンは dev-loop 同梱）。

### エンジン（dev-loop 同梱）

- 観点カタログ: `assets/review-perspectives.md`（ga-dev-reviewer 相当の breadth。13 観点のキー＋焦点チェックリスト）。
- Layer 1 reviewer: `dev-loop:dev-reviewer`（単一観点・読み取り専用）。観点数だけ並行起動。
- Layer 2 meta: `dev-loop:dev-review-meta`（レビュー結果のレビュー・読み取り専用）。1 回起動。
- オーケストレーション: `/step` 本体（メインエージェント）。**修正を行うのは常にメインエージェントのみ**（reviewer は編集しない）。
- ※もしエージェント型が解決できない環境なら、同じプロンプトで `general-purpose` 代替起動してもよい。

### レビュー対象 diff

- 常に **`git diff <baseCommit>`（作業ツリー＋staged の累積差分）**。`baseCommit` はステップ着手時の HEAD で、
  着手時（プロトコル 7）に `verification.review.baseCommit` へ**永続化**する（`/clear` 後の再開の鍵）。
- 再レビュー（round r≥2）は cold review せず、**前ラウンドの未解決キー**を seed に渡し、
  「解決済みが本当に直ったか＋修正で混入した退行」を累積 diff 上で検証する。

### ループ（round r = 1..maxRounds）

1. **Layer 1（breadth=横）**: `perspectives[]` の各観点を `dev-reviewer` で**並行起動**。各 reviewer は担当観点のみで
   `[must]/[ask]/[suggestion]/[nits]`＋安定キー付き指摘を返す（`key = sha1(観点|file|rule|要約)` 先頭 8 桁）。
2. **Layer 2（meta=縦／その結果のレビュー）**: `dev-review-meta` に Layer 1 全指摘＋ステップ文脈
   （ゴール/成功条件/`dependsOn`/`globalLearnings`/`baseCommit`/設定）を渡す。重複排除・**現物コードでの偽陽性反証**・
   スコープ判定（`in-step`/`out-of-scope`/`dependency`）・成功条件突合・観点カバレッジ確認・`requiresHuman`/`autoFixable`
   判定を行い、refined findings＋`verdict` を構造化（JSON）で返す。
3. **カバレッジ補充**: meta が `coverageGaps`（不足観点）を返したら、その観点の `dev-reviewer` を追加起動して 1〜2 に戻す。
4. **収束判定**:
   - `blockOn`（既定 `must`）以上の未解決が**無い** → **収束**。
   - 未解決があり `autoFix=true` かつ**全て `autoFixable`（in-step ＆ !requiresHuman ＆ 機械修正可）** →
     メインエージェントが修正 → **`gateCommands` を再実行**（退行ゼロをコミット時に保証）→ r++ で 1 へ。
   - **オシレーション検知**: round r の未解決 must キー集合が round r-1 の**真部分集合に縮小しない**
     （直したはずが再燃／A↔B 往復）→ **即停止＝収束失敗**（maxRounds より早く止める）。
   - **maxRounds 超過** / **reviewer 失敗**（エラー・パース不能）→ **収束失敗**（失敗を「指摘なし」と誤判定しない）。

### 次に適切な処理（最終 refined 結果 → state アクション 決定表）

| 最終結果 | 判定 | 次アクション |
|---|---|---|
| `blockOn` 以上の未解決なし（最初から clean / 修正後 clean） | 合格 | state を最終形へ（`status=done`、`verification.review` 記録、**`currentStep` 前進・`lastUpdated` 更新まで全部コミット前に**）→ `git add -A` で dev-state.json＋planDoc＋実装を一括ステージ → **収束時に 1 回だけ**コミット（メッセージ末尾に `[review: <verdict>, <rounds>r]`）→ `git status` で clean 確認 |
| 未解決 must が `requiresHuman`（インフラ等）/ in-step で直せない / approach 誤り | 不合格 | **コミットしない**・dirty 維持・`status=blocked`、人間タスクを `verification.review.remaining[]` と `verification.notes` に明記し**停止・報告** |
| `dependency` 起因（先行ステップの退行/前提崩れ） | 差し戻し | 該当 `dependsOn` ステップへ戻って是正（既存原則「退行は該当ステップへ戻る」） |
| out-of-scope だが妥当な指摘（in-step 成功条件は満たす） | フォローアップ | `steps[]` に新規 pending ステップ追記＋`globalLearnings` 記録。当ステップは done で前進 |

reviewer/meta は `assets/review-perspectives.md` の設計原則（implement-vs-gate で意図的に維持した experimental スタブを
「未実装」と咎めない、無言のスタブは咎める 等）を尊重する。

### 再開（`/clear` を跨ぐ mid-review）

- `/step` 開始時、`in-progress` かつ `verification.review.verdict` が未確定/`blocked` を **mid-review** として検出する。
- **clean-tree チェックを緩和**: mid-review 再開では dirty が正常（実装＋適用済み修正を保持）。`baseCommit` と照合し、
  ステップ対象ファイル外の無関係変更が混じる場合のみ停止して確認する。
- round 途中は復元せず **round 1 からやり直す**が、永続化済み `remaining[]` のキーを seed に渡すため冪等
  （キーは内容ハッシュで安定、収束はキー集合縮小の不変条件で支配される）。
- `verdict:"blocked"` のステップは既定で**読み取り報告のみ**（無人の再 `/step` で再スラッシングしない）。
  明示的な `/step <id>`、または人間が修正後に `/step-review`／state 手編集で再挑戦する。

## 設計原則

1. **実働経路を最優先** — 実際にユーザが通る経路の実バグを最初に潰す。
2. **品質ゲートを早期に緑化** — 赤いゲートは検証の信頼性を奪う。早く緑にして本物のゲートにする。
3. **広告 == 実装（正直さ）** — 各スタブは「実装する」か「experimental 明示／呼んだら throw」の二択。無言のスタブを作らない。
4. **DX・テストを機能網羅より優先** — トレードオフ時は機能数より DX とテスト網羅を取る。重インフラは需要が出るまで experimental 維持可。
5. **小さく・独立・検証可能** — 1 ステップ = 1 コミットで完結し、必ず検証できる粒度に保つ。

### implement-vs-gate 判断フレーム（スタブ機能を扱うステップ冒頭で適用）

- **Q-A**: 今すぐ動く必要のある具体的ユースケース/example があるか？
- **Q-B**: DX 価値は高いか（多くの利用者が最初に触るか）？
- **Q-C**: 実装コスト（永続化・外部プロトコル・セキュリティ）は妥当か？

いずれも Yes 寄り → **実装**。そうでなければ **experimental 維持**（本ステップは「ゲート文言＋未実装テストの確定」だけ行い、実装は将来のトリガ発生時）。

## dev-state.json スキーマ

```jsonc
{
  "project": "string",            // プロジェクト名
  "planDoc": "string",            // 仕様ファイルのパス（既定 ".claude/dev-plan.md"）
  "reviewSource": "string|null",  // 出典（レビュー/仕様文書）。任意
  "branch": "string|null",        // 作業ブランチ。null なら branch チェックをスキップ
  "planVersion": "string",        // 例 "1.0.0"
  "lastUpdated": "string",        // 例 "2026-06-26-step03"
  "baseline": {                   // 起点の現実（/plan-init が記録）
    "commit": "string",
    "typecheck": "string", "test": "string", "lint": "string", "build": "string"
    // gateCommands に存在するキーだけ。値は実行結果の要約文字列
  },
  "gateCommands": {               // 全ステップ共通の回帰チェック（存在するキーだけ実行される）
    "typecheck": "string",        // 例 "pnpm typecheck"
    "test": "string",             // 例 "pnpm test"
    "lint": "string",             // 例 "pnpm lint"
    "build": "string"             // 例 "cd examples/app && pnpm build"
  },
  "review": {                     // 階層的レビューゲート設定（全ステップ共通）。省略 or enabled:false で従来挙動
    "enabled": true,
    "maxRounds": 3,               // Layer1+Layer2 反復上限。超過＋未収束は blocked
    "blockOn": "must",            // "must" | "ask"。この重大度以上の未解決で収束失敗
    "autoFix": true,              // true: in-step ＆ requiresHuman=false の指摘を自動修正して再レビュー
    "perspectives": [             // 起動する観点（assets/review-perspectives.md のキー）。空=カタログ全観点
      "principles", "architecture", "code-quality", "correctness", "testing",
      "security", "performance", "ui-ux", "documentation", "ai-generated"
    ]
  },
  "currentStep": 1,               // 次に着手する pending ステップの id
  "steps": [
    {
      "id": 1,
      "phase": "string|null",     // 任意のフェーズ名
      "title": "string",
      "reviewIds": ["string"],    // 任意（出典のレビュー ID 等）
      "dependsOn": [0],           // 先行が必要なステップ id
      "goal": "string",           // 1 行ゴール
      "status": "pending",        // pending|in-progress|done|deferred|blocked
      "startedAt": "string|null",
      "completedAt": "string|null",
      "commit": "string|null",    // git コミットハッシュ。自分のコミットに自分のハッシュは埋め込めないため、
                                  // このステップのコミット時点では null。次ステップ着手時(手順7)に HEAD を backfill する
      "verification": {           // 完了時に記録（gateCommands のキー＋notes）
        "typecheck": "string", "test": "string", "lint": "string", "build": "string",
        "notes": "string",
        "review": {               // レビューゲート結果。enabled=false / 未実施なら null
          "baseCommit": "string", // git diff <baseCommit> の基点。着手時に記録（再開の鍵）
          "rounds": 0,            // 実際の反復回数
          "verdict": "string",    // "clean" | "blocked" | "out-of-scope-followups"
          "remaining": [          // 未解決指摘（verdict!=clean のとき）
            {
              "key": "string",            // 安定ハッシュ（オシレーション検知用）
              "severity": "string",       // "must" | "ask" | "suggestion" | "nits"
              "scope": "string",          // "in-step" | "out-of-scope" | "dependency"
              "requiresHuman": false,     // true=人間対応必須（インフラ設定/鍵/外部連携 等）
              "file": "string",           // "path:line"
              "summary": "string",
              "fromPerspective": ["string"]
            }
          ],
          "followups": [          // 別ステップ化した out-of-scope 指摘（steps[] と相互参照）
            { "asNewStepId": 0, "summary": "string" }
          ],
          "summary": "string"     // 人間向け 1〜2 行サマリ
        }
      },
      "learnings": ["string"]     // このステップで得た知見
    }
  ],
  "globalLearnings": ["string"]   // ステップ横断の地雷・パターン（/step が随時追記）
}
```

## dev-plan.md 構造

1. ヘッダ: プロジェクト名 / 出典 / 起点コミット / 作業ブランチ / 計画バージョン。
2. **§0 実行プロトコル**（上記の 12+1。レビューゲートを含む）と「起動方法（`/step` / `/step-review`）」。
3. **品質ゲート表**（gateCommands と起点/不変条件）。
4. **設計原則** ＋ **implement-vs-gate フレーム**。
5. **フェーズ & ステップ一覧表**（id / title / reviewIds / 区分 / 既定方針）。
6. **各ステップ詳細**（§1, §2, …）: ゴール / 背景 / 成功条件（チェックリスト）/ 検証方法 / 想定変更ファイル(`file:line`) / 依存 / コミットメッセージ雛形。

テンプレートは `assets/dev-plan.template.md` と `assets/dev-state.template.json` を参照。

## 運用のコツ

- **1 セッション 1 ステップ**を基本に、ステップ間で `/clear` してよい（state から復元できる）。
- **コミット規律**: state（`dev-state.json`）の更新は**コミット前に最終形まで一括**で行い、`currentStep`/`lastUpdated` を
  コミット後に回さない。コミットは `git add -A` で `dev-state.json`＋`planDoc`＋実装を**まとめてステージ**し、直後に
  `git status` で clean を確認する（`dev-state.json`/`dev-plan.md` が未コミットで残るのがコミット漏れの典型パターン）。
- 着手前に必ず `file:line` を現物で再確認する（計画は時点情報）。
- 退行を見つけたら新ステップを足すより、**該当ステップへ戻って是正**する（dependsOn の前提を守る）。
- 重い/外部依存の機能は焦って実装せず implement-vs-gate で判断し、`deferred` を恐れない。
- レビューゲートが `blocked` を返したら、`remaining[]` の人間タスク（`requiresHuman`）を片付けてから
  `/step-review` で再チェックし、緑になったら `/step <id>` で確定する。reviewer に直させず**人間が直す**前提のものは無理に autoFix しない。
- breadth レビューは観点数だけ並行起動されるためコストが上がる。軽いステップは `perspectives[]` を絞る／`maxRounds` を下げる。
