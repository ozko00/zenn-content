---
title: "Orca botの定期実行は作る前に読め。automations 14本を棚卸しした"
emoji: "🐋"
type: "tech"
topics: ["orca", "aiagent", "cli", "cron", "automation"]
published: true
---

AI ボットに定期実行を任せていると、「いま何が、いつ、どの条件で動くのか」を自分で言えなくなります。設定画面を開けば分かるはずですが、開いたついでに触ってしまうのが怖い。そこで、Orca（マルチエージェント IDE）の CLI を**読み取り専用のコマンドだけ**で動かし、手元の bot に載っている automation を棚卸ししました。

この記事を書いている作業自体も、Orca bot の automation から worker に委任されたものです。その run が履歴にどう見えたかも合わせて載せます。

**この記事で得られる情報**

- `orca automations list / show / runs / trigger-options` が実際に何を出すか（Orca CLI 1.4.269 の出力）
- 一覧の `next:` が UTC で出ること、`Custom schedule` の中身の見方
- 実行履歴の `completed` が「頼んだ作業が終わった」を意味しなかった実例
- bot が「時刻」以外で起動できるイベントの調べ方

**対象読者**

- Orca bot やそれに近い仕組みで、エージェントを定期実行している人
- 設定を変えずに現状だけ確かめたい人

確認日は 2026-11-22、環境は Linux・タイムゾーン Asia/Tokyo です。出力に含まれる ID は `<automation-id>` のように伏せています。作成・編集・実行のコマンド（`create` `edit` `run` `remove`）は一度も実行していません。

## 先に結論

- automation は 14本あり、`enabled` が 11本、`disabled` が 3本でした。
- 一覧の `next:` は **UTC** です。「火金 10:31」の次回は `2026-11-24T01:31:00.000Z` と出ます。9時間ずれて見えるだけで、設定は合っていました。
- 複数の曜日を指定したものは一覧では `Custom schedule` としか出ません。中身は `show` の `rrule:` 行にある 5フィールドの cron です。
- `disabled` の automation にも `next:` は表示されます。動くかどうかは `enabled` / `disabled` の表示で見ます。
- 履歴の `completed` は bot の run が終わったという意味でした。この記事を書く作業を起動した run は、私がまだ書いている最中に `completed` になっていました。

## 1. `list`：まず全体を数える

```bash
orca automations list
```

1件が3行で出ます（抜粋）。

```text
<automation-id>  AI速報（平日9:00）  claude  enabled
Weekdays at 9:00 AM  next: 2026-11-23T00:00:00.000Z
host: self

<automation-id>  Zenn投稿（火金10:31）  claude  enabled
Custom schedule  next: 2026-11-24T01:31:00.000Z
host: self

<automation-id>  テスト：note投稿（月木10:23）  claude  disabled
Custom schedule  next: 2026-11-23T01:23:00.000Z
host: self
```

14本の内訳は次のとおりでした。

| 一覧での表示 | 本数 | 例 |
|---|---|---|
| `Weekdays at …` / `Daily at …` / `Sundays at …` / `Wednesdays at …` | 6 | AI速報、不動産入門記事、週次の効果測定 |
| `Custom schedule` | 8 | Zenn投稿（火金）、note投稿（月木）、サイト記事（月水土） |

`Custom schedule` になっていた8本は、`show` で見るとすべて曜日を2つ以上指定した cron でした（`1,4`、`2,5`、`1,3,6`）。

### `next:` は UTC で出る

このマシンのタイムゾーンは Asia/Tokyo です。「平日 9:00」の次回が `2026-11-23T00:00:00.000Z`、つまり日本時間の 11月23日 9:00 です。日曜 8:00 のものは `2026-11-28T23:00:00.000Z` と、日付まで前日になります。

`orca automations create --help` には「Times are the host's local time unless --timezone names a zone」とあります。設定する時刻はホストのローカル時刻、表示される次回時刻は UTC、という組み合わせです。一覧を見て「9時間早く動く設定になっている」と思って直すと、そこで壊れます。

### `disabled` でも `next:` は出る

テスト用の3本は `disabled` ですが、`next:` には有効なものと同じ形式で次回時刻が入っていました。`next:` の行だけを見て「次は月曜に動く」と判断しないほうが安全です。1行目の末尾を見ます。

## 2. `show`：`Custom schedule` の中身を読む

```bash
orca automations show <automation-id>
```

```text
name: Zenn投稿（火金10:31）
provider: claude
agent: the bot's
enabled: true
runs in: new
schedule: Custom schedule
rrule: 31 10 * * 2,5
precheck: none
nextRunAt: 2026-11-24T01:31:00.000Z
workspaceMode: existing
reuseSession: false
host: self
prompt: Zenn記事を1本投稿する。…（以下略）
```

読みどころは4つです。

| 行 | この automation での値 | 意味 |
|---|---|---|
| `rrule:` | `31 10 * * 2,5` | 行の名前は rrule だが、入っていたのは 5フィールドの cron。火・金の 10:31 |
| `runs in:` | `new` | 毎回新しい会話で始まる（`create --help` の `--runs-in new` の説明による） |
| `precheck:` | `none` | 実行前に走らせる確認コマンドはなし |
| `reuseSession:` | `false` | 前回のセッションを使い回さない |

ほかの automation も同じ形でした。一覧で `Weekdays at 9:00 AM` と出るものは `rrule: 0 9 * * 1-5`、`Daily at 9:17 AM` は `17 9 * * *`、`Sundays at 8:00 AM` は `0 8 * * 0` です。一覧のラベルは読みやすくした表示で、実体は `rrule:` 行で確かめられます。

`prompt:` には bot への指示が全文で出ます。今回の bot では「実作業は自分でせず worker に任せる」「デザインを変える変更は本番に出さず承認を求める」「worker を起動したら待たずに終える」といった共通ルールが、各 automation の prompt に書かれていました。prompt には内部の ID が含まれることがあるので、出力をそのまま人に貼るのは避けます。

### `precheck` は使っていなかった

`create --help` によると、`--precheck <command>` を付けると定期実行の前にそのコマンドが走り、終了コード 0 なら続行、それ以外・エラー・タイムアウトなら実行を飛ばします。飛ばした回は履歴に `skipped_precheck` と出るそうです。手動実行（`orca automations run`）は precheck を通らない、とも書かれています。

手元の14本は、`show` で見るとすべて `precheck: none` でした（`runs in: new`・`reuseSession: false` も14本共通）。この機能の動作は私は試していません。ここに書いたのはヘルプの記載です。

## 3. `runs`：履歴の `completed` を信じすぎない

```bash
orca automations runs --limit 12
```

```text
<run-id>  <automation-id>  dispatched  manual  2026-11-22T04:06:25.264Z
週次の効果測定（日20:00） run 1

<run-id>  <automation-id>  completed  manual  2026-11-22T04:06:23.813Z
テスト：Zenn投稿（火金10:31） run 1
```

履歴は全部で 11件。すべて `manual`（手動実行）で、開始は 04:06:23〜04:06:25 UTC の約2秒間に並んでいました。状態を2回数えた結果です。

| 見た時刻（JST） | completed | dispatched |
|---|---|---|
| 13:07 | 7 | 4 |
| 13:08 | 8 | 3 |

ここで分かったことが2つあります。

**`completed` は「bot の run が終わった」で、「頼んだ作業が終わった」ではありませんでした。** 「テスト：Zenn投稿」の run は 13:07 の時点で `completed` でしたが、その run から委任されたこの記事は、そのときまだ1行も書かれていません。prompt に「worker を起動したら待たずに終える」とあるので、bot は worker を起動して自分の run を閉じます。委任型の automation では、履歴が緑でも成果物は別の場所で確かめる必要があります。今回なら、リポジトリのコミットと公開後の URL です。

**履歴がない automation もありました。** 14本のうち3本（note投稿・Zenn投稿・サイト記事の本番用）は `runs --id <automation-id> --json` の結果が `"runs": []` でした。

`--json` を付けると、1件ごとに `status` `trigger` `sessionKind` `precheckResult` `usage` `error` `scheduledFor`（エポックミリ秒）などが取れます。11件とも `error` は `null`、`sessionKind` は `chat` でした。

## 4. `trigger-options`：時刻以外で起動できるか

```bash
orca automations trigger-options --bot <bot名>
```

bot ごとに、どのイベントで automation を始められるかが出ます。手元の bot ではこうでした。

| ソース | 結果 | 出力に書かれていた理由 |
|---|---|---|
| github | 使える（`github.pr_opened` `github.ci_failed` など 20種） | bot に GitHub がリンクされている |
| webhook | 使える（`webhook.call`） | automation ごとに専用の URL が出る |
| slack | 使えない | Slack コネクタがない |
| linear | 使えない | Linear コネクタがない |
| gmail / calendar | 使えない | この bot では Google がオフ |

使えないソースには「なぜ使えないか」と「オーナーが何をすれば使えるか」のコマンドまで出ます。出力の最後には「一覧にないイベントは、スケジュールでポーリングして代用せず、オーナーに伝える」という趣旨の一文がありました。

`create --help` には、イベント起動（`--on`）は時刻指定（`--trigger`）の代わりで、両方を同時に付けると拒否される、スケジュールとイベントの両方が要るなら automation を2つ作る、と書かれています。手元の14本はすべて時刻起動で、イベント起動のものはまだありません。ここも私は作って試してはいません。

## 実際に使ってみた所感

- 読むだけなら4コマンドで足りました。`list` で数え、気になるものを `show`、動いたかどうかを `runs`、広げられる範囲を `trigger-options` で見ます。
- いちばん間違えそうだったのは時刻です。名前に「火金10:31」と書いてあるのに `next:` は `01:31` と出ます。名前に日本時間を入れておく運用は、この読み違いの保険になっていました。
- 推測で打ったコマンドが外れることもあります。進捗管理の「項目」を CLI から読もうとして `orca bot item list` を試したところ、`Unknown command: bot item` でした。1.4.269 の `orca bot --help` にこのサブコマンドはありません。あると思い込まず、まず `--help` を読むのが早道です。

## アンチパターン

1. **`next:` を日本時間だと思って時刻を直す。** 表示は UTC です。直す前に `show` の `rrule:` を見ます。
2. **`next:` があるから動くと思う。** `disabled` でも次回時刻は出ていました。
3. **`completed` を成果物の完了として扱う。** 委任型では bot の run が先に終わります。
4. **確認のつもりで `orca automations run` を打つ。** これは実行です。ヘルプによれば precheck も通りません。
5. **`show` の出力をそのまま共有する。** prompt に内部の ID が入っていることがあります。

## チェックリスト

- [ ] `list` で本数と `enabled` / `disabled` を数えた
- [ ] `Custom schedule` のものは `show` で `rrule:` を確かめた
- [ ] `next:` を UTC として読み、ローカル時刻に直して名前と照らした
- [ ] `runs` の `completed` とは別に、成果物（コミット・公開 URL）を確かめた
- [ ] 履歴が空の automation を把握した
- [ ] `trigger-options --bot` で、使えるイベントと使えない理由を確かめた

## この記事の限界

- 1台のマシン・1つの bot・Orca CLI 1.4.269 での観察です。バージョンが変われば表示も変わる可能性があります。
- 履歴 11件はすべて手動実行でした。時刻による自動起動の履歴（`trigger` の値や、実行を逃したときの扱い）は観察できていません。
- precheck、イベント起動（`--on`）、`--timezone` は、ヘルプの記載を引いただけで動作は試していません。

## 参考資料

一次情報は、手元で実行した次のコマンドの出力です（2026-11-22、Orca CLI 1.4.269）。

```bash
orca --version
orca bot --help
orca bot list
orca automations --help
orca automations list
orca automations show <automation-id>
orca automations runs --limit 12
orca automations runs --id <automation-id> --json
orca automations create --help        # ヘルプを読んだだけで、作成はしていない
orca automations trigger-options --bot <bot名>
orca skills get orca-cli --reference automations   # CLI 同梱のリファレンス
```

---

著者: Kosuke（[プロフィール](https://ai.sanvia.co.jp/authors/kosuke)）。AI とデータサイエンスを学び、不動産・教育の現場で AI の導入を支援しています。
