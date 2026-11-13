# WordPress Plugin Translation — Agent Instructions

translate.wordpress.org (GlotPress) からエクスポートした `.po` を日本語翻訳し、提案・インポートするための作業リポジトリ。

## 表記ルールの正典

`wordpress-ja-translation-guide` Skill を**常に正とする**（`SKILL.md` / `references/notation-rules.md` / `references/word-choice-rules.md`）。

**このファイルに要約を置かない。** 以前置いていた要約表が原典と食い違い、「全角『？』は使わない」を「`?` 自体を使わない」と読み違えて疑問文の訳文 24 件を `。` で終わらせた。迷ったら要約ではなく原典を読む。

機械チェックは Skill 同梱の `validate_po.py`（大量出力時は `--errors-only`）。分担パイプラインの 3 本
（`po_chunk.py` / `po_collect.py` / `po_apply_loop.py`）も Skill の `scripts/` にある（このリポジトリには無い）。

**validate が通っても「規約準拠」ではない。** 表記ルールの一部しか見ていない。たとえばプレースホルダー前後のスペースは `%d` 系の**直後**しか検査せず（`_PH_SPACE_RE` の対象は `[difu]`）、`%s` は素通りする。ここは `notation-rules.md` 6-1 を読んで自分で判断する領域（下訳の回収では、translators から数値と分かる `%s` に限って `po_collect.py` の `PH_NUM_SPACE` が拾う）。

## 訳語を決める手順

訳文を書いてから確認するのでは遅い。**書く前に**この順で引く。

0. **そのプラグインのディレクトリに `*-glossary.csv` があれば、まずそれを読む**（`ls "{Plugin Name}"/*.csv`）。GlotPress の Project Glossary をエクスポートしたもので、`en,ja,pos,description` の4列。**そのプラグイン専用の正典であり、1 と 2 のどちらよりも強い**
1. `references/glossary.md` を**原語で**検索する（`override` `dimensions` のように英単語で引く。日本語から引かない）
2. リポジトリ内の既存訳を grep する（`grep -rn --include="*.po" -E "原語|訳語候補" .`）。**ここで得られるのは訳語の根拠ではなく、次の 2 つだけ**
   - **製品固有の UI ラベル・機能名・対になる文字列。** changelog や説明文が `"Reset Plugin"` のように引用するラベルは、実 UI の訳と一致していないとユーザーが管理画面でそれを探せない。用語集は製品固有語を収録しないので、ここは既存訳が正典
   - **文体・語尾の前例。** 他プラグインの readme changelog における体言止めや句点の扱いなど
3. 見出しか本文か、ボタンかツールチップか迷ったら、**プラグインの実ソースで文脈を確認する**（次節）

**既存訳は承認済みでも間違っていることがある。一般語の訳語は 0 と 1 が勝つ。** 既存訳が多数派でも用語集と食い違ったら用語集に従う。Site Kit の `query` は既存 `.po` が「クエリ」2 件・「クエリー」1 件だったが、用語集の指定どおり「クエリー」を採った。

これを飛ばして `override` を「優先されます」と訳した（用語集の指定は「上書き」）。`%1$s of %2$s` は Code Snippets に既存訳があったのに、確認前に別の語順で確定させた。**日本語として自然かどうかで決めない。0 → 1 → 2 の順に強く、既存訳が用語集を上書きすることはない。**

0 を飛ばすと 2 の結果を信じて逆方向に倒れる。Site Kit の `Analytics` は Project Glossary が「アナリティクス」と指定しているが、その存在に気付かないまま UI `.po` の既存訳だけで判断し、人間の差し戻しと再修正で一往復を無駄にした。**`.po` を開く前に `ls *.csv`。**

## 原文の文脈はプラグインの実ソースで確認する

`msgid` だけでは、その文字列が見出しなのか本文なのか、ボタンラベルなのかツールチップなのか分からない。語尾（体言止め / 「〜ましょう」/ 敬体）はそこで決まるので、**迷ったら実物を読む。**

`.po` の `#:` 行がファイルパスを示している。取得は **SVN の raw** を使う。

```
https://plugins.svn.wordpress.org/{slug}/tags/{version}/{path}
```

`plugins.trac.wordpress.org` の `?format=txt` は Cloudflare に 403 で弾かれて curl では取れない。minified バンドルでも `grep -o -P '.{400}"原文".{400}' file.js` で前後を切り出せば、`title:` / `description:` / `label:` / `pdf.label` などの役割が読める。

2026-08-30 の Site Kit で、`Turn casual visitors into loyal readers` を見出しと判断して体言止め「〜に変える」で確定させた。ソースを読むと同一コンポーネントの三項演算子のもう一方の分岐が `Grow your revenue and deepen reader engagement`（既存訳「〜深めましょう」）で、**同じバナーの表裏だった**。ソースを見なければ語尾が割れたまま Import していた。

## ディレクトリ構成

```
{Plugin Name}/
  wp-plugins-{slug}-stable-ja.po         # UI 文字列
  wp-plugins-{slug}-stable-readme-ja.po  # readme 文字列
```

末尾の `-fuzzy` / `-translated` / `-changesrequested` は GlotPress エクスポート時のステータスフィルター違い（例: Query Monitor、Rank Math）。

### `-fuzzy` は訳文だけ読んではいけない

fuzzy は「原文が変わったのに訳が追いついていない」エントリーの集合。訳文単体では日本語として成立していて違和感がないので、**必ず msgid と msgstr を一語ずつ突き合わせる**。EWWW Image Optimizer の 5 件は全件が要修正で、壊れ方は次の 4 パターンだった。

- **旧原文の断片が残る** — 原文から消えた `Stage 2` が訳文の冒頭に残存
- **原文で追加された語が訳されていない** — `Image search` が訳文に無い
- **数値・バージョンが古い** — 原文 `PHP 8.1` に対し訳文は「7.2」。そのまま Import するとユーザーに誤情報を流す
- **プレースホルダーの型が変わっている** — 原文 `%1$s` に対し訳文 `%1$d`。`validate_po.py` が `PH_MISMATCH` の ERROR で検出する（型の変化はここでしか気付けない）

## GlotPress 由来のコメント

- `#, gp-priority: high` — プラグイン名・重要 UI。必ず翻訳
- `#, gp-priority: low` — changelog 項目など。翻訳を省略可
- `#. translators: ...` — 変数の意味を確認してから訳す。**`%s` に数値が入るかはここでしか分からない**。プレースホルダー前後のスペースは変換指定子ではなく置き換わる中身で決まる（原典: `notation-rules.md` 6-1）ので、読み飛ばすと必ず間違える
- `#. Found in ...` — readme 内の位置情報

## 編集後の確認

`PO-Revision-Date:` は手動更新不要（GlotPress が更新する）。`X-Generator:` は変更しない。文字コードは UTF-8 のまま。

## このワークスペース固有の例外

バージョン番号・規格番号は数字扱いせず、直後の日本語との間に半角スペースを残す（`PHP 8.1 以上`、`ISO8601 の日時`、`run_updates_v1 を使用`）。`validate_po.py` はこれを `NUM_SPACING_TOKEN` の WARN で報告するが**修正しない**（ERROR ではないので Import の障害にはならない。`--ignore NUM_SPACING_TOKEN` でまとめて抑制できる）。一般の数字はスペースなし（`5分`、`%d件`）。

### 文末の句点は原文の終端記号に合わせる

規則の原典は `word-choice-rules.md` 2-8 末尾の「句読点の有無は原文に合わせる」。ワークスペースではこれを `!` `?` `:` にも広げ、**終端記号は原文にあるものだけをそのまま写す**（原文に無ければ訳文にも付けない）。

Skill の「見出し・箇条書き・ボタンラベルは体言止め」を、**句点を落としてよい根拠として使わない**。体言止めにするかどうか（`〜を実行` か `〜を実行する` か）と、句点を打つかどうかは別の判断で、後者は原文が決める。チェックボックスのラベルのように体言止めに見える文字列でも、原文が `Run code within <?php ?> tags.` なら訳文は「<?php ?> タグ内でコードを実行する。」になる。

2026-08-25 の Code Snippets UI で、体言止めルールを優先して `.` 終わりの訳文から `。` を落とし、人間に差し戻された。`validate_po.py` はこれを検出しない。適用後に原文の末尾と訳文の末尾を突き合わせて確認すること。

## 未翻訳エントリーの翻訳

**未翻訳が 1 件でもあれば必ず `/translate-po path/to/ja.po`。** po-draft-translator サブエージェントが下訳を生成し、メインが直列に適用する。**メインは未翻訳エントリーを自分で訳さない。** 件数やチャンク数で例外を作らない（チャンクが 1 本しか立たなくても使う）。利点は並列ではなく、`po_collect.py` の機械チェック（終端記号・プレースホルダー・全角記号・HTML タグ）、`ref:` 見本の添付、`.work/` に残る証跡、メインのコンテキスト温存にある。

サブエージェントの定義は Skill の `assets/agents/po-draft-translator.md` を `~/.claude/agents/` にコピーしたものを使う（このリポジトリには置かない）。**Skill を更新したらコピーし直す**（`cp ~/.claude/skills/wordpress-ja-translation-guide/assets/agents/po-draft-translator.md ~/.claude/agents/`）。

- **fuzzy の修正はメインが行う。** `po_chunk.py` は fuzzy を対象にしない。同じセッションに未翻訳もあれば、その分は `/translate-po`
- **UI と readme の両方に未翻訳があれば**、ファイルごとに別の `.work/` を作り、Agent 呼び出しは 1 波にまとめる（2026-09-21 の Royal MCP は UI 43 件 + readme 12 件を 3 本同時に投げた）
- `po_chunk.py` の 30 件 / 3,000 字 は乗せた後の分割量で、使うかどうかの判断とは無関係

経緯（2026-09-12 に置いた「30 件以上」の線を 2026-09-23 に撤廃した理由）と実測は [.docs/parallel-translation-workflow.md](.docs/parallel-translation-workflow.md) の「適用範囲」にある。

原則・フェーズ・スクリプトの使い方は Skill の `references/parallel-translation-workflow.md` が持ち、このワークスペースでの上乗せは [.claude/commands/translate-po.md](.claude/commands/translate-po.md) に、実測と経緯は [.docs/parallel-translation-workflow.md](.docs/parallel-translation-workflow.md) にある。ここには置かない。**人間レビューと手動 Import は省略不可。**

中間成果物 `.work/{slug}/` は `.gitignore` **しない**。人間が下訳を直接レビューできる証跡なのでコミットする。

スクリプトの不具合を見つけたら、このリポジトリで直さず Skill 側（rocket-martue/wordpress-ja-translation-guide）にイシューを立てる。回帰テストも Skill 側の `tests/orchestration/` にある（新しい不具合を直すときは再現するケースを先に足す）。
