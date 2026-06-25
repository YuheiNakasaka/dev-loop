---
description: Scaffold a dev-loop plan + state ledger from a goal or spec/review document
argument-hint: "[ゴール文 / 仕様 / レビュー文書のパス]"
---

dev-loop ハーネスを初期化します。最終成果物は次の 2 ファイル:

- `.claude/dev-plan.md` … 仕様（何をやるか）。順序付きステップの正典。
- `.claude/dev-state.json` … 状態台帳（何が起きたか・設定）。`/step` が読み書きする。

入力: `$ARGUMENTS`（自由記述のゴール、または review/spec 文書へのパス）。空なら何を達成したいかをユーザに尋ねる。

雛形は dev-loop スキル（`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/SKILL.md`）と
`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/assets/`（`dev-plan.template.md` / `dev-state.template.json`）にある。
これらを読んでスキーマに**厳密準拠**で生成すること。

手順:

1. **冪等性チェック**: `.claude/dev-state.json` が既に存在すれば内容を要約し、上書き / 追記 / 中止をユーザに確認する
   （勝手に上書きしない）。
2. **入力収集**:
   - プロジェクト名 = `package.json` の `name`、無ければリポジトリのディレクトリ名。
   - 出典 = `$ARGUMENTS`。パスならそのファイルを読み、自由記述ならそのまま `reviewSource` の代わりにゴール要約として扱う。
3. **品質ゲート自動検出**: まず `package.json` の `scripts` を見て `typecheck`/`test`/`lint`/`build`（相当）を探す。
   無ければ `Makefile` / `Cargo.toml` / `go.mod` / `pyproject.toml` / `composer.json` 等から候補を推定する。
   検出した候補を提示し、**AskUserQuestion で確認・修正**してもらう（部分的でも空でも可。実行可能な形のコマンド文字列で `gateCommands` に格納）。
   - **モノレポ注意**: 対象パッケージに `cd <dir> &&` を付ける等、リポジトリ root から実行できる文字列にする。
4. **作業ブランチ**: 既定 `dev-loop/<plan-slug>` を提案し確認する（ユーザが「現ブランチで」と言えば `branch` は空でよい）。
   合意できたらブランチを作成/切替する。**ベースライン記録**: 現 HEAD コミットハッシュを `baseline.commit` に入れ、
   `gateCommands` を 1 回ずつ実行して結果を `baseline` に保存する（失敗してもよい＝起点の現実を正直に記録する）。
5. **ステップ分解**: ゴールを順序付きステップへ分解する。各ステップは
   `id` / `phase`(任意) / `title` / `reviewIds`(任意) / `dependsOn` / `goal` / `status: "pending"` を持つ。
   設計原則を守る: **小さく独立・1 ステップ 1 コミット・必ず検証可能・実働経路を最優先・広告==実装（無言のスタブを作らない）**。
   `dev-plan.md` には各ステップの「ゴール / 成功条件（チェックリスト）/ 検証方法 / 想定変更ファイル(`file:line`) / 依存 / コミットメッセージ雛形」を書く。
   `dev-state.json` の `steps[]` は同じ id でメタを持ち（`verification`/`learnings`/`commit` は未着手のため null/空）、`currentStep` は最初の pending に、`globalLearnings: []` で開始する。
6. 生成後、ステップ数・ゲート・ブランチ・ベースライン結果を要約し、「**`/step` で 1 ステップ目を開始**、`/plan-status` で進捗確認」と案内する。
   この `plan-init` 自体はコミットしない（最初の `/step` 群でコミットしていく。計画ファイルをコミットするかはユーザに委ねる）。
