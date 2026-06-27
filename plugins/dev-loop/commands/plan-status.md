---
description: Show dev-loop progress from .claude/dev-state.json (read-only)
argument-hint: ""
---

dev-loop の進捗を表示します。**読み取り専用**——`dev-state.json` を一切変更しないこと。

1. `.claude/dev-state.json` を読む。無ければ「まだ初期化されていません。`/plan-init` を実行してください」と案内して終了。
2. ヘッダを表示: `project` / `branch` / `planVersion` / `lastUpdated` / `currentStep`。
3. **ステップ表**を表示（`steps[]` を id 昇順で）:

   | id | phase | title | status | review | commit |
   |----|-------|-------|--------|--------|--------|

   status は `pending` / `in-progress` / `done` / `deferred` / `blocked` をそのまま。
   `done`/`deferred` の行には short commit（あれば）を出す。
   `review` 列は `verification.review` があれば `<verdict> (<rounds>r)`（例 `clean (2r)` / `blocked (3r)`）、
   レビュー未実施/無効なら `—` を出す。
4. **サマリ**: 各 status の件数（例: done 12 / pending 8 / in-progress 1 / deferred 0 / blocked 0）と完了率（done / 総数）を出す。
5. **ゲート状況**: `baseline` と、直近 `done` ステップの `verification` を並べて、退行（緑→赤）がないかを一目で示す。
6. **レビュー状況**: `blocked` または未収束のステップがあれば、その `verification.review.remaining[]` の
   `[must]`／`requiresHuman` 指摘を `file:line` ＋要約で列挙し、「人間対応 → `/step-review` で再チェック → `/step <id>` で確定」を案内する。
   `followups[]`（別ステップ化した out-of-scope 指摘）があれば対応 step id とともに示す。
7. **globalLearnings**: 件数と、直近 3 件を箇条書きで出す（多ければ省略を明記）。
8. **次アクション**: `blocked` があれば最優先で「人間対応が必要なステップ」を示す。
   in-progress があれば「`/step` で再開」、無ければ「`/step` で次のステップ（id=`currentStep`）」を案内する。
