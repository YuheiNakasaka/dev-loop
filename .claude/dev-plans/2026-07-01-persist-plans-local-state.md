# dev-loop 開発計画 — 計画の永続化（上書き禁止）と dev-state.json の git 管理除外

- **出典**: ユーザ要望（自由記述ゴール）: 「計画ファイルが常に上書きされると重要な意思決定が残りにくい。計画は上書きせずディレクトリに全て永続化して git 管理したい。また dev-state.json は多人数開発でコンフリクトの原因になるので git 管理外にしたい」
- **起点コミット**: `880e42b`
- **作業ブランチ**: `dev-loop/persist-plans-local-state`
- **状態ファイル**: `.claude/dev-state.json`（実行台帳。各ステップの完了状態・検証結果・学びを記録。**本計画で導入する新方針を先取り＝git 管理外・ローカル/開発者ごと**。`.gitignore` 済み）
- **計画バージョン**: 1.0.0

このドキュメントは「**何をやるか（仕様）**」の正典。`.claude/dev-state.json` は「**何が起きたか（状態）**」の正典。
両者を併読して 1 ステップずつ確実に進める。計画ファイルは `.claude/dev-plans/<date>-<slug>.md` に永続化され、上書きされない（アクティブな計画は state の `planDoc` が指す本ファイル）。

---

## 背景 / なぜこの変更をするか

dev-loop の計画は単一ファイル `.claude/dev-plan.md` に置かれ、`/plan-init` を実行するたびに上書きされていた。
そのため新しい計画を作ると、過去の計画（＝そこに至った重要な意思決定の記録）がファイルから消え、git 履歴を掘らないと辿れなくなる。
また状態台帳 `.claude/dev-state.json` は直近コミット `6d067e1` で「毎ステップ必ずコミットする」規律が入ったが、
currentStep・commit ハッシュ・レビュー結果など更新頻度が非常に高く、複数人が同じ PJ で開発するとマージコンフリクトの温床になる。

この計画では 2 点を変える:

1. **計画を上書きせず永続化・git 管理**: `/plan-init` は毎回 `.claude/dev-plans/<date>-<slug>.md` を新規作成し、既存計画を上書きしない。
   過去の計画はファイル＋git 履歴として残り、意思決定が失われない。アクティブな計画は state の `planDoc` が指す。
2. **`dev-state.json` を git 管理外に**: `/plan-init` が `.claude/dev-state.json` を対象リポジトリの `.gitignore` に自動追記（冪等）し、
   `/step` はコミットしない。状態はローカル/開発者ごとになり、マージコンフリクトが構造的に起きなくなる。
   トレードオフとして状態は他メンバー/別マシンへ共有されないが、進捗はコミット済みの計画ファイル＋`Step N: title` のコミット履歴から辿れる。

意思決定の詳細な根拠は decision 記録（`plan-file-versioned-not-overwritten` / `dev-state-json-git-ignored-local`）にも保存済み。

---

## 0. 実行プロトコル（毎ステップ必ず守る）

```
[開始前]
  1. dev-state.json を読む（planDoc/branch/gateCommands/review/currentStep/steps/globalLearnings を取得）
  2. 当ステップの dependsOn が全て status=done か確認（未完/退行は先に是正）
  3. globalLearnings で既知の地雷を再確認
  4. 当ステップの「ゴール / 成功条件 / 検証方法」を本書で再確認
[実行中]
  5. dev-state.json の当ステップを status=in-progress, startedAt にセット（review.enabled なら verification.review.baseCommit に現 HEAD を記録）
  6. 「検証方法」を実行し退行ゼロを確認（本 PJ は自動ゲートなし＝手動検証）
[階層的レビューゲート（review.enabled のとき。round r=1..maxRounds）]
  7. Layer1: git diff <baseCommit> を perspectives[] の dev-reviewer で並行レビュー → Layer2: dev-review-meta で裁定
  8. 収束（blockOn 以上の未解決なし）→ 9 へ。autoFix 可能な must は自動修正→再検証→再レビュー。未収束（requiresHuman/オシレーション/maxRounds 超過）は blocked
[完了時]
  9. 収束時: dev-state.json を**最終形へ**（status=done, completedAt, verification(＋review), learnings, currentStep を次へ, lastUpdated 更新）。
     ※本 PJ では dev-state.json は git 管理外＝コミットされない（ローカルのみ）。未収束は status=blocked で記録し停止・報告
 10. 収束時のみ git add -A で **計画ファイル(planDoc)＋実装** をステージ（dev-state.json は .gitignore 済みで staged されない＝意図どおり）。git commit -am や部分 add は使わない
 11. 収束時のみ git commit（1 ステップ = 1 コミット。レビュー時はメッセージ末尾に [review: <verdict>, <rounds>r] を付記）
     → 直後に git status --porcelain で clean を確認（ignored な dev-state.json は表示されない。計画ファイル/.gitignore の未コミット残差分はコミット漏れ＝追従コミットで取り込む）
```

### 起動方法（コンテキストを毎回クリアして進める運用）

状態を本書 ＋ `.claude/dev-state.json` に外出ししているため `/clear` でコンテキストを消しても継続できる。

- `/step` … 次の pending（in-progress があれば再開）を進める。**末尾に階層的レビューゲートを実行**する
- `/step 3` … 特定ステップを指定して進める
- `/step-review` … レビューゲートだけを現作業ツリーに対して回す（読み取り専用。コミット/前進しない）
- `/plan-status` … 進捗を確認する（レビュー verdict・blocked の人間タスクも表示）

### 品質ゲート（全ステップ共通の回帰チェック）

このリポジトリは md/json 中心でビルド・テスト基盤がない。**自動ゲートなし・手動検証**。
JSON を触るステップでは `python3 -m json.tool < <file>` で構文妥当性を確認する程度に留める。

| ゲート | コマンド | 起点 | 不変条件 |
|---|---|---|---|
| 型 | （なし） | N/A | — |
| テスト | （なし） | N/A | — |
| Lint | （なし） | N/A | — |
| ビルド | （なし） | N/A | — |

### 階層的レビューゲート設定（dev-state.json `review`）

各ステップ末尾で**観点別 reviewer の並行レビュー（Layer 1）→ メタレビュー（Layer 2）→ 反復**を回す。
設定: `enabled=true` / `maxRounds=3` / `blockOn=must` / `autoFix=true` /
`perspectives=[principles, architecture, correctness, documentation, ai-generated]`。
（今回の変更は docs/コマンド仕様が中心のため security・performance・ui-ux・testing・code-quality は外している。）
詳細プロトコルは dev-loop スキルの「階層的レビューゲート」節、観点定義は `assets/review-perspectives.md`。

---

## 設計原則

1. **実働経路を最優先で正す** — 実際に使われるコマンド（plan-init / step）の挙動を正しくする。
2. **広告 == 実装（正直さ）** — ドキュメントの記述と実際のコマンド挙動を一致させ、旧記述の言い残しを根絶する。
3. **小さく・独立コミット・検証可能** — 1 ステップ = 1 コミットで完結し、必ず検証（読み合わせ/scratch dry-run）できる粒度に保つ。
4. **矛盾を残さない** — `6d067e1` が入れた「state を常にコミット」方針を state について意図的に反転させるため、同コミットが触れた step.md / SKILL.md / dev-plan.template.md を漏れなく更新する。
5. **後方互換の移行を明示** — 既に dev-state.json を追跡済みの既存プロジェクト向けに `git rm --cached` を用意する。

---

## フェーズ & ステップ一覧

| Phase | Step | タイトル | 出典 ID | 区分 | 既定方針 |
|---|---|---|---|---|---|
| テンプレート | 1 | テンプレの planDoc プレースホルダ化＋状態=ローカル明記 | — | P0 | 実施 |
| スキル仕様 | 2 | SKILL.md をファイル管理方針/スキーマ/プロトコルで一貫更新 | — | P0 | 実施 |
| plan-init 実装 | 3 | plan-init に計画パス生成（上書き禁止・連番）を追加 | — | P0 | 実施 |
| plan-init 実装 | 4 | plan-init に .gitignore 自動追記＋既追跡の移行を追加 | — | P0 | 実施 |
| step 実装 | 5 | step.md のコミット規律を state 非コミットに統一 | — | P0 | 実施 |
| ドキュメント/配布 | 6 | README/plugin.json（version 0.3.0）を新挙動へ更新 | — | P1 | 実施 |

---

## ステップ詳細

### Step 1 — テンプレの planDoc プレースホルダ化＋状態=ローカル明記
- **ゴール**: `dev-state.template.json` の `planDoc` を `{{PLAN_DOC}}` プレースホルダにし、`dev-plan.template.md` のヘッダで状態ファイルを「**git 管理外（ローカル/開発者ごと）**」と明記する。
- **背景**: テンプレの `planDoc` が `.claude/dev-plan.md` 固定で、新レイアウトでは生成される全プロジェクトで誤りになる。状態ファイルの git 方針もテンプレに書かれていない。
- **成功条件**:
  - [ ] `dev-state.template.json` の `planDoc` が `{{PLAN_DOC}}` になっている
  - [ ] `dev-state.template.json` が構文的に妥当（`python3 -m json.tool` が通る）
  - [ ] `dev-plan.template.md` のヘッダ（現 L6 / L9 付近）に「状態ファイルは git 管理外・ローカル」の旨が書かれ、進捗はコミット履歴で辿れる旨を補足
- **検証方法**: `python3 -m json.tool < plugins/dev-loop/skills/dev-loop/assets/dev-state.template.json`、テンプレ 2 ファイルの差分読み合わせ
- **想定変更ファイル**: `plugins/dev-loop/skills/dev-loop/assets/dev-state.template.json:3`, `plugins/dev-loop/skills/dev-loop/assets/dev-plan.template.md:6`
- **依存**: なし
- **コミットメッセージ**: `Step 1: Placeholder planDoc + mark state file as local in templates`（末尾に `[review: <verdict>, <rounds>r]`）

### Step 2 — SKILL.md をファイル管理方針/スキーマ/プロトコルで一貫更新
- **ゴール**: SKILL.md に「計画=`.claude/dev-plans/<date>-<slug>.md` で永続化・上書きしない」「状態=git 管理外・ローカル」を反映し、state スキーマ（`planDoc` コメント）・実行プロトコル（clean check / staging / commit 後 assert）・コミット規律・多人数運用の根拠を矛盾なく更新する。
- **背景**: SKILL.md は plan/state のデフォルトパス・「両ファイルを常にコミット」規律を documenting しており、放置すると新挙動と矛盾する。`6d067e1` が触れた記述を含む。
- **成功条件**:
  - [ ] 計画ファイルの場所・バージョニング方針（上書きしない・過去計画は保持）を記載
  - [ ] 状態ファイル=git-ignored/local を記載し、schema の `planDoc` コメントを新スキームに更新
  - [ ] プロトコル step 3（clean 例外）/ step 12（staging）/ step 13（assert）・決定表・コミット規律 tip が「plan+code のみコミット、state はローカル」で統一
  - [ ] 「state が未コミット＝コミット漏れ」等の旧記述が残っていない
- **検証方法**: `grep -rn "dev-state.json" plugins/dev-loop/skills/dev-loop/SKILL.md` で矛盾記述ゼロを確認、step.md との不変条件突合（読み合わせ）
- **想定変更ファイル**: `plugins/dev-loop/skills/dev-loop/SKILL.md:11`（他 L12/32/44-48/95/134/219-221 付近。着手時に現物で再確認）
- **依存**: 1
- **コミットメッセージ**: `Step 2: Update SKILL.md for versioned plans + local (git-ignored) state`（末尾に review 付記）

### Step 3 — plan-init に計画パス生成（上書き禁止・連番）を追加
- **ゴール**: `plan-init.md` に、計画を `.claude/dev-plans/<date>-<slug>.md` に新規作成する手順を記述する（date=YYYY-MM-DD、slug=ゴールを小文字 ASCII 化・非英数を `-` に畳む・空なら `plan`、`.claude/dev-plans/` を作成、同名衝突時は `-2`/`-3`… の連番、既存計画ファイルは**絶対に上書きしない**、生成パスを `planDoc` に設定）。
- **背景**: 現 plan-init は `.claude/dev-plan.md` を固定生成し上書きしていた。永続化のため新規作成・非上書きに変える。
- **成功条件**:
  - [ ] slug/date/衝突連番の規則が具体的に明記されている
  - [ ] 「既存の計画ファイルは上書きしない（新規作成のみ）」が明記されている
  - [ ] 冪等性チェック（現 step 1）が dev-state.json の設定に対してのみ言及し、計画は常に新規である旨に整合
  - [ ] `planDoc` に生成パスを設定する手順がある
- **検証方法**: サンプルゴールでのパス導出手順を dry-run 説明できること、scratch repo で同名 2 回実行→2 個目が `-2` になることを確認
- **想定変更ファイル**: `plugins/dev-loop/commands/plan-init.md:6`（intro のファイル一覧）, `:19`（冪等性）, `:36`（ステップ分解前にパス生成を挿入。着手時に再確認）
- **依存**: 1, 2
- **コミットメッセージ**: `Step 3: plan-init creates versioned, non-overwriting plan files`（末尾に review 付記）

### Step 4 — plan-init に .gitignore 自動追記＋既追跡の移行を追加
- **ゴール**: `plan-init.md` に、`.claude/dev-state.json` を対象リポジトリ root の `.gitignore` に冪等追記する手順（無ければ作成、完全一致で重複追記しない、末尾改行考慮）と、既に追跡済みなら `git rm --cached .claude/dev-state.json`（作業ファイルは保持）で index から外す移行手順を追加する。
- **背景**: state を git 管理外にするには `.gitignore` 追記が必要。既存 PJ で追跡済みの場合は `.gitignore` だけでは無効化されないため `git rm --cached` が要る。
- **成功条件**:
  - [ ] `.claude/dev-state.json` を完全一致チェックしてから追記（重複を作らない）
  - [ ] `.gitignore` が無ければ作成する
  - [ ] `git ls-files --error-unmatch` 等で既追跡を検知し、追跡時のみ `git rm --cached` を実行（作業ファイルは残す）
  - [ ] これらの変更（.gitignore 追記・staged deletion）は plan-init ではコミットせず、初回 `/step` が取り込む旨を記載
- **検証方法**: scratch repo で「初回=追記される／再実行=no-op／既追跡=cached から除去」を確認
- **想定変更ファイル**: `plugins/dev-loop/commands/plan-init.md`（作業ブランチ/ベースライン step 付近に挿入。着手時に再確認）
- **依存**: 3
- **コミットメッセージ**: `Step 4: plan-init auto-ignores dev-state.json (with git rm --cached migration)`（末尾に review 付記）

### Step 5 — step.md のコミット規律を state 非コミットに統一
- **ゴール**: `step.md` を「state をコミットしない」運用に統一する。step 3 の clean 判定の例外を「計画ファイル（＋初回の `.gitignore`/staged deletion）」のみにし（dev-state.json は ignored で不可視）、step 7/11 の注記を「state はローカルに書くがコミットされない」に、step 12 の `git add -A` 文言を「計画＋実装をステージ（state は ignored で除外＝意図どおり）」に、step 13 の clean 判定を「`git status --porcelain` は ignored を含まない前提」に更新する。`6d067e1` の記述と矛盾を残さない。
- **背景**: 現 step.md は「git add -A で dev-state.json＋plan＋impl を一括ステージ」「dev-state.json 未コミット＝コミット漏れ」と書いており、新方針と直接矛盾する。
- **成功条件**:
  - [ ] 「state はコミットされない（ローカル）」に記述が統一されている
  - [ ] step 13 の clean 判定が「ignored な state は porcelain に出ない＝コミット漏れ扱いしない」になっている
  - [ ] 計画ファイルは従来どおりコミットされる旨が保たれている
  - [ ] SKILL.md（Step 2）とコミット不変条件が一致している
- **検証方法**: step.md / SKILL.md / dev-plan.template.md の該当プロトコル節を並べて読み、コミット規律が同一であることを確認。`grep -rn "コミット漏れ\|dev-state.json" plugins/dev-loop/commands/step.md`
- **想定変更ファイル**: `plugins/dev-loop/commands/step.md:9`（既定 plan パス）, `:25`（clean 例外）, `:77`（staging）, `:81`（assert）付近（着手時に再確認）
- **依存**: 2
- **コミットメッセージ**: `Step 5: step.md commits plan+code only; state stays local (git-ignored)`（末尾に review 付記）

### Step 6 — README/plugin.json（version 0.3.0）を新挙動へ更新
- **ゴール**: root `README.md`・`plugins/dev-loop/README.md`・`plugin.json`（`version` 0.2.1→**0.3.0**、`description`）を新挙動（計画は `.claude/dev-plans/` に永続化・上書きしない／state は git 管理外・ローカル）へ更新する。必要なら `marketplace.json` の description も整合させる。
- **背景**: 利用者向けドキュメントと配布メタが旧挙動のままだと誤解を招く。ファイルレイアウトが変わる後方非互換のため minor バージョンを上げる。
- **成功条件**:
  - [ ] 両 README が新レイアウト＋state git 管理外（進捗はコミット履歴で辿る）を説明している
  - [ ] `plugin.json` の version が `0.3.0`、description が新挙動を反映
  - [ ] `plugin.json`（と触った場合 marketplace.json）が構文的に妥当
- **検証方法**: README 読み合わせ、`python3 -m json.tool < plugins/dev-loop/.claude-plugin/plugin.json`
- **想定変更ファイル**: `README.md`, `plugins/dev-loop/README.md`, `plugins/dev-loop/.claude-plugin/plugin.json:4`（version）`:5`（description）
- **依存**: 1, 2, 3, 4, 5
- **コミットメッセージ**: `Step 6: Document new plan/state file policy; bump to 0.3.0`（末尾に review 付記）

<!-- 以降ステップを追加する場合は同フォーマットで。out-of-scope の指摘は /step が followups として本節に追記する。 -->
