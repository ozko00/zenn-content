---
title: "「見直して」は効いたか？日本語の制約付き文章で採点基準・見直し・few-shotを2モデルで実測"
emoji: "🧪"
type: "tech"
topics: ["claude", "claudecode", "agentsdk", "llm", "prompt"]
published: true
---

![表紙](/images/a6b5a060ff/cover.png)


Claude Opus 5.5 と Claude Haiku 5.5 に、制約の多い日本語の文章を書かせて自動で採点しました。プロンプトの工夫は4つの条件で比べています。今回の範囲（7 タスク × 各 3 回）では、採点基準の同梱・見直し1回・few-shot 1例のどれを足しても、全制約合格率は baseline と「差があるとは言えない」結果でした。不合格は全 168 試行のうち、合格 163 試行を除いた残りだけです。
一方、見直し1回（別の呼び出しで見直させる形）は、平均出力トークンが Opus 5.5 で 1.75 倍、Haiku 5.5 で 1.53 倍になり、平均所要時間も 1.77 倍と 1.79 倍に伸びました。採点基準と few-shot 1例では、コストはほとんど変わりませんでした。
違反が出たのは文末表現・段落数・全体の文字数で、「語の回数」や「数字の個数」では出ませんでした。プロンプトに何かを足す前に、まず自分のタスクを baseline で測り、どの種類の制約で落ちるのかを確かめるのが先だと考えます。

## はじめに：公式ガイドは「再チェックさせるな」と言うが、日本語の字数でも同じか

Claude の公式ドキュメントでは、「見直し」の扱いがモデルによって違います。

- [Prompting Claude Opus 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5) の Self-correction 節では、Opus 5 は指示がなくても自分の誤りを見つけて直すとしています。そのため "double-check your answer" のような再チェック指示は、コストが増えるだけで結果は良くならないので避けるよう勧めています。
- [Prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) は一般論として、"Before you finish, verify your answer against [test criteria]." のような一文で誤りを確実に捕まえられるとしています（特にコーディングと数学）。ただし Opus 5 は例外で、この指示は書き換えずに削除するよう書いています。
- [Prompting Claude Fable 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5) は "Separate, fresh-context verifier subagents tend to outperform self-critique." として、別コンテキストの検証役のほうが自己批評より成果が出やすいとしています。

これらの記述が例に挙げているのは、コーディングや数学、長時間のタスクです。「480字以上540字以内」「この語をちょうど3回」のように、数えないと守れない日本語の制約は直接は扱っていません。そこで、自動で採点できる小さな実験を組んで測りました。

## 前提：使ったモデル

[release notes](https://platform.claude.com/docs/en/release-notes/overview.md) によると、リリース順は Fable 5（2026-06-09）→ Opus 5（2026-07-24、入力 $5 / 出力 $25、100 万トークンあたり）→ Opus 5.5（2026-09-22、入力 $4 / 出力 $20）→ Haiku 5.5（2026-10-07）です。

今回の実験では Opus 5 ではなく **Opus 5.5** と **Haiku 5.5** を使いました。[Prompting Claude Opus 5.5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5) は、Opus 5 のパターンを「妥当な出発点」とするだけで、自己チェックについて Opus 5.5 個別の記述は見当たりませんでした。best practices にも、モデル名を挙げた手法は別のモデルに当てはめる前に自分の eval で確かめるよう書かれています。つまりこの記事は、「Opus 5 向けの勧めを Opus 5.5 と Haiku 5.5 で確かめた」ものです。

出力トークンを読むうえで大事な前提が2つあります。

- Opus 5.5 は既定の effort が medium で、thinking は常にオンです（無効にできません）。Haiku 5.5 も thinking パラメータを省略すると既定でオンになります（best practices）。今回は effort / thinking を指定せず、SDK の既定のまま動かしました（`run-config.json` の `sdk_options_note`）。
- [CHANGELOG](https://github.com/anthropics/claude-agent-sdk-typescript/blob/main/CHANGELOG.md) の 0.3.257 によると、thinking のトークン（`thinkingTokens`）は出力トークン（`outputTokens`）の内数です。つまり、出力トークンには thinking のぶんも入っています。今回の実測でも、出力トークンの大半は thinking でした（後述のコストの表）。

モデル ID は実行前に1回ずつ呼んで確かめました（`model-check.json`）。SDK が実行時に報告したモデル名は次のとおりです。

| key | 指定したモデル ID | 実行時に SDK が報告したモデル |
|---|---|---|
| opus | claude-opus-5-5 | claude-opus-5-5 |
| haiku | claude-haiku-5-5 | claude-haiku-5-5 |

## 一度目の実験は天井に当たった

最初の版（`experiment-v1/`）は、短い文章のタスクに少しの制約を付けたものでした。結果は全条件・全試行が全制約合格の天井効果で、条件の違いが見えませんでした。そこで、タスクを長くし、制約を増やし、制約どうしがぶつかる組み合わせにして作り直しました（v2）。たとえば次のような組み合わせです。

- 「ペアプログラミング」をちょうど3回使う × 最初の段落には入れない
- 見出しを「## 1. 〇〇」の形にする × 半角数字は見出しの番号も含めて5〜6個まで
- 「v2」をちょうど4回使う × 半角数字は「v2」の「2」も含めて5〜7個

難しくしたのは、条件の差が見えるようにするためです。v2 の設計は実行前に `design.json` に固定しました（`"fixed": true`）。結果を見てからタスクを選び直してはいません。

## 方法

### タスク（7 個）

題材はどれも「たとえばの設定」で、400〜800字程度の業務文です。t1 社内ブログ（ペアプロ試行の報告）、t2 経費精算アプリの案内（だ・である調）、t3 API v2 のお知らせ、t4 勉強会レポート、t5 メンテナンスのお知らせ、t6 採用ページの職種紹介（だ・である調）、t7 障害報告（ポストモーテム）の7つです。制約は1タスクあたり 11〜12 個、合計 82 個あります。t1 の指示文（`design.json` の `tasks[0].input`）を、数え方の部分を除いて載せます。

```text:experiment/design.json（tasks[0].input より抜粋）
社内ブログに載せる記事を書いてください。題材（たとえばの設定）: ある開発チームが、二人一組で開発する「ペアプログラミング」を三か月試した。その結果を社内に報告する。

条件:
- 文字数は480字以上540字以内
- 「## 」で始まる見出しをちょうど2つ置く
- 見出しの文字（「## 」を除く）は4〜12字で、数字を含めない
- 地の文の段落はちょうど3つ
- 「- 」で始まる箇条書きをちょうど3項目
- 箇条書きの各項目は「- 」を除いて15〜25字
- 「ペア」「レビュー」「新人」「障害」「合意」をすべて含める
- 「とても」「非常に」「重要」「効率」を使わない（「効率的」なども不可）
- 「ペアプログラミング」はちょうど3回使う
- 地の文の文末は「です。」「ます。」「ました。」「ません。」「でした。」「ください。」のどれかにする
- 半角の数字を2〜3個使い、どれも2以上30以下にする
- 最初の段落には「ペアプログラミング」を入れない
```

各タスクには、全制約を満たす参照解（`reference`）を用意しました。`validate=1` で採点器にかけ、合格することを確かめています。これは制約どうしが矛盾していないことの確認です。few-shot の例も同じように確かめました。

### 条件（4つ）

- **baseline**：タスク（条件付き）だけを渡す
- **rubric**：タスクに加えて、自動採点の基準（制約ごとの判定ルール）を番号付きで渡す
- **self_critique**：1回目の回答のあと、**別の呼び出し**でタスクと回答を渡し、1回だけ見直させる
- **few_shot**：別の題材（ハッカソン報告）のタスクと、その全制約を満たした回答を1つ見せる

![4条件の流れ。baseline・rubric・few_shot は入力だけが違う1回の呼び出し。self_critique は下書きとタスクを新しい2回目の呼び出しに渡して1回だけ見直す](/images/a6b5a060ff/b3-conditions.png)

self_critique は「同じ会話での見直し」ではありません。`persistSession: false` で、2回目は新しい呼び出しです。形としては best practices の「Chain complex prompts」（下書き → 基準に照らしてレビュー → 修正を、それぞれ別の API 呼び出しにする）に近いものです。ただし、同じモデルが同じシステムプロンプトで見直しているので、Fable 5 ガイドの言う「別コンテキストの検証役サブエージェント」とも違います。few_shot の例は1つだけで、best practices が勧める 3〜5 個より少ない点にも注意してください。

見直しのプロンプトは次のとおりです。

```js:experiment/run.mjs
export const DEFAULT_CRITIQUE = `以下はあるタスクへのあなたの回答です。タスクの要件に照らして1回だけ見直し、問題があれば直した最終回答だけを出力してください。見直しの説明や前置きは書かないでください。

# タスク
{task}

# 回答
{answer}`;
```

### モデル・回数・実行条件

2モデル × 4条件 × 7 タスク × 3 回で、合計 168 試行です（1セルあたり 21 試行）。エラーは 0 件でした。呼び出しには [Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) の `query()` を使いました（旧 URL の platform.claude.com/docs/en/agent-sdk/overview からは転送されます）。オプションは次のコードのとおりです。

```js:experiment/run.mjs
    for await (const m of query({ prompt, options: {
      model, systemPrompt: system, tools: [], allowedTools: [], settingSources: [], maxTurns: 1, persistSession: false,
      permissionMode: 'default', canUseTool: async () => ({ behavior: 'deny', message: '実験ではツールは使えません' }),
      cwd: '/tmp', env: agentEnv(),
    } })) {
```

- ツールなし（`tools: []`）、`maxTurns: 1`、`settingSources: []`（ユーザー・プロジェクト・ローカルの設定を読み込まない）
- システムプロンプトは固定（`system-prompt.txt`：`You are a careful assistant. Follow the user's instructions exactly. Output only what is asked.`）
- 実行順はシード `20261016` でシャッフルし、時間帯や混雑の影響が特定の条件に偏らないようにしました。並列数は4です。

[TypeScript リファレンス](https://code.claude.com/docs/en/agent-sdk/typescript)には、プロンプトを書いたとおりに届ける `verbatimPrompts` オプションもあります（SDK 0.3.280 以降、Claude Code 2.1.248 以降）。今回は使っていません。CHANGELOG（0.3.280）によると、このオプションでは `@path` の展開やスラッシュコマンドの振り分けが行われません。プロンプトにそうした文字列が入りうる場合は、候補になります（CLAUDE.md などの設定の読み込みは、今回のように `settingSources` で制御します）。

## 採点基準

LLM による採点は使わず、すべてルールで判定しました（`run.mjs` の `runCheck`）。数え方は、どのタスクの指示文にも同じものを入れています。

```text:experiment/design.json（counting_rules）
数え方:
- 文字数は、改行を除くすべての文字を数えます（空白・記号・見出しの「## 」も1字ずつ数えます）。
- 見出し行は「## 」で始まる行、箇条書き行は「- 」で始まる行です。段落は、見出し行・箇条書き行・空行で区切られた、地の文のまとまりです。
- 数字は、半角数字の連なりを1個と数えます（例:「12件」は1個、「v2」の「2」も1個）。全角数字は使えません。
- 語の回数は、見出し・箇条書きを含む本文全体での出現回数です。
```

制約は9つのタイプに分けて集計しました。例はどれも t1 のものです。

| タイプ | 判定する制約の数 | たとえば |
|---|---|---|
| 文字数 | 13 | 全体が480〜540字、箇条書きの各項目が15〜25字 |
| 構成の数 | 21 | 段落ちょうど3つ、箇条書きちょうど3項目 |
| 見出し形式 | 5 | 「## 」＋数字を含まない4〜12字 |
| 必須語 | 7 | 「ペア」「レビュー」などをすべて含む |
| 禁止語 | 7 | 「とても」「重要」などを含まない |
| 語の回数 | 7 | 「ペアプログラミング」がちょうど3回 |
| 文末表現 | 7 | 地の文の各文が「です」「ます」などで終わる |
| 数字 | 7 | 半角数字が2〜3個で、どれも2以上30以下 |
| 語の位置 | 8 | 最初の段落に「ペアプログラミング」を入れない |

指標は次のとおりです。

- 全制約合格率：1試行のすべての制約を満たした割合。Wilson 法の信頼区間を付けました（`run.mjs` の `wilson(k, n, z = 1.96)`）。
- 制約ごとの遵守率の平均
- 制約タイプ別の違反率：違反した数 ÷ そのタイプを判定した数
- 出力トークン（`usage.output_tokens`）と所要時間（`duration_ms`）。self_critique はどちらも2回の呼び出しの合計です。

文字数・語の回数・文末の判定は、次のコードで行っています。なお、次の「結果」の節では Opus 5.5 を Opus、Haiku 5.5 を Haiku と略します。

```js:experiment/run.mjs
    case 'chars_range': { const n = chars(a); return n >= c.min && n <= c.max; }
    case 'paragraph_count': return P.paragraphs.length === Number(c.value);
    case 'bullet_count': return P.bullets.length === Number(c.value);
    case 'bullet_chars_range': return P.bullets.length > 0 && P.bullets.every(b => { const n = chars(b); return n >= c.min && n <= c.max; });
```

```js:experiment/run.mjs
    case 'count_exact': return countOcc(a, c.value) === Number(c.count);
    case 'sentence_end': return P.paragraphs.length > 0 && P.paragraphs.every(p => p.endsWith('。') && p.split('。').filter(s => s.trim()).every(s => c.endings.some(e => s.endsWith(e))));
```

<!-- RESULTS:BEGIN -->
## 結果

### 全制約合格率

| モデル | 条件 | 試行数 | 全制約合格 | 全制約合格率(%) | 95%区間(%) | 制約ごとの遵守率の平均(%) | 平均出力トークン | 平均所要時間(秒) |
|---|---|---|---|---|---|---|---|---|
| opus | baseline | 21 | 19 | 90.5 | 71.1〜97.3 | 99.2 | 3299 | 31.6 |
| opus | rubric | 21 | 21 | 100.0 | 84.5〜100.0 | 100.0 | 3062 | 29.3 |
| opus | self_critique | 21 | 21 | 100.0 | 84.5〜100.0 | 100.0 | 5775 | 56.1 |
| opus | few_shot | 21 | 21 | 100.0 | 84.5〜100.0 | 100.0 | 3325 | 35.0 |
| haiku | baseline | 21 | 20 | 95.2 | 77.3〜99.2 | 99.6 | 11236 | 56.9 |
| haiku | rubric | 21 | 20 | 95.2 | 77.3〜99.2 | 99.6 | 11835 | 60.2 |
| haiku | self_critique | 21 | 20 | 95.2 | 77.3〜99.2 | 99.6 | 17152 | 101.6 |
| haiku | few_shot | 21 | 21 | 100.0 | 84.5〜100.0 | 100.0 | 11237 | 51.2 |

![モデル×条件ごとの全制約合格率と、制約ごとの遵守率の平均](/images/a6b5a060ff/b3-results.png)

Opus で不合格が出たのは baseline だけでした（19/21、90.5%）。rubric・self_critique・few_shot はどれも 21/21 です。Haiku では baseline・rubric・self_critique がそれぞれ 20/21、few_shot が 21/21 でした。

baseline との差は、Opus ではどの条件も +9.5 ポイントです。Haiku では rubric と self_critique が ±0.0 ポイント、few_shot が +4.8 ポイントでした。ただし、この差は数試行ぶんにすぎません。Opus の baseline の区間は 71.1〜97.3%、ほかの三条件は 84.5〜100.0% で、大きく重なっています。Haiku も同じです（77.3〜99.2% と 84.5〜100.0%）。このため、**条件のあいだで差があるとは言えません**。両モデルを合わせても、baseline は 92.9%（81.0〜97.5%）、few_shot は 100.0%（91.6〜100.0%）で、区間は重なっています。

タスク別の合格数は次のとおりです。不合格は t4 と t6 に集まっています。

| タスク | opus/baseline | opus/rubric | opus/self_critique | opus/few_shot | haiku/baseline | haiku/rubric | haiku/self_critique | haiku/few_shot |
|---|---|---|---|---|---|---|---|---|
| t1 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| t2 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| t3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| t4 | 3/3 | 3/3 | 3/3 | 3/3 | 2/3 | 3/3 | 2/3 | 3/3 |
| t5 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |
| t6 | 1/3 | 3/3 | 3/3 | 3/3 | 3/3 | 2/3 | 3/3 | 3/3 |
| t7 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 | 3/3 |

### どの制約で落ちたか

| モデル | 条件 | 文字数 | 構成の数 | 見出し形式 | 必須語 | 禁止語 | 語の回数 | 文末表現 | 数字 | 語の位置 |
|---|---|---|---|---|---|---|---|---|---|---|
| opus | baseline | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 9.5 | 0.0 | 0.0 |
| opus | rubric | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| opus | self_critique | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| opus | few_shot | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| haiku | baseline | 0.0 | 1.6 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| haiku | rubric | 2.6 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |
| haiku | self_critique | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 4.8 | 0.0 | 0.0 |
| haiku | few_shot | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 | 0.0 |

![制約タイプ別の違反率（モデル×条件）](/images/a6b5a060ff/b3-violations.png)

全試行を合わせると、必須語 0.0%、禁止語 0.0%、語の回数 0.0%、数字 0.0%、語の位置 0.0%、見出し形式 0.0% で、どれも違反はありませんでした。違反が出たのは、文末表現（3/168）、構成の数（1/504）、文字数（1/312）だけです。中身を raw ログ（`raw/*.jsonl` の `detail`）で確かめました。

- **Opus / baseline**：違反 2 件はどちらも t6（採用ページ・だ・である調）の文末表現でした。許されている文末は「である。」「する。」「いる。」「ない。」ですが、「〜仕組みとして残す。」「〜改善まで手がける。」のように、ふつうの動詞の終止形で終わる文が混ざっていました。
- **Haiku / baseline**：t4 の段落数が指定より少なかった（構成の数、1 件）。
- **Haiku / rubric**：t6 の全体の文字数が下限を大きく下回った（1 件）。
- **Haiku / self_critique**：t4 で「〜すればよいのか。」という疑問文が文末の条件に違反した（1 件）。見直し前の下書きにも同じ違反があり、見直しでは直りませんでした。

`design.json` の事前の仮説は、「大きいモデルでも、文字数の狭い範囲・語の出現回数・数字の個数のように数えないと守れない制約で違反が残る」というものでした。今回の範囲では、語の回数と数字ではこの仮説は当たりませんでした。全体の文字数の違反も Haiku の 1 件だけです。小さいモデルのほうが違反が多い、という仮説も確かめられませんでした（Opus 全体 82/84、Haiku 全体 81/84）。

### コスト：トークンと時間

| モデル | 条件 | 平均出力トークン | うち thinking | baseline 比 | 平均時間(秒) | 中央値(秒) | baseline 比 | 平均文字数 |
|---|---|---|---|---|---|---|---|---|
| opus | baseline | 3299 | 2812 | 1.00 | 31.6 | 33.7 | 1.00 | 523 |
| opus | rubric | 3062 | 2577 | 0.93 | 29.3 | 29.4 | 0.93 | 522 |
| opus | self_critique | 5775 | 4812 | 1.75 | 56.1 | 53.1 | 1.77 | 521 |
| opus | few_shot | 3325 | 2840 | 1.01 | 35.0 | 33.3 | 1.11 | 528 |
| haiku | baseline | 11236 | 10753 | 1.00 | 56.9 | 47.2 | 1.00 | 524 |
| haiku | rubric | 11835 | 11357 | 1.05 | 60.2 | 53.3 | 1.06 | 520 |
| haiku | self_critique | 17152 | 16184 | 1.53 | 101.6 | 67.5 | 1.79 | 526 |
| haiku | few_shot | 11237 | 10751 | 1.00 | 51.2 | 46.8 | 0.90 | 528 |

両モデルとも、self_critique は呼び出しが二度になるぶん、出力トークンも時間もはっきり増えました。Opus は出力トークン 1.75 倍・時間 1.77 倍、Haiku は 1.53 倍・1.79 倍です。rubric と few_shot は baseline に近い値でした（Opus では 0.93 倍と 1.01 倍、Haiku では 1.05 倍と 1.00 倍）。

本文の長さ（平均文字数）は、どの条件でもほぼ同じでした。出力トークンの大半は thinking です。Haiku は Opus より thinking が多く、時間も長くかかりました。Haiku の self_critique は、平均 101.6 秒に対して中央値 67.5 秒と差が大きく、時間のかかった試行が平均を押し上げています。
<!-- RESULTS:END -->

## 考察：公式ガイドの勧めとどう重なるか

Opus 5.5 では、見直し1回で合格率が上がったのは baseline の不合格ぶんだけで、区間は重なっていました。一方で、トークンと時間は増えています。これは、Opus 5 ガイドの「再チェック指示はコストが増えるだけで結果は良くならない」という記述と矛盾しない結果です。ただし、今回の見直しは別の呼び出しで行っており、ガイドが想定する「同じプロンプトの中で double-check を指示する」形とは違います。また、測ったのは Opus 5 ではなく Opus 5.5 です。Haiku 5.5 では、見直しても合格数は baseline と同じでした。見直し前の下書きにあった文末の違反がそのまま残った例もありました。

![「見直し」の3つの形。同じプロンプトでの再チェック指示（Opus 5 ガイドは非推奨）、別の呼び出しで同じモデルが見直す形（今回測ったのはこれだけ）、別コンテキストの検証役（Fable 5 ガイドは自己批評より有効とする）](/images/a6b5a060ff/b3-verify-forms.png)

採点基準の同梱と few-shot 1例は、コストをほとんど増やさずに、今回は不合格がゼロか1件でした。筆者の見方では、これらは「足しても損が小さい選択肢」とは言えます。しかし、効いたとまでは言えません。

Opus 5.5 の baseline の違反は、t6 の「だ・である調」の文末に集中していました。許される文末を4つに絞った指定に対して、ふつうの動詞の終止形で終わる文が出た、という形です。これはタスク固有の弱点だと考えられます。量産するなら、条件を足して全体を押し上げるより先に、自分のタスクで落ちやすい制約の種類を見つけるほうが役に立つと考えます（推測です）。

## 限界

- 規模が小さい：7 タスク × 3 回 × 2モデルです。不合格は全体で数件しかなく、条件の差は1〜2試行ぶんです。区間は重なっており、条件の順位は付けられません。検定もしていません。
- タスクの範囲：タスクは筆者が作った日本語の業務文7種だけです。ジャンル・長さ・制約の種類が変われば、結果も変わりえます。
- 生成条件：effort / thinking は SDK の既定のままです。Opus 5.5 ガイドは、effort を明示し、複数の水準を自分の eval で試すよう勧めています。水準を変えれば、結果もコストも変わる可能性があります。
- 見直しの形：見直しは1種類の文面・別の呼び出し・1回だけです。同じ会話での見直し、別モデルや別コンテキストの検証役、採点結果を返して直させる形は測っていません。
- few-shot：例は1つ・別題材だけです。例の文言をどれだけ真似たかは測っていません。
- 再現性：シードで固定しているのは実行順だけで、モデルの出力は固定できません。同じ設定で再実行しても、数字は揺れます。所要時間も、並列実行や時間帯の影響を受けます。
- 見直しの効果：見直し前の下書きの合否は raw ログに残っていますが、`summary.json` では集計していません。このため、「見直しで直った件数」「壊れた件数」は示していません。

## 再現方法

コード一式は `experiment/` にあります。`run.mjs` が実行・採点・集計まで行い、`summary.json` は `scores.csv` だけから計算しています。手順は `README.md` に書いたとおりです。

```bash
node experiment/run.mjs experiment validate=1   # 設計の検査（参照解が制約を満たすか）
node experiment/run.mjs experiment pilot=1      # パイロット（experiment/pilot/ に出力）
node experiment/run.mjs experiment              # 本番。止まっても同じコマンドで続きから（resume）
```

条件は `design.json` の `variants` で定義しています。`{task}` にタスク本文、`{rubric}` に採点基準の番号付き一覧、`{example_input}` / `{example_output}` に few-shot の例が入ります。

```json:experiment/design.json
  "variants": [
    {
      "id": "baseline",
      "label": "タスク（条件つき）だけを渡す",
      "template": "{task}",
      "self_critique": false
    },
    {
      "id": "rubric",
      "label": "タスクに加えて、自動採点の基準（制約ごとの判定ルール）を番号付きで渡す",
      "template": "{task}\n\n回答は次の採点基準で自動的に採点されます。\n\n{rubric}",
      "self_critique": false
    },
    {
      "id": "self_critique",
      "label": "1回目の回答のあと、別の呼び出しでタスクと回答を渡して1回だけ見直させる",
      "template": "{task}",
      "self_critique": true
    },
    {
      "id": "few_shot",
      "label": "別の題材のタスクと、その条件をすべて満たした回答例を1つ見せる",
      "template": "以下は、別のタスクと、その条件をすべて満たした回答の例です。書き方の参考にしてください。\n\n# 例のタスク\n{example_input}\n\n# 例の回答\n{example_output}\n\n# あなたのタスク\n{task}",
      "self_critique": false
    }
  ]
```

自分のタスクで試す手順は次のとおりです。

1. `tasks[]` にタスクを追加する（`input`、`checks`、全制約を満たす `reference`）
2. `validate=1` を実行し、制約が矛盾していないことを確かめる
3. `pilot=1` で天井効果（全セル合格）になっていないかを見る
4. 本番を回す

![自分のタスクで試す4ステップ。validate=1 で不合格なら制約を直し、pilot=1 で天井効果なら制約を増やして難しくしてから本番に進む](/images/a6b5a060ff/b3-workflow.png)

`checks` の `type` は、`chars_range`、`count_exact`、`sentence_end`、`numbers` など、`run.mjs` 冒頭のコメントに一覧があります。

## まとめ

- Opus 5.5 / Haiku 5.5、7 タスク × 各 3 回の範囲では、採点基準の同梱・見直し1回・few-shot 1例のどれも、全制約合格率で baseline と差があるとは言えませんでした。
- 見直し1回（別の呼び出し）は、出力トークンと時間を確実に増やしました。採点基準と few-shot 1例では、コストはほぼ増えませんでした。
- 違反は文末表現・段落数・全体の文字数に少しだけ出て、語の回数や数字では出ませんでした。まず baseline で自分のタスクを測り、落ちる制約の種類を見てから、何を足すかを決めるのがよいと考えます。

## 参考資料

- Prompting Claude Opus 5 — Anthropic (Claude Platform Docs) — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5 — 公開日: 不明 — 確認日: 2026-10-16
- Prompting best practices — Anthropic (Claude Platform Docs) — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices — 公開日: 不明 — 確認日: 2026-10-16
- Prompting best practices（旧 Multishot prompting の URL。統合先へ転送） — Anthropic (Claude Platform Docs) — https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/multishot-prompting — 公開日: 不明 — 確認日: 2026-10-16
- Prompting best practices（旧 Chain complex prompts の URL。統合先へ転送） — Anthropic (Claude Platform Docs) — https://docs.claude.com/en/docs/build-with-claude/prompt-engineering/chain-prompts — 公開日: 不明 — 確認日: 2026-10-16
- Prompting Claude Fable 5 — Anthropic (Claude Platform Docs) — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5 — 公開日: 不明 — 確認日: 2026-10-16
- Prompting Claude Opus 5.5 — Anthropic (Claude Platform Docs) — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5 — 公開日: 不明 — 確認日: 2026-10-16
- Claude Platform release notes — Anthropic (Claude Platform Docs) — https://platform.claude.com/docs/en/release-notes/overview.md — 公開日: 不明（随時更新） — 確認日: 2026-10-16
- Introducing Claude Opus 5 — Anthropic — https://www.anthropic.com/news/claude-opus-5 — 公開日: 2026-07-24 — 確認日: 2026-10-16
- Agent SDK overview — Anthropic (Claude Code Docs) — https://code.claude.com/docs/en/agent-sdk/overview — 公開日: 不明 — 確認日: 2026-10-16
- Agent SDK reference - TypeScript — Anthropic (Claude Code Docs) — https://code.claude.com/docs/en/agent-sdk/typescript — 公開日: 不明 — 確認日: 2026-10-16
- claude-agent-sdk-typescript CHANGELOG.md — Anthropic (GitHub) — https://github.com/anthropics/claude-agent-sdk-typescript/blob/main/CHANGELOG.md — 公開日: 不明（各版に日付なし） — 確認日: 2026-10-16

運営: AIみちしるべ（https://ai.sanvia.co.jp）
