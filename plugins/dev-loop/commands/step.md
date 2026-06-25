---
description: Execute one step of the active dev-loop plan (driven by .claude/dev-state.json)
argument-hint: "[step number — optional; defaults to the next pending step]"
---

dev-loop の計画を **1 ステップだけ** 進めてください。
正典は会話履歴ではなく、次の 2 ファイル（設定値もこの 2 つから読む。prose にハードコードしない）:

- 仕様（何をやるか）: 既定 `.claude/dev-plan.md`（実際のパスは state の `planDoc`）
- 状態台帳（何が起きたか・設定）: `.claude/dev-state.json`

**対象ステップ**: `$ARGUMENTS`
（空欄なら `dev-state.json` の `currentStep`＝次の `status: "pending"` を対象にする。
`status: "in-progress"` のステップがあれば新規開始せず **それを再開** する。）

プロトコルの詳細は dev-loop スキル（`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/SKILL.md`）が単一真実源。要点:

1. **`.claude/dev-state.json` を読む。** 無ければ「先に `/plan-init` を実行してください」と案内して**停止**する。
   ここから設定を取得する: `planDoc`・`branch`・`gateCommands`・`currentStep`・`steps[]`・`globalLearnings`。
2. **対象ステップを確定**する（`$ARGUMENTS` 指定を最優先。空なら `currentStep`。`in-progress` があれば再開）。
3. **前提を確認**する。`branch` が設定されていれば、そのブランチ上にいること・作業ツリーがクリーンなことを確認する
   （違えば報告して**停止**。勝手に commit/stash しない）。`branch` が空ならこのチェックはスキップ。
4. 対象ステップの `dependsOn` が全て `status: "done"` かつ `verification` が緑であることを確認する。
   未完・退行があれば **先にそれを是正** してから進む（同じ失敗の再発防止）。
5. `globalLearnings` を読み、このプロジェクト固有の既知の地雷を回避する。
6. `planDoc` で対象ステップの「ゴール / 成功条件 / 検証方法」を読む。
   記載の `file:line` は計画時点の値なので、**必ず現物コードで再確認** してから変更する。
7. `dev-state.json` の当該ステップを `status: "in-progress"`、`startedAt` に更新する。
8. **実装する。** 完了したら「検証方法」＋ `gateCommands` に **定義されている各ゲートだけ** を実行する
   （例: `typecheck` / `test` / `lint` / `build`。定義のないキーは飛ばす）。退行ゼロを確認する。
9. `dev-state.json` の当該ステップを更新する
   （`status: "done"`、`completedAt`、`commit`、`verification` 結果、`learnings`）。
   方針判断で見送る場合は `status: "deferred"` ＋ 理由を `verification.notes` に残す。
   将来の地雷になりうる発見は `globalLearnings` にも追記する。
10. **1 ステップ = 1 コミット**。メッセージは `Step <id>: <title>`（`reviewIds` があれば `(IDs)` を付記）、
    末尾に Co-Authored-By トレーラを付ける。`branch` 未設定なら現在のブランチにそのままコミットする。
11. `dev-state.json` の `currentStep` を次へ進め、`lastUpdated` を更新する。

着手前に、対象ステップの「ゴール / 成功条件 / 検証方法」と、確認した前提（`dependsOn` の状態・現物コードとの差分）を
**簡潔に提示してから** 実装に入ること。1 回の `/step` で進めるのは 1 ステップだけ。終わったら次は新しい `/step`
（コンテキストを `/clear` しても state から再開できる）。
