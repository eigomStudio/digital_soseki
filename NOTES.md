Digital Soseki 2 — 設計メモ

タブレット＋ペンで青空文庫テキストを TEI XML タグ付けするための単一ファイル Web アプリ。
本体は `index.html` 一つに HTML/CSS/JS をすべて内包。外部依存は xlsx 読み込み用の SheetJS（CDN）だけ。

## 守るべき不変点（壊すと性能・整合が崩れる）

### 1. セグメント描画（最重要）
- **文字を1個ずつ span 化しない**。これが過去版の重さの原因だった。
- 段落は `<p data-ps data-pe>`。段落内は「タグ境界で区切ったセグメント」だけを `<span>` にし、無装飾の地の文は素のテキストノードのまま。ノード数は文字数ではなくタグ数に比例する。
- `renderPara()` が重なりをフラット化してセグメント化する。最内の背景色＝最小の被覆 bg span。said→下線、quote→斜体、bg が2つ以上→入れ子ヒント（`--nest`）。ルビは `<ruby class=rb>base<rt>reading</rt></ruby>`。

### 2. 選択は caret ヒットテスト＋オーバーレイ矩形（ドラッグ中は再描画しない）
- `document.caretRangeFromPoint`（fallback `caretPositionFromPoint`）で DOM 点を取り、`nodeToGlobal()` で全体の文字オフセットに変換。逆方向は `globalToPoint()` / `globalRangeToDom()`。
- ドラッグ中は `#overlay` にハイライト矩形（`range.getClientRects()`）を描くだけ。**テキストは一切再描画しない**。
- ドラッグ終了で `pendingRange=[lo,hi]` を確定。タップ（移動なし）は `selectByOffset()` で既存 span を選択（編集用）。
- オフセット変換は全位置で往復一致することを検証済み。`<p>` の `data-ps`/`data-pe` と、`rt` を除外する TreeWalker が前提。ここを変えるなら往復テストを必ず通すこと。

### 3. 操作フローは「選択 → パレットで確定 → 反映」
- リアルタイム塗りはしない。パレットボタンは「現在の選択へ適用」する momentary アクション。
- 消しゴムは「選択範囲を部分消去」（split/shrink/delete、ルビは保持）。

### 4. タグ定義は5列モデル（色／タグ／ラベル／属性／属性値）
- 定義（`DEFS`）の各行 = `{color, tag(要素), label, attr, val}`。これが全タグ（構造タグ含む）の唯一の正。内蔵 `DEFAULT_DEFS` は初期値。
- `rebuildDefs()` が2つに振り分ける：
  - `SIMPLE{}` … `attr !== "ana"`。id は `defId()`＝`タグ_属性_値` を sanitize。各 1 ボタン、相互排他。
  - `ANAG{}` … `attr === "ana"` を要素ごとにグループ化（通常は `seg`）。**複数選択可**で `ana="#a #b"` に束ねる。span 側は `{tagId:"seg", anas:[...]}`。
- 表示チャンネルは要素から導出：`said`→下線、`quote`→斜体、それ以外→背景色。`channelOf()` 参照。
- ルビは定義に含めない組み込み（読み込み時に自動生成）。
- インスタンス個別の編集属性は要素ごとにハードコード：`EDITABLE = {persName:[ref], place:[ref], said:[who,toWhom]}`。これらはサイドパネルで入力（Excel には書かない）。
- Excel/CSV 読み込みは `applyDefRows()`。1行目ヘッダー語（色/color, タグ/tag, ラベル/label, 属性値→属性 の順で判定）で列を推定し、ダメなら位置順 `[color,tag,label,attr,val]`。**ヘッダー語がユーザのファイルと違うと読み違える**ので、その時はここを調整。

### 5. TEI 入出力の往復ルール
- 段落 = 改行区切りの1行。**行頭の全角空白（一字下げ）は本文に保持**し、そのまま `<p>` 内に出る。往復で一致する。
- head が段落全体を覆う場合は `<p>` でなく `<head>` ブロックとして出力（`isHead()` 判定）。
- `buildSegment()` が開閉タグを深さ順に挿入。`buildOC()` が要素・固定属性・編集属性・seg の `ana` を組み立てる。
- 読み込みは `parseTEI()`＝DOMParser で `<body>` を走査。`tagIdFromEl()` が要素＋属性から定義 id（または seg の anas）へ逆引き。未知要素は素通し（タグ化せず子を辿る）。`looksLikeTEI()` で本文/TEI を自動判別。

### 6. 交差ルール
- タグは部分交差を許さない（入れ子のみ）。`crosses()` で判定し、交差する適用は拒否して警告。

## 永続化
- `localStorage` キー `teiTagger.v3` に `{plain, spans, seq, fontSize, defs, jisMap}` をデバウンス保存。起動時 `restore()`。
- 本文は別ファイル読み込みか明示クリアまで保持。タグ定義・外字テーブルも一度読めば保持。
- 注意：`file://` 直開きだと Safari の構成次第で localStorage が効かないことがある。**GitHub Pages（https）で配信すれば解消**（CDN の xlsx もオンラインで安定）。

## 青空文庫の前処理（読み込み時）
- ヘッダ除去：55本ダッシュ等の罫線2本に挟まれたブロックを落とし、底本/入力/校正以降も落とす（`stripAozora()`）。
- 注記除去：`(?<!※)［＃…］` を削除（外字注記 `※［＃…］` は残す）。
- 外字変換：`※［＃…第N水準M-KK-TT］` を、ユーザ選択の JIS-Unicode テーブルで変換。キー＝`{N}-{HEX(KK+32)}{HEX(TT+32)}`（**水準N＋ku＋ten。面Mは使わない**。元の Python と一致）。未解決・第4水準などは `●`。
- 空行は詰めるが、行頭の一字下げは残す。

## いじるときの注意 / 既知の限界
- caret 選択は実レイアウトが要るので headless では単体テストできない。オフセット変換ロジック側でテストすること。
- 第4水準（面2）や、テーブル無し・未解決の外字は `●`。
- Excel の列順がモデルと違うと誤読み込みになる。`applyDefRows()` の検出語を確認。

## ざっくり検証の例
HTML を jsdom に `runScripts:"dangerously"` で読み込み（CDN の script タグは除去）、`window.loadPlain()` → `exportTEI()` → `parseTEI()` の往復一致と、`#palette .pen` 数、`#canvas p` 描画を確認するのが手早い。caret 選択以外のデータ経路はこれで回る。
