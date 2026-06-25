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
| `/plan-init [ゴール/仕様/レビュー文書]` | `dev-plan.md` + `dev-state.json` を生成。ゲート自動検出・ベースライン記録。 |
| `/step [id]` | 1 ステップ実行（依存確認→実装→ゲート→state 更新→1 コミット）。空なら次の pending。 |
| `/plan-status` | 進捗の読み取り専用ビュー。 |

## 実行プロトコル（`/step` が毎回守る 10 + 1）

```
[開始前]
  1. dev-state.json を読み、設定(planDoc/branch/gateCommands/currentStep/steps/globalLearnings)を取得
  2. 対象ステップ確定（引数 > currentStep。in-progress があれば再開）
  3. branch があればそのブランチ上＆作業ツリー clean を確認（違えば停止）
  4. dependsOn が全て done＋検証緑か確認。未完/退行は先に是正
  5. globalLearnings で既知の地雷を確認
  6. planDoc で「ゴール/成功条件/検証方法」を読み、file:line を現物コードで再確認
[実行中]
  7. 当ステップを status=in-progress, startedAt に更新
  8. 実装 → 「検証方法」＋ gateCommands の各ゲートを実行（定義キーのみ）。退行ゼロを確認
[完了時]
  9. status=done(見送りは deferred＋理由), completedAt, commit, verification, learnings を記録
 10. 1 ステップ = 1 コミット（`Step <id>: <title> (reviewIds)` ＋ Co-Authored-By）
 11. currentStep を次へ、lastUpdated を更新
```

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
      "commit": "string|null",    // git コミットハッシュ
      "verification": {           // 完了時に記録（gateCommands のキー＋notes）
        "typecheck": "string", "test": "string", "lint": "string", "build": "string",
        "notes": "string"
      },
      "learnings": ["string"]     // このステップで得た知見
    }
  ],
  "globalLearnings": ["string"]   // ステップ横断の地雷・パターン（/step が随時追記）
}
```

## dev-plan.md 構造

1. ヘッダ: プロジェクト名 / 出典 / 起点コミット / 作業ブランチ / 計画バージョン。
2. **§0 実行プロトコル**（上記の 10+1）と「起動方法（`/step`）」。
3. **品質ゲート表**（gateCommands と起点/不変条件）。
4. **設計原則** ＋ **implement-vs-gate フレーム**。
5. **フェーズ & ステップ一覧表**（id / title / reviewIds / 区分 / 既定方針）。
6. **各ステップ詳細**（§1, §2, …）: ゴール / 背景 / 成功条件（チェックリスト）/ 検証方法 / 想定変更ファイル(`file:line`) / 依存 / コミットメッセージ雛形。

テンプレートは `assets/dev-plan.template.md` と `assets/dev-state.template.json` を参照。

## 運用のコツ

- **1 セッション 1 ステップ**を基本に、ステップ間で `/clear` してよい（state から復元できる）。
- 着手前に必ず `file:line` を現物で再確認する（計画は時点情報）。
- 退行を見つけたら新ステップを足すより、**該当ステップへ戻って是正**する（dependsOn の前提を守る）。
- 重い/外部依存の機能は焦って実装せず implement-vs-gate で判断し、`deferred` を恐れない。
