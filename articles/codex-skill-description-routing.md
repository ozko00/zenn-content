---
title: "Codexスキルのdescriptionを広く書くと無関係な作業でも読まれるのか：OpenAIの助言を150回で確認"
emoji: "🌾"
type: "tech"
topics: ["codex", "openai", "llm", "agents", "prompt"]
published: true
---

OpenAI が 2026-09-11 の開発者ブログで書いた「スキルの description を広く書くと、関係のない作業でもスキルが読み込まれる」という指摘を、Codex CLI で150回実行して確かめました。gpt-6-astra と gpt-6.1-sol では、記事の「悪い例」の description にすると、マイグレーションと無関係な DB 作業 15回すべてでスキルが読み込まれました。「良い例」では 0回でした。一方 gpt-6-luna は、どちらの description でも無関係な作業では一度も読み込みませんでした。（確認日: 2026-10-27）

![田んぼの用水路の分水口。古い木の堰板が片側をふさぎ、水は開いた1本の水路にだけ流れ込み、もう一方はほぼ乾いている写真。スキルの description が、どの作業のときにスキルを使うかを振り分けることのメタファー](/images/codex-skill-description-routing/hero.jpg)

以下の数値は、すべて手元の生ログ（`codex exec --json` の出力）を集計スクリプトにかけた結果から転記しています。

## 一次ソース：OpenAI「Rethinking skills and prompts for GPT-6 Astra」

元記事は OpenAI Developers ブログの [Rethinking skills and prompts for GPT-6 Astra](https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra) です（ページ表示の日付 2026-09-11、取得日 2026-10-27）。モデルが賢くなった今、これまでに溜めたスキル・`AGENTS.md`・プロンプトを見直そう、という内容です。

スキルについての主張は次の3点です。

1. description は、いつ使うべきかが分かる範囲で、できるだけ短くする。原文は "skill descriptions should be as short as possible while making it clear when the model should use them"。
2. 複数のワークフローを持つスキルは、ルートの文書を最小限の振り分け役にして、詳細は別の文書やスクリプトに置く（progressive disclosure）。
3. 手順を細かく書きすぎたスキルは、いまのモデルではかえって結果を悪くすることがある。

1 には、Postgres マイグレーション用スキルの description の悪い例と良い例が図で載っています。

| | description（原文のまま） |
|---|---|
| 悪い例 | Create and validate Postgres schema migrations. Use when working with databases, queries, models, or persistence. |
| 良い例 | Create and validate Postgres schema migrations. Use when adding or changing a migration, or reviewing its rollout. |

図の説明は "the bad skill description can push the model to use it anytime it touches anything related to a database, rather than only when it has to handle a migration." です。悪い例では、マイグレーションを扱うときだけでなく、DB に関係する作業のたびにスキルが使われうる、という指摘です。

なお、公式ドキュメント [Build skills](https://learn.chatgpt.com/docs/build-skills)（取得日 2026-10-27）には、Codex は最初はスキルの name と description（と SKILL.md のパス）だけを読み、使うと決めたときに SKILL.md 本体を読む、と書かれています。この最初の一覧は既定でコンテキストウィンドウの 2% まで（不明な場合は 8,000 文字）で、スキルが多いと description が先に短縮されます。予算は `skills.max_context_tokens` で変えられ、上限は 10,000 トークンです（[Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)、取得日 2026-10-27）。

この記事で確かめたのは 1 の「広い description は無関係な作業でもスキルを読み込ませる」という部分だけです。2・3 と、description の短縮は試していません。

## 実験の設定

架空の小さなリポジトリにスキルを1つだけ置き、description だけを悪い例と良い例で入れ替えて、同じ作業を頼みました。

```text
$ codex --version
codex-cli 0.163.0
```

- 実行日時: 2026-10-27 10:43〜11:02（JST）。同じ Linux マシンで、4本並列で実行
- 認証: ChatGPT アカウントでのサインイン
- モデル: `gpt-6-astra`、`gpt-6.1-sol`、`gpt-6-luna`。`-m` で指定し、Codex の起動ヘッダに `model: gpt-6-astra` などと表示されることを確かめました（同じヘッダには `reasoning effort: none` と表示されていました。推論レベルは指定していません）
- 実行コマンド（1回ごと）: `codex exec --json --ephemeral --skip-git-repo-check --ignore-user-config -s workspace-write -m <モデル> -o <最終メッセージ> "<依頼文>"`
- 入力: この記事のために作った架空のリポジトリ `shop-api`（`app/db.py` にクエリ関数3つ、`migrations/` に SQL 3本、テスト1本）。実在のサービスとは関係ありません
- 1回ごとに新しい作業ディレクトリへリポジトリを複製して `git init` し、前の回の変更が残らないようにしました
- 回数: 3モデル × description 2種 × 作業5種 × 5回 ＝ 150回

スキルは `.agents/skills/pg-migrations/SKILL.md` に置きました（Codex がリポジトリ用のスキルを探す場所。[Build skills](https://learn.chatgpt.com/docs/build-skills)）。本文には、使われたかどうかが最終メッセージで分かるように、`MIGRATION-CHECKLIST: done` という行を出す指示を入れています。

```markdown
---
name: pg-migrations
description: （悪い例 または 良い例の文言をそのまま入れる）
---

# Postgres schema migrations

1. Add a new numbered file under `migrations/` (`NNNN_<summary>.sql`). Never edit a migration that may already have run.
2. For large tables, avoid locking: use `CREATE INDEX CONCURRENTLY` (in its own migration, outside a transaction) and add `NOT NULL` columns in steps (add nullable, backfill, then set NOT NULL).
3. Write a matching rollback note at the end of the file as a SQL comment: `-- rollback: ...`.
4. In your final message, include the line `MIGRATION-CHECKLIST: done`.
```

頼んだ作業は5種類です。3つはマイグレーションと無関係な DB 周りの作業、2つはマイグレーションの作業です。

| ID | 種類 | 依頼文 |
|---|---|---|
| N1 | 無関係 | app/db.py の get_active_users が削除済みユーザーも返してしまいます。deleted_at が NULL でないユーザーを除外するように、このクエリだけを直してください。 |
| N2 | 無関係 | app/db.py の count_orders_by_user に docstring と型ヒントを付けてください。 |
| N3 | 無関係 | app/db.py の SQL 文字列をモジュール先頭の定数に切り出して読みやすくしてください。動作は変えないでください。 |
| M1 | マイグレーション | users テーブルに last_login_at（timestamptz、NULL 可）を追加するマイグレーションを書いてください。 |
| M2 | マイグレーション | migrations/0003_add_orders_status.sql を本番に流す前にレビューして、ロールアウト上の問題点を指摘してください。ファイルは変更しないでください。 |

「スキルを読み込んだ」の判定は、`--json` のログでシェルコマンドの文字列に `pg-migrations/SKILL.md` が入っている実行が1回以上あるかどうかです。`rg --files` の結果にファイル名が出ただけの回（5回ありました）は、読み込みに数えていません。

## 結果：astra と sol は悪い例で毎回読み込み、luna は読み込まなかった

無関係な作業（N1〜N3）では、description によってはっきり差が出ました。マイグレーション作業（M1・M2）では、どちらの description でもほぼ毎回読み込まれました。

| モデル | description | 無関係な作業で読み込み | マイグレーション作業で読み込み |
|---|---|---|---|
| gpt-6-astra | 悪い例 | 15/15 | 10/10 |
| gpt-6-astra | 良い例 | 0/15 | 10/10 |
| gpt-6.1-sol | 悪い例 | 15/15 | 10/10 |
| gpt-6.1-sol | 良い例 | 0/15 | 10/10 |
| gpt-6-luna | 悪い例 | 0/15 | 9/10 |
| gpt-6-luna | 良い例 | 0/15 | 8/10 |

- 150回すべて終了コード 0 でした。
- 無関係な作業での悪い例と良い例の差（15/15 対 0/15）は、Fisher の正確検定（両側）で p ＝ 1.29×10⁻⁸ でした（gpt-6-astra、gpt-6.1-sol それぞれ）。gpt-6-luna は 0/15 対 0/15 で、差はありません。
- マイグレーション作業での読み込みは、gpt-6-luna で 9/10 と 8/10 でした。差があるとは言えません（p ＝ 1.0）。読み込まなかった3回（悪い例の M2 が1回、良い例の M1・M2 が各1回）は、スキルを見ずに既存の SQL を読んで作業していました。

スキルを読み込んだ無関係な作業では、最終メッセージにスキルが指示した `MIGRATION-CHECKLIST: done` の行が出ました。gpt-6-astra で 14/15 回、gpt-6.1-sol で 12/15 回です。docstring を付けるだけの作業でも、マイグレーション用の指示に従った跡が出力に残ったことになります。良い例の無関係な作業では、この行は一度も出ませんでした（0/45）。

無関係な作業で `migrations/` のファイルを変えた回は、どの条件でも 0 回でした。読み込んだうえで、今回は当てはまらないと判断していたことになります。

入力トークンと所要時間（無関係な作業、中央値）は次のとおりです。

| モデル | description | 入力トークン 中央値（最小〜最大） | 所要時間 中央値（最小〜最大） |
|---|---|---|---|
| gpt-6-astra | 悪い例 | 97,207（75,112〜140,912） | 37.6秒（22.8〜260.4） |
| gpt-6-astra | 良い例 | 92,164（74,213〜107,679） | 33.3秒（17.4〜231.3） |
| gpt-6.1-sol | 悪い例 | 93,478（76,049〜114,109） | 27.2秒（19.6〜40.5） |
| gpt-6.1-sol | 良い例 | 92,692（74,348〜122,012） | 25.7秒（19.1〜34.9） |

悪い例の中央値がやや大きいものの、範囲が大きく重なっています。今回のスキル本文は短い（4項目）ので、トークンと時間に差があるとは言えません。入力トークンは `turn.completed` の `usage.input_tokens` の値で、キャッシュされた分も含みます。

## 生出力の抜粋

`--json` ログの `agent_message` と `command_execution` を、改変せずに抜き出しました（作業ディレクトリのパスはログのまま）。

gpt-6.1-sol・悪い例・N1（クエリの修正）。依頼はクエリの修正だけですが、「関連する作業ルール」としてスキルを読みに行っています。

```text
agent_message | `get_active_users` のクエリを確認し、`deleted_at IS NULL` の条件だけを追加します。関連する作業ルールも確認します。
command_execution | /bin/bash -lc "pwd && rg --files -g 'AGENTS.md' -g 'db.py' -g 'SKILL.md' -g '*test*' -g 'pyproject.toml'"
command_execution | /bin/bash -lc 'cat .agents/skills/pg-migrations/SKILL.md'
agent_message | クエリ作業に適用される `pg-migrations` スキルを確認しました。今回はご指定どおり、マイグレーションは追加せずクエリだけを修正します。
（中略）
agent_message | [app/db.py](/tmp/shop-api-k972kdxv/app/db.py:6) の `get_active_users` に `WHERE deleted_at IS NULL` を追加しました。変更はこのクエリだけです。

差分チェックは通過しました。pytest が未インストールのため、テストは実行できませんでした。

MIGRATION-CHECKLIST: done
```

gpt-6-astra・悪い例・N3（SQL の定数化）。「DB 関連の変更なので」と、description の文言どおりの理由でスキルを読んでいます。

```text
agent_message | `app/db.py` とリポジトリの指示を確認し、動作を変えずに SQL 文字列を定数へ切り出します。DB 関連の変更なので、`pg-migrations` スキルも確認します。
command_execution | /bin/bash -lc "pwd && rg --files -g AGENTS.md -g 'app/db.py' -g '*test*' -g 'pyproject.toml' -g 'requirements*' -g 'SKILL.md' -g 'pytest.ini'"
command_execution | /bin/bash -lc 'cat .agents/skills/pg-migrations/SKILL.md'
（中略）
agent_message | [app/db.py](/tmp/shop-api-0qz8av92/app/db.py) の SQL 3件をモジュール先頭の定数に切り出しました。SQL と引数が変更前と一致することを確認済みです。

pytest 未導入のため、既存テストを直接実行して成功を確認しました。スキーマ変更はありません。

MIGRATION-CHECKLIST: done
```

gpt-6-astra・良い例・N1。スキルは読まずに作業を終えています。

```text
agent_message | `get_active_users` のクエリを確認し、`deleted_at IS NULL` の条件を追加します。
command_execution | /bin/bash -lc "pwd; rg --files -g AGENTS.md -g db.py -g '*test*' -g pyproject.toml -g pytest.ini"
（中略）
agent_message | [app/db.py](/tmp/shop-api-c7hmmtzx/app/db.py:6) の `get_active_users` のクエリだけに `WHERE deleted_at IS NULL` を追加しました。削除済みユーザーを除外します。

テストは `pytest` が未インストールのため実行できませんでした。
```

gpt-6-luna・悪い例・N1。悪い例の description でも、対象の関数だけを読んで終えています。

```text
agent_message | `get_active_users` だけを確認し、削除済みユーザーを除外する条件をそのクエリに追加します。
command_execution | /bin/bash -lc 'rg -n -C 5 "get_active_users" app/db.py'
command_execution | /bin/bash -lc "sed -n '1,80p' app/db.py"
agent_message | `get_active_users` のクエリに `WHERE deleted_at IS NULL` を追加しました。他のクエリは変更していません。
```

## 解釈と限界

OpenAI の指摘は、gpt-6-astra と gpt-6.1-sol ではそのまま再現しました。「Use when working with databases, queries, models, or persistence.」と書けば、モデルは書かれたとおり、クエリの修正や docstring の追加でもスキルを読みます。良い例のように対象の作業（マイグレーションの追加・変更・ロールアウトの確認）だけを挙げると、無関係な作業では読まれなくなりました。マイグレーション作業での読み込みは、範囲を狭めても減っていません。

gpt-6-luna は、悪い例でも無関係な作業ではスキルを読みませんでした。全体にコマンドの数が少なく（中央値2〜3）、対象のファイルだけを見て終える傾向がありました。ただしマイグレーション作業でも 20回中3回は読んでいないので、「luna は description の範囲を正しく見分けた」とまでは言えません。スキルを読みに行く頻度そのものが低い、とも読めます。

限界は次のとおりです。

- スキルは1つだけ、リポジトリは小さい架空のもの、作業は5種類です。スキルが多い環境や、description が互いに重なる環境（元記事が言う「矛盾する description」）は試していません。
- 今回のスキル本文は短く、読み込んでもトークンと時間に差は見えませんでした。本文や参照資料が長いスキルでは、無関係な読み込みのコストは大きくなると考えられますが、測っていません。
- 判定は「`SKILL.md` を読むコマンドを実行したか」です。読んだ内容がどこまで作業に影響したかは、`MIGRATION-CHECKLIST: done` の行が出たかどうかでしか見ていません。今回は無関係な作業で `migrations/` を変えた回はありませんでした。
- 推論レベルは指定していません（起動ヘッダの表示は `reasoning effort: none`）。推論レベルを上げた場合の結果は未確認です。
- 実行環境には Codex の system スキル（`~/.codex/skills/.system` にある imagegen・skill-creator など）が入っていました。これらがモデルに渡された一覧に載っていたかどうかは、`--json` のログからは確認できていません（未確認）。`--ignore-user-config` を付けて、ユーザー設定ファイル（config.toml）は読ませていません。
- 結果は 2026-10-27 時点の codex-cli 0.163.0 と各モデルでのものです。Codex CLI（ChatGPT アカウント）経由で動かした結果であり、ChatGPT の画面や API で同じ結果になるとは限りません。

実務では、「この領域に関係するなら使う」という書き方より、「この作業のときに使う」と作業の単位で書くほうが、少なくとも astra と sol では余計な読み込みが減ります。description を書き換えたら、今回のように無関係な依頼と対象の依頼を数回ずつ投げ、`--json` ログで SKILL.md が読まれたかを見ると、手早く確かめられます。公式ドキュメントにも「Test prompts against the skill description to confirm the right trigger behavior.」とあります（[Build skills](https://learn.chatgpt.com/docs/build-skills)）。

## 再現手順

上に載せた `SKILL.md` と依頼文があれば再現できます。架空リポジトリの中身は自由に作って構いません（`app/db.py` に SQL を使う関数がいくつかあり、`migrations/` に SQL ファイルがある程度で十分です）。

1回分のコマンドは次のとおりです。

```bash
# 毎回まっさらな作業ディレクトリを作る
work=$(mktemp -d)
cp -r shop-api/. "$work"/
git -C "$work" init -q && git -C "$work" add -A && \
  git -C "$work" -c user.name=t -c user.email=t@example.invalid commit -qm init

cd "$work"
codex exec --json --ephemeral --skip-git-repo-check --ignore-user-config \
  -s workspace-write -m gpt-6-astra -o last.txt \
  "app/db.py の count_orders_by_user に docstring と型ヒントを付けてください。" \
  < /dev/null > run.jsonl
```

スキルを読んだかどうかは、`--json` の出力から次のように判定しました。

```python
import json

def skill_read(jsonl_path, mark="pg-migrations/SKILL.md"):
    for line in open(jsonl_path, encoding="utf-8"):
        e = json.loads(line)
        item = e.get("item", {})
        if (e.get("type") == "item.completed"
                and item.get("type") == "command_execution"
                and mark in item.get("command", "")):
            return True
    return False

print(skill_read("run.jsonl"))
```

150回の一括実行では、これをモデル・description・作業・反復の組み合わせごとに回し、`turn.completed` の `usage` から入力・出力トークンを取りました。Fisher の正確検定は2×2の表に対して `math.comb` で計算しています。

## よくある質問

### Q. description を短くすると、本来使ってほしい作業でも使われなくなりませんか

今回の範囲では減りませんでした。マイグレーション作業での読み込みは、gpt-6-astra と gpt-6.1-sol で悪い例・良い例ともに 10/10、gpt-6-luna で 9/10 と 8/10 です。良い例は「adding or changing a migration, or reviewing its rollout」と、対象の作業を具体的に挙げています。短くするときも、使ってほしい作業の言葉は残すのがよさそうです。公式ドキュメントにも、description が短縮されても照合できるよう、主な用途と手がかりになる語を前に置くよう書かれています。

### Q. 無関係な作業でスキルを読まれると、何が困りますか

今回目に見えたのは、無関係な作業の最終メッセージに、スキルが指示した行（`MIGRATION-CHECKLIST: done`）が混ざったことです（gpt-6-astra 14/15、gpt-6.1-sol 12/15）。元記事は、スキルを読むとコンテキストを使い、その作業に当てはまらない指示が入る点を挙げています。今回のスキルは短かったので、トークンと時間の差ははっきりしませんでした。

### Q. Claude Code のスキルでも同じことが起きますか

この記事では確かめていません。今回は Codex CLI だけで計測し、Claude Code では同じ実験をしていません。

## 参考資料

- Rethinking skills and prompts for GPT-6 Astra — OpenAI Developers ブログ — https://developers.openai.com/blog/rethinking-skills-and-prompts-for-gpt-6-astra — 公表日: 2026-09-11（ページ表示）— 取得日: 2026-10-27
- Build skills（スキルの置き場所、初期一覧の予算、description の書き方）— OpenAI — https://learn.chatgpt.com/docs/build-skills — 公表日: 不明 — 取得日: 2026-10-27
- Configuration Reference（`skills.max_context_tokens`）— OpenAI — https://learn.chatgpt.com/docs/config-file/config-reference — 公表日: 不明 — 取得日: 2026-10-27
- Non-interactive mode（`codex exec`、`--json`）— OpenAI — https://learn.chatgpt.com/docs/non-interactive-mode — 公表日: 不明 — 取得日: 2026-10-27

---

筆者: Kosuke。AIみちしるべ（[https://ai.sanvia.co.jp](https://ai.sanvia.co.jp)）を運営しています。Codex CLI のインストールや `codex exec` の基本は[Codex CLIの使い方の実機検証記事](https://ai.sanvia.co.jp/articles/codex-cli-usage)にまとめています（[著者ページ](https://ai.sanvia.co.jp/authors/kosuke)）。この記事の架空リポジトリ・依頼文・生ログ・集計スクリプトの出力は手元に保存してあります。
