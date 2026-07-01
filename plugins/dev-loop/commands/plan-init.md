---
description: Scaffold a dev-loop plan + state ledger from a goal or spec/review document
argument-hint: "[ゴール文 / 仕様 / レビュー文書のパス]"
---

dev-loop ハーネスを初期化します。最終成果物は次の 2 ファイル:

- `.claude/dev-plans/<date>-<slug>.md` … 仕様（何をやるか）。順序付きステップの正典。**実行ごとに新規作成し、既存の計画ファイルは上書きしない**（過去の計画＝意思決定を保持）。アクティブな計画は `dev-state.json` の `planDoc` が指す。
- `.claude/dev-state.json` … 状態台帳（何が起きたか・設定）。`/step` が読み書きする。

入力: `$ARGUMENTS`（自由記述のゴール、または review/spec 文書へのパス）。空なら何を達成したいかをユーザに尋ねる。

雛形は dev-loop スキル（`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/SKILL.md`）と
`${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/assets/`（`dev-plan.template.md` / `dev-state.template.json`）にある。
これらを読んでスキーマに**厳密準拠**で生成すること。

手順:

1. **冪等性チェック**: `.claude/dev-state.json` が既に存在すれば内容を要約し、上書き / 追記 / 中止をユーザに確認する
   （勝手に上書きしない）。※この確認は `dev-state.json`（設定・進捗）に対してのみ。**計画ファイルは常に新規作成し、既存を上書きしない**（手順 5 でパスを生成）。
2. **入力収集**:
   - プロジェクト名 = `package.json` の `name`、無ければリポジトリのディレクトリ名。
   - 出典 = `$ARGUMENTS`。パスならそのファイルを読み、自由記述ならそのまま `reviewSource` の代わりにゴール要約として扱う。
3. **品質ゲート自動検出**: まず `package.json` の `scripts` を見て `typecheck`/`test`/`lint`/`build`（相当）を探す。
   無ければ `Makefile` / `Cargo.toml` / `go.mod` / `pyproject.toml` / `composer.json` 等から候補を推定する。
   検出した候補を提示し、**AskUserQuestion で確認・修正**してもらう（部分的でも空でも可。実行可能な形のコマンド文字列で `gateCommands` に格納）。
   - **モノレポ注意**: 対象パッケージに `cd <dir> &&` を付ける等、リポジトリ root から実行できる文字列にする。
   - **階層的レビューゲート設定**（`review`）: 既定値を提示し、**AskUserQuestion で確認・調整**してもらう
     （`enabled: true` / `maxRounds: 3` / `blockOn: "must"` / `autoFix: true` / `perspectives`= 既定 10 観点）。
     観点は `${CLAUDE_PLUGIN_ROOT}/skills/dev-loop/assets/review-perspectives.md` のキーから選ぶ
     （既定: `principles`/`architecture`/`code-quality`/`correctness`/`testing`/`security`/`performance`/`ui-ux`/`documentation`/`ai-generated`。
     空配列＝全観点）。レビュー不要なら `enabled: false` でよい（その場合 `/step` は従来どおり動く）。
4. **作業ブランチ**: 既定 `dev-loop/<slug>` を提案し確認する（`<slug>` は手順 5 で定義するゴール由来スラッグと同一。ユーザが「現ブランチで」と言えば `branch` は空でよい）。
   合意できたらブランチを作成/切替する。**ベースライン記録**: 現 HEAD コミットハッシュを `baseline.commit` に入れ、
   `gateCommands` を 1 回ずつ実行して結果を `baseline` に保存する（失敗してもよい＝起点の現実を正直に記録する）。
5. **計画ファイルのパス生成 → ステップ分解**:
   - **パス生成（上書き禁止）**:
     - `<date>` = 今日（`YYYY-MM-DD`）。`<slug>` = ゴールを小文字化し、非英数字の連続を `-` に畳んで前後の `-` を除去（~50 字に切り詰め。空になるなら `plan`）。この `<slug>` は手順 4 のブランチ名 `dev-loop/<slug>` にも同じ値を用いる。
     - 対象パス = `.claude/dev-plans/<date>-<slug>.md`。`.claude/dev-plans/` が無ければ作成する。
     - **上書き禁止**: 初回は連番なし。同名が既にあれば `-2` から始めて `-3`… と、空いている名前になるまで連番を増やす（既存の計画ファイルは絶対に開かない・上書きしない）。
   - **planDoc 設定**: 生成した計画ファイルのパスを、そのまま `dev-state.json` の `planDoc`（テンプレの `{{PLAN_DOC}}`）へ設定する。
   - **ステップ分解**: ゴールを順序付きステップへ分解する。各ステップは
     `id` / `phase`(任意) / `title` / `reviewIds`(任意) / `dependsOn` / `goal` / `status: "pending"` を持つ。
     設計原則を守る: **小さく独立・1 ステップ 1 コミット・必ず検証可能・実働経路を最優先・広告==実装（無言のスタブを作らない）**。
     計画ファイルには各ステップの「ゴール / 成功条件（チェックリスト）/ 検証方法 / 想定変更ファイル(`file:line`) / 依存 / コミットメッセージ雛形」を書く。
     `dev-state.json` の `steps[]` は同じ id でメタを持ち（`verification`/`learnings`/`commit` は未着手のため null/空。
     `verification.review` は `null` で初期化）、`currentStep` は最初の pending に、`globalLearnings: []` で開始する。
     手順 3 で確定した `review` 設定ブロックも `gateCommands` の隣に格納する（スキーマは SKILL.md / テンプレート準拠）。
6. 生成後、ステップ数・ゲート・**レビュー設定（有効/観点数/maxRounds）**・ブランチ・ベースライン結果を要約し、
   「**`/step` で 1 ステップ目を開始**、`/step-review` でレビューだけ確認、`/plan-status` で進捗確認」と案内する。
   この `plan-init` 自体はコミットしない。生成した `dev-plan.md` / `dev-state.json` は未コミットのまま残るが、
   最初の `/step` が前提チェックでこれら制御ファイルの未コミットを許容し、ステップ完了時の `git add -A` で
   実装変更とまとめて取り込む（＝計画ファイルは初回 `/step` のコミットに必ず含まれる）。
