<div align="center">

<img src="assets/logo.svg" width="96" alt="朱批の印">

# Vermilion · 朱批skill

**AI が書いた長いレポートを、皇帝のように裁可する。**

AI が上奏文を書き、あなたが朱筆を執る。

[English](README.md) · [简体中文](README.zh-CN.md) · [繁體中文](README.zh-TW.md) · **日本語** · [Español](README.es.md)

[![Claude Code skill](https://img.shields.io/badge/Claude%20Code-skill-c8361d)](#インストール)
[![依存なし](https://img.shields.io/badge/依存-0-2e7d4f)](#得られるもの)
[![HTML ファイル 1 つ](https://img.shields.io/badge/フォーム-HTML%20ファイル%201%20つ-1f1a17)](#得られるもの)
[![MIT](https://img.shields.io/badge/license-MIT-6b625a)](LICENSE)

<a href="assets/vermilion-demo.mp4"><img src="assets/vermilion-demo.gif" width="860" alt="AI の長いレポートが 1 つのレビューフォームになり、レビュアーがカードごとに裁可し、2 回目には未決の項目だけが残る"></a>

⭐ **朱批skill が役に立ったら、Star をお願いします。** ほかの人が見つけやすくなります。

</div>

---

## なぜ必要か

いまの AI エージェントは長いレポートを書きます。監査、移行計画、設計レビュー、調査のまとめ。1 つのレポートに、**あなた**が決める・承認する・訂正すべき項目が 10〜50 個あるのは普通です。

レポートを読むのは問題ありません。**つらいのはフィードバックです。** スクロールし、引用し、チャットに「7 番目の項目について…」と打つ。どこまで答えたかを見失い、エージェントはどの項目のことか取り違えます。1 週間後には、何が決まったのか、どの版の文章に基づくのか、誰にも分かりません。

中国の皇帝は、臣下から届く長い上奏文（奏章）に長文で返事をしませんでした。朱墨で短い決定を書き込みました。これが「朱批」です。**朱批skill はあなたにこの役割を与えます。** エージェントは各項目を 1 枚のカードにし、根拠・選択肢・推奨案を添えます。あなたはフォームでカードごとに裁可します。エージェントは回答を正確に読み取り、コメントに返信し、次の回ではまだ決まっていない項目だけを尋ねます。

**結果：より良い決定と、より良いフィードバックを、少ない手間で。** すべての決定が見え、取りこぼしがありません。理由が必要な所には理由があります。エージェントが受け取るのは、解釈が必要な文章ではなく構造化された回答です。

## インストール

**Claude Code プラグインとして：**

```
/plugin marketplace add HaoranZhang122/vermilion-review
/plugin install vermilion@vermilion-review
```

**または通常の skill として：** `skills/vermilion/` を `~/.claude/skills/`（すべてのプロジェクト）または `.claude/skills/`（1 つのプロジェクト）にコピーします。

Python 3.8 以上が必要です。任意のセルフテストを実行するには、Chrome、Chromium または Edge も必要です。

## 使い方

```mermaid
flowchart LR
  A["長いレポート"] --> F["レビューフォーム<br/>（HTML ファイル 1 つ）"]
  B["レポートなし：<br/>エージェントが直接出力"] --> F
  F --> R["カードごとに裁可し、<br/>Seal #amp; send を押す"]
  R --> N["エージェントが返信し、<br/>未決の項目だけが戻る"]
  N -->|"次の回へ。未決がなくなるまで"| F
```

**1. 始める。** エージェントに長いレポートを渡します。または、レポートを省いて、結果を直接レビューフォームで出してもらいます：

```
Use vermilion to turn docs/migration-report.md into a review form. ultracode
```

```
Plan the database migration, and give me the result as a Vermilion review instead of a long report. ultracode
```

（日本語で頼んでも構いません。例：「vermilion で docs/migration-report.md をレビューフォームにして。ultracode」）

**2. 回答する。** エージェントから届いたフォームを開き、カードごとに裁可して **Seal & send** を押します。エージェントに「保存した」と伝えます。

**3. 繰り返す。** エージェントは回答を読み、未決の項目だけを載せた短いフォームを送ります。各項目には、あなたのコメントへの返信が付きます。未決がなくなると、最終的な決定記録を書きます。

> **Ultracode モードを強く推奨します。** 長いレポートでは、エージェントにマルチエージェントのワークフローを実行させてください。カードごとに 1 人の書き手、元の資料と照合する独立した検証者をカードごとに 1 人、最後に一貫性のチェック。長いレポートを要約する書き手は、但し書きを落とし、数字を変えがちです。検証者がそれを見つけます。詳しくは [references/ultracode.md](skills/vermilion/references/ultracode.md)（英語）。

**エージェントなしで今すぐ試す：** リポジトリをクローンし、[`skills/vermilion/examples/demo/forms/`](skills/vermilion/examples/demo/forms/) のファイルをブラウザで開きます（GitHub 上では HTML ファイルはソースとして表示されます）。架空のデータベース移行レビューで、空の 1 回目、レビュアーが返した 1 回目、2 回目が入っています。

## 得られるもの

**レビュアー（あなた）にとって**
- **1 つの決定に 1 枚のカード。** 各カードには 1 行の要約、根拠（迷ったときに開く）、図、各選択肢のコストとリスク、推奨案が 1 つあります。
- **素早く裁可。** <kbd>A</kbd> で推奨案を承認、<kbd>1</kbd>〜<kbd>9</kbd> で選択肢を選択、<kbd>J</kbd>/<kbd>K</kbd> でカード間を移動、<kbd>N</kbd> で次の未回答カードへ、<kbd>F</kbd> で要相談の印、<kbd>C</kbd> でコメント。決めたカードには朱印（准・改・驳・缓・问）が押されます。
- **確信度を伝える。** 確信あり・やや・不確か。不確かな項目は、次の回でエージェントが詳しく説明します。
- **理由が必要な所で理由を。** 反対・修正・「その他」を選ぶと、カーソルがコメント欄に移り、1 文の理由を求めます。
- **Seal & send。** ワンクリックで、未回答・不確か・理由のないカードを一覧し、回答をファイルに保存し、チャット用の短い要約をコピーします。保存したファイルがフォームのフォルダ、ダウンロード、デスクトップにあれば、エージェントが自分で見つけます。それ以外はパスを伝えてください。
- **2 回目以降は短い。** 確定した項目を 1 つの表にまとめ（どの行も再び開けます）、未決の項目だけをカードにします。前回のあなたの回答とエージェントの返信も表示されます。

**エージェントにとって**
- 依存なしの Python CLI（`vermilion.py`）：`init`、`check`、`build`、`receive`、`next-round`、`revise`、`rebuild`、`answers`、`status`、`selftest`。
- `check` はカード・リンク・図を検証し、平易な英語のスコア（約 80% ASD-STE100、任意）も出します。
- `receive` は、別のレビューのファイル、下書きプレビュー、一度も保存されていないファイル、発行したフォームとカードが違うファイルを受け付けません。
- `next-round` は各項目を「確定」か「未決」に分け、理由を付けます。`DECISIONS_after_R<n>.md` を書き、次の回のカードを下書きし、返信が必要な所に `TODO(AI)` を残します。決定のないまま消える項目はありません。

**記録として**
- **回答は HTML ファイルの中に保存されます。** Chrome と Edge では、最初の保存で保存ダイアログが開きます。フォームのファイルそのものを選んで置き換えを確認すると、以後はそのファイルに直接書き込みます。他のブラウザと VS Code のプレビューは、更新したコピーをダウンロードします。
- **発行・返却されたファイルはすべて凍結され**（読み取り専用のコピー）、sha256 とともに `REVIEW_LOG.md` に記録されます。各カードは内容の指紋を持ち、`receive` が発行したフォームと照合するので、各回答はあなたが受け取ったカードの組までたどれます。
- **版**は `R<回>.<改訂>`。選択肢の文字の意味は決して変わりません。改訂（R1.1）は文言だけを直し、回答を引き継ぎます。新しい回（R2.0）は未決の項目だけを尋ねます。
- **プライバシー重視の設計。** サーバーもネットワーク通信も CDN もありません。フォームはディスク上の 1 つのファイルです。

## コマンド

| コマンド | 内容 |
|---|---|
| `vermilion.py init --dir review --slug NAME --title TEXT` | レビュー用フォルダと設定ファイルを作ります。 |
| `vermilion.py check --dir review --version R1.0` | カードを検証し、平易な英語のスコアを出します。 |
| `vermilion.py build --dir review --version R1.0 [--draft]` | プレビュー（`--draft`）を作るか、フォームを発行します（作業用コピー、凍結コピー、ログ）。 |
| `vermilion.py receive --dir review --find` | レビュー用フォルダ、~/Downloads、~/Desktop にあるこのレビューの新しい保存をすべて、古い順に受け取ります（`--file SAVED.html` で指定も可）。それぞれを凍結し、回答（`.md` と `.json`）を取り出して記録します。 |
| `vermilion.py next-round --dir review [--dry-run] [--open ID ...]` | 回答を分類し、決定ファイルを書き、次の回を下書きします。 |
| `vermilion.py revise --dir review --from R1.0 --to R1.1` | 改訂用にカードの組をコピーします。 |
| `vermilion.py build --dir review --version R1.1 --carry SAVED.html` | その回の回答を引き継ぐ改訂版を発行します。 |
| `vermilion.py rebuild --dir review --version R1.0 --out FILE` | ログから発行済みフォームを再構築し、バイト単位で比較します。 |
| `vermilion.py answers SAVED.html` | 任意のフォームの回答を表示します。記録はしません。 |
| `vermilion.py status --dir review` | ログとレビューの進み具合を表示します。 |
| `vermilion.py selftest FORM.html` | フォーム内蔵のセルフテストをヘッドレスブラウザで実行します。 |

詳細は [SKILL.md](skills/vermilion/SKILL.md) と [references/](skills/vermilion/references/)（英語）にあります。カードの形式、カードの書き方、平易な英語のルール、回と版、Ultracode のやり方です。

## よくある質問

**どのブラウザ？** Chrome または Edge では、最初の保存で保存先を尋ねられます。フォームのファイルそのものを選んでください。Firefox、Safari、VS Code のプレビューでも使えます。保存すると更新したコピーがダウンロードされるので、そのコピーをエージェントに渡してください。

**保存せずにタブを閉じた。** 同じブラウザで同じファイルを開いてください。ブラウザに残っている回答をフォームが提示します。

**エージェントが言う決定を信用できる？** 決定ファイルには、すべての項目の状態と理由が載ります。各回答は、あなたが保存したファイルの凍結コピーとその sha256 までたどれます。コメントはあなたの言葉のままエージェントに渡り、エージェントはすべてのコメントを読み、次の回で未決カードごとに返信します。

**他のエージェント（Codex、Cursor、Gemini CLI）でも使える？** CLI はただの Python、SKILL.md はただの Markdown です。ファイルを読んでコマンドを実行できるエージェントなら従えます。

**データはどこかに送られる？** いいえ。フォームはネットワーク通信を一切しません。図はファイルに埋め込まれています。

## Star の推移

朱批skill で時間が節約できたら、⭐ がいちばん簡単なお礼になり、ほかの人が見つける助けにもなります。

<a href="https://star-history.com/#HaoranZhang122/vermilion-review&Date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=HaoranZhang122/vermilion-review&type=Date&theme=dark" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=HaoranZhang122/vermilion-review&type=Date" />
    <img alt="Star History Chart" src="https://api.star-history.com/svg?repos=HaoranZhang122/vermilion-review&type=Date" width="640" />
  </picture>
</a>

## 作者

Haoran Jack Zhang · [@HaoranZhang122](https://github.com/HaoranZhang122)

質問やアイデアは [issue](https://github.com/HaoranZhang122/vermilion-review/issues) でどうぞ。

## ライセンス

MIT。[LICENSE](LICENSE) を参照。
