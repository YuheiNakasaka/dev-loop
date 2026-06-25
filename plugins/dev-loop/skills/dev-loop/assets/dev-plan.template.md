# {{PROJECT}} 開発計画 — {{SOURCE_SUMMARY}}

- **出典**: {{REVIEW_SOURCE}}
- **起点コミット**: `{{BASELINE_COMMIT}}`
- **作業ブランチ**: `{{BRANCH}}`
- **状態ファイル**: `.claude/dev-state.json`（実行台帳。各ステップの完了状態・検証結果・学びを記録）
- **計画バージョン**: 1.0.0

このドキュメントは「**何をやるか（仕様）**」の正典。`.claude/dev-state.json` は「**何が起きたか（状態）**」の正典。
両者を併読して 1 ステップずつ確実に進める。

---

## 0. 実行プロトコル（毎ステップ必ず守る）

```
[開始前]
  1. dev-state.json を読む（planDoc/branch/gateCommands/currentStep/steps/globalLearnings を取得）
  2. 当ステップの dependsOn が全て status=done かつ verification 緑か確認（未完/退行は先に是正）
  3. globalLearnings で既知の地雷を再確認
  4. 当ステップの「ゴール / 成功条件 / 検証方法」を本書で再確認
[実行中]
  5. dev-state.json の当ステップを status=in-progress, startedAt にセット
[完了時]
  6. 「検証方法」＋ 4 つの品質ゲートを実行し退行ゼロを確認
  7. dev-state.json に status=done, completedAt, commit, verification, learnings を記録
  8. git commit（1 ステップ = 1 コミット）
  9. currentStep を次へ、lastUpdated を更新
```

### 起動方法（コンテキストを毎回クリアして進める運用）

状態を本書 ＋ `.claude/dev-state.json` に外出ししているため `/clear` でコンテキストを消しても継続できる。

- `/step` … 次の pending（in-progress があれば再開）を進める
- `/step 3` … 特定ステップを指定して進める
- `/plan-status` … 進捗を確認する

### 品質ゲート（全ステップ共通の回帰チェック）

| ゲート | コマンド | 起点 | 不変条件 |
|---|---|---|---|
| 型 | `{{GATE_TYPECHECK}}` | {{BASELINE_TYPECHECK}} | 常に PASS |
| テスト | `{{GATE_TEST}}` | {{BASELINE_TEST}} | 常に PASS。件数は増える方向 |
| Lint | `{{GATE_LINT}}` | {{BASELINE_LINT}} | 緑化以降は常に PASS |
| ビルド | `{{GATE_BUILD}}` | {{BASELINE_BUILD}} | 常に PASS |

---

## 設計原則

1. **実働経路を最優先で正す** — 実際に動く経路の実バグを最初に潰す。
2. **品質ゲートを早期に緑化** — 赤いゲートは検証の信頼性を奪う。
3. **広告 == 実装（正直さ）** — 各スタブは「実装する」か「experimental 明示／呼んだら throw」の二択。無言のスタブを根絶。
4. **DX とテストを機能網羅より優先** — トレードオフ時は機能数より DX とテスト網羅を取る。
5. **小さく・独立コミット・検証可能** — 1 ステップ = 1 コミットで完結し、必ず検証できる粒度に保つ。

### implement-vs-gate 判断フレーム

- **Q-A**: 今すぐ動く必要のある具体的ユースケースがあるか？
- **Q-B**: DX 価値は高いか？
- **Q-C**: 実装コスト（永続化・外部プロトコル・セキュリティ）は妥当か？

いずれも Yes 寄り → **実装**。そうでなければ **experimental 維持**（ゲート文言＋未実装テストの確定のみ）。

---

## フェーズ & ステップ一覧

| Phase | Step | タイトル | 出典 ID | 区分 | 既定方針 |
|---|---|---|---|---|---|
| {{PHASE}} | 1 | {{STEP_TITLE}} | — | P0 | 実施 |
| | … | … | | | |

---

## ステップ詳細

### Step 1 — {{STEP_TITLE}}
- **ゴール**: {{STEP_GOAL}}
- **背景**: {{STEP_BACKGROUND}}
- **成功条件**:
  - [ ] {{SUCCESS_1}}
  - [ ] {{SUCCESS_2}}
- **検証方法**: {{STEP_VERIFICATION}}
- **想定変更ファイル**: `{{FILE}}:{{LINE}}`（計画時点。着手時に現物で再確認）
- **依存**: なし
- **コミットメッセージ**: `Step 1: {{STEP_TITLE}}`

<!-- 以降 Step 2, 3, … を同フォーマットで追加 -->
