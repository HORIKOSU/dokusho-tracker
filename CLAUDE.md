# Booskill（読書管理ツール）｜Claude Code 引き継ぎ用 CLAUDE.md

最終更新：2026-10-07（月間目標機能まで反映済み）
公開URL：https://horikosu.github.io/dokusho-tracker/
リポジトリ：https://github.com/HORIKOSU/dokusho-tracker（Public／GitHub Pages・main・/root）

## 1. これは何か
- 単一ファイル `index.html`（HTML+CSS+JS、約2400行・約480KB）だけで動く読書管理ツール。ビルド工程・外部ファイルなし。
- データは各端末のブラウザ localStorage のみ（クラウド同期なし）。コードは公開されるが記録は他人に見えない。
- 作業対象は基本的に **`index.html` 1ファイルのみ**。ファイル名は必ず `index.html`。

## 2. ユーザーとの約束（最重要）
- 返答・メモ・コミットメッセージは**日本語**。
- 機能追加・デザイン変更は「実装 → ブラウザで動作確認 → スクリーンショットを見せる → ユーザーが明示的に『反映して』と言ってから本番ファイルへ反映」の順。**承認前に本番(`~/Documents/GitHub/dokusho-tracker/index.html`)を書き換えない**（作業用コピーで検証する）。
- 反対意見を出されても、根拠があればすぐに意見を変えない。最新情報を前提に答える。
- 見やすさ重視：適度に改行・余白を使って整理して回答する。
- 新規作成ファイル名の先頭は `yymmdd_〇〇`（例：`261007_引き継ぎメモ.md`）。ただし公開用の `index.html` は例外。
- スレッド名（会話タイトル）は日本語にする。

## 3. 反映・公開の流れ
1. `index.html` を編集・検証する。
2. 承認後、`~/Documents/GitHub/dokusho-tracker/index.html` に反映する。
3. ユーザーが GitHub Desktop で「Commit to main」→「Push origin」（1〜2分で公開サイトに反映）。
4. スマホで最新を見るときは URL 末尾に `?v=数字` を付けて強制読み込み。
- `.git/index.lock` が残ってコミット失敗する既知問題：`rm -f .git/index.lock` →（ターミナルで add/commit）→ push は GitHub Desktop の「Push origin」を使う。
- Cowork(Claudeデスクトップ)運用時は Projects「101.趣味開発」の `02.output/260805_読書管理ツール/index.html` にも同内容を保存していた。Claude Code に移行後は、この Projects 側コピーが古くならないよう、区切りのよいところで同期するか、移行した旨をユーザーに確認する。

## 4. データモデル（localStorage）
| キー | 内容 |
|---|---|
| `dokusho_books_v1` | `{v:1, books:[…]}`（読み込みは素の配列も許容） |
| `dokusho_sortmode_v1` | タブごとの並び替え設定 |
| `dokusho_goal_v1` | `{monthly:N}` 月間目標冊数（1〜99、未設定は0扱い） |
| `gb_api_key` | Google Books APIキー（任意） |
| `gemini_api_key` / `gemini_model` | AI分類用キー（任意。使用状況は未確認） |

book の主なフィールド：`id, isbn, title, author, publisher, pubdate, cover, status('want'|'stack'|'reading'|'done'), addedAt, updatedAt, source, memo, rating, priority(1〜5・want/stackのみ), history:[{status,at}]`
- `history` は最大200件。無い古いデータは `ensureHistory` が `updatedAt` から補完する。
- 「読み終えた日」＝ history の最後の `done` エントリ（`lastDoneEntry` / `doneMonthKey(b)` → "YYYY-MM"）。月間集計・リングはこれを基準にする。

## 5. 画面と主な機能
- 4つの棚：読みたい本(want)／積読本(stack)／読んでる本(reading)／読んだ本(done)。FABから本を追加（書名検索・ISBN入力・バーコード/ZXingカメラ読み取り）。
- 検索：openBD（ISBN）＋Google Books（キー任意。429対策でキー設定導線あり）。
- トップのヒーロー（`renderHeroStat`）：今月の読了数、先月比、中央リング、直近3か月のミニリング(`computeRecentMonths/buildMonthGauges`)。
  - 月間目標が未設定 → 中央リングは累計マイルストーン（`nextMilestone`、「あとN冊」）＋「🎯 月間目標を設定」ボタン。
  - 目標設定済み → 中央リングは「今月 N/目標 冊」の達成率リング、ボタンは「あとM冊」／達成時「🎉 達成！」。設定シートは `openGoalSheet()`（ー/＋ステッパー、プリセット[1,2,3,4,5,8,10]、保存、解除）。
- 優先度★(1〜5)：want/stack のカードと詳細で入力。カード上の変更は即保存するが**その場では並び替えず**、「並び替える」確認バー(`#resortBar`)を出す。並び順は★降順→新規登録が新しい順。
- 詳細シート（`openBookMenu`）：発売日・ISBN・履歴など。背景タップで閉じるのは click のみ。
- スキル機能：本をジャンルに分類して7スキルのレーダー／カバレッジ表示。`classifyBook` は `CATEGORY_RULES`（Google Books のカテゴリ）→ `SKILL_RULES` のキーワード得点の順で、毎回その場で再計算（キャッシュ非依存）。
- データ：JSONの書き出し／読み込み（`exportBooks/importBooks`、ISBN重複はスキップ）。

## 6. 開発・検証のしかた
- 共有シート：`openSheet(html)` / `closeSheet()`（`#overlay` / `#sheet` を共用）。
- デバッグ用に `window.__dokusho` に主要関数・state が公開されている。
- 検証は Playwright で、`localStorage` に `{v:1,books:[…]}` を投入して `reload` → 画面・DOM テキストを確認する方式が使いやすい（モバイル幅 400×820 程度）。
- サンドボックスでは Google Fonts（M PLUS Rounded 1c）が読み込めずフォントがフォールバックする。実サイトでは問題なし。外部通信エラー（フォント）はコンソールに出ても無視してよい。
- 検証用の一時ファイルは作業フォルダ外（/tmp 等）に置き、リポジトリに混ぜない。

## 7. 現在の状態
- 反映済み・公開待ち：月間目標機能（`dokusho_goal_v1`）。GitHub Desktop で Commit→Push が済んでいるかユーザーに確認すること。
- 直近の完了項目：タップで閉じる挙動の修正／カード星の控えめ表示／並び替え確認バー／詳細の★視認性／ジャンル分類の精度向上／★5段階化／発売日・ISBN表示／ヒーローのコンパクト化／今月読了数＋先月比／直近3か月リング／月間目標。
- 未着手の要望は現時点でなし（次はユーザーの指示を待つ）。
