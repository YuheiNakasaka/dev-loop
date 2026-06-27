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
`status: "in-progress"` のステップがあれば新規開始せず **それを再開** する。
再開時、当該ステップが `verification.review` を持ち `verdict` が未確定/`blocked` なら **mid-review 再開** として扱う
＝ 作業ツリーの dirty を許容し、`verification.review.baseCommit` を基点にレビューゲートから続行する。）

プロトコルの詳細は dev-loop スキル（`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/SKILL.md`）が単一真実源。要点:

1. **`.claude/dev-state.json` を読む。** 無ければ「先に `/plan-init` を実行してください」と案内して**停止**する。
   ここから設定を取得する: `planDoc`・`branch`・`gateCommands`・`review`・`currentStep`・`steps[]`・`globalLearnings`。
2. **対象ステップを確定**する（`$ARGUMENTS` 指定を最優先。空なら `currentStep`。`in-progress` があれば再開）。
3. **前提を確認**する。`branch` が設定されていれば、そのブランチ上にいること・作業ツリーがクリーンなことを確認する
   （違えば報告して**停止**。勝手に commit/stash しない）。`branch` が空ならこのチェックはスキップ。
   **例外**: mid-review 再開（手順 2）では作業ツリーの dirty は正常。`verification.review.baseCommit` と照合し、
   ステップ対象ファイル外の無関係変更が混じる場合のみ停止して確認する。
4. 対象ステップの `dependsOn` が全て `status: "done"` かつ `verification` が緑であることを確認する。
   未完・退行があれば **先にそれを是正** してから進む（同じ失敗の再発防止）。
5. `globalLearnings` を読み、このプロジェクト固有の既知の地雷を回避する。
6. `planDoc` で対象ステップの「ゴール / 成功条件 / 検証方法」を読む。
   記載の `file:line` は計画時点の値なので、**必ず現物コードで再確認** してから変更する。
7. `dev-state.json` の当該ステップを `status: "in-progress"`、`startedAt` に更新する。
   `review.enabled` なら、このとき `verification.review.baseCommit` に**現 HEAD コミットハッシュ**を記録する
   （レビュー対象 diff の基点。`/clear` を跨ぐ再開の鍵）。
8. **実装する。** 完了したら「検証方法」＋ `gateCommands` に **定義されている各ゲートだけ** を実行する
   （例: `typecheck` / `test` / `lint` / `build`。定義のないキーは飛ばす）。退行ゼロを確認する。

   --- ここから **階層的レビューゲート**（`review.enabled` のときだけ。`false`/不在なら手順 9〜10 を飛ばして 11 へ） ---
   プロトコルの単一真実源は dev-loop スキルの「階層的レビューゲート」節。round `r = 1..review.maxRounds` で回す:
9. **Layer 1（breadth）。** `git diff <baseCommit>`（作業ツリー＋staged）を対象に、`review.perspectives[]`
   （空ならカタログ全観点。観点定義は `${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/assets/review-perspectives.md`）の
   **各観点を `dev-loop:dev-reviewer` サブエージェントとして並行起動**する（Agent ツールを 1 メッセージで複数同時呼び出し）。
   各 reviewer には「担当観点＋その焦点チェックリスト・`baseCommit`・ステップのゴール/成功条件/想定変更ファイル・
   リポジトリパス・（r≥2 なら）前ラウンドの未解決キー」を渡す。ラベル＋安定キー付き指摘を収集する。
10. **Layer 2（meta＝レビュー結果のレビュー）＋ 収束判定。** `dev-loop:dev-review-meta` を 1 回起動し、Layer 1 全指摘＋
    ステップ文脈（ゴール/成功条件/`dependsOn`/`globalLearnings`/`baseCommit`/`review` 設定）を渡す。
    返ってくる `verdict`・`openFindings`（`scope`/`requiresHuman`/`autoFixable` 付き）・`followups`・`coverageGaps` を使って:
    - `coverageGaps` があれば不足観点の reviewer を追加起動して Layer 2 をやり直す。
    - `review.blockOn`（既定 `must`）以上の未解決が**無い** → **収束**。手順 11 へ。
    - 未解決があり `review.autoFix=true` かつ **全て `autoFixable`** → **メインエージェント自身が修正**（reviewer は編集しない）→
      `gateCommands` を再実行（退行ゼロ確認）→ `r++` で手順 9 へ戻る。
    - **収束失敗**（次のいずれか）→ 手順 11 の blocked 分岐へ:
      ① 未解決に `requiresHuman:true` / in-step で直せない / approach 誤りがある、
      ② `dependency` 起因（先行ステップへ差し戻すべき）、
      ③ オシレーション（未解決 must のキー集合が前ラウンドの真部分集合に縮小しない）、
      ④ `maxRounds` 超過、⑤ reviewer がエラー/パース不能（「指摘なし」と誤判定しない）。
    --- レビューゲートここまで ---

11. **state を更新する。**
    - **収束した（または `review.enabled=false`）場合**: `status: "done"`（方針見送りは `status: "deferred"` ＋ 理由を
      `verification.notes`）、`completedAt`、`verification`（ゲート結果＋ `review` 結果: `baseCommit`/`rounds`/`verdict`/
      `remaining`(空)/`followups`/`summary`）、`learnings` を記録。out-of-scope の妥当な指摘は `steps[]` に新規 pending
      ステップとして追記し（`followups[].asNewStepId` に相互参照）、必要なら `globalLearnings` にも追記する。
    - **収束失敗（blocked）の場合**: `status: "blocked"`、`verification.review`（`verdict: "blocked"`・`remaining[]` に
      人間タスク＝ `requiresHuman` や差し戻し先を明記）と `verification.notes` を記録する。**コミットしない・`currentStep` を
      進めない・作業ツリーは dirty のまま**にして、人間がやるべきことを提示して**停止・報告**する。`dependency` 起因なら
      該当 `dependsOn` ステップへ戻って是正する。
12. **収束時のみ 1 ステップ = 1 コミット。** メッセージは `Step <id>: <title>`（`reviewIds` があれば `(IDs)` を付記）、
    レビュー実施時は末尾に `[review: <verdict>, <rounds>r]` を付記し、さらに末尾に Co-Authored-By トレーラを付ける。
    `branch` 未設定なら現在のブランチにそのままコミットする。
13. **収束時のみ** `dev-state.json` の `currentStep` を次へ進め、`lastUpdated` を更新する。

着手前に、対象ステップの「ゴール / 成功条件 / 検証方法」と、確認した前提（`dependsOn` の状態・現物コードとの差分）を
**簡潔に提示してから** 実装に入ること。終了時は、レビューを実施したら最終 `verdict`・round 数・未解決（あれば人間タスク）を
**簡潔に報告**する。1 回の `/step` で進めるのは 1 ステップだけ。終わったら次は新しい `/step`
（コンテキストを `/clear` しても state から再開できる。`blocked` は人間対応後に `/step-review` で再チェック → `/step <id>` で確定）。
