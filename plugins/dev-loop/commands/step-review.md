---
description: Run the dev-loop hierarchical review gate against the current working tree (read-only — no fixes, no commit, no state advance)
argument-hint: "[step number — optional; defaults to currentStep / the in-progress step]"
---

dev-loop の**階層的レビューゲートだけ**を、現在の作業ツリーに対して 1 回だけ回します。
**読み取り専用**——コードを編集せず、コミットせず、`currentStep` も進めません。
手動修正後の再チェックや、コミット前の dry-run に使います。

正典は会話履歴ではなく次の 2 ファイル（設定値もここから読む。prose にハードコードしない）:

- 状態台帳・設定: `.claude/dev-state.json`（`review` 設定・`steps[]`・`globalLearnings` を読む）
- レビュープロトコルの単一真実源: dev-loop スキル（`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/SKILL.md`）の「階層的レビューゲート」節
- 観点カタログ: `${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/assets/review-perspectives.md`

**対象ステップ**: `$ARGUMENTS`（空欄なら `in-progress` のステップ、無ければ `currentStep`）。

手順:

1. **`.claude/dev-state.json` を読む。** 無ければ「先に `/plan-init` を実行してください」と案内して**停止**する。
   `review.enabled=false`（または `review` 不在）なら「レビューは無効です」と伝えて停止する。
2. **対象ステップを確定**し、その「ゴール / 成功条件 / 想定変更ファイル / `dependsOn`」を `planDoc` で確認する。
3. **レビュー基点 `baseCommit` を決める。**
   - 対象ステップに `verification.review.baseCommit` があればそれを使う。
   - 無ければ「直近の done ステップの `commit`（無ければステップ着手前の妥当な基点）」を基点候補として提示し、
     `git diff <baseCommit>` が現作業ツリーの変更を捉えていることを確認する。
4. **Layer 1（breadth）。** `review.perspectives[]`（空ならカタログ全観点）の各観点を `dev-loop:dev-reviewer` で
   **並行起動**する（Agent ツールの複数同時呼び出し）。各 reviewer に観点・`baseCommit`・ステップ文脈・リポジトリパスを渡す。
5. **Layer 2（meta）。** `dev-loop:dev-review-meta` を 1 回起動し、Layer 1 全指摘＋ステップ文脈＋`review` 設定を渡して
   `verdict`・`openFindings`・`followups`・`coverageGaps` を得る。`coverageGaps` があれば不足観点を追加起動して 1 度だけやり直す。
   **自動修正はしない**（このコマンドは読み取り専用なので 1 パスで終える）。
6. **結果を表示する**（state は既定で**書き換えない**）:
   - **総合 verdict**: `clean` / `needs-fix` / `blocked` / `out-of-scope-followups`。
   - **未解決指摘**: `[must]/[ask]` を `file:line` ＋要約 ＋ `scope`（in-step/out-of-scope/dependency）＋ `requiresHuman` 付きで列挙。
   - **フォローアップ候補**（out-of-scope）と **棄却された指摘（偽陽性）** の要約。
   - **次アクションの助言**: clean なら「`/step <id>` で確定可能」、blocked なら「人間タスク（`requiresHuman`）を片付けてから再チェック」、
     needs-fix なら「`/step <id>` で autoFix ループに載せる」。
7. **任意**: ユーザが明示的に「記録して」と指示した場合に限り、対象ステップの `verification.review` に結果を書き込む。
   既定は表示のみで `dev-state.json` を変更しない。**コミットや `currentStep` 前進は一切しない。**
