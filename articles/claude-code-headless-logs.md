---
title: "Claude Codeをヘッドレスで回すときのstream-jsonログの読み方と集計"
emoji: "🪵"
type: "tech"
topics: ["claudecode", "claude", "llm", "cli", "python"]
published: true
---

Claude Code を `claude -p --output-format stream-json --verbose` で毎日自動実行していると、1実行につき数百行の JSON ログが溜まります。この記事では、その stream-json ログの**行の種類と各フィールドの意味**を公式ドキュメントで確認しながら、**完了した実行だけを抜き出して集計する小さな Python スクリプト**を作り、手元の実ログ 42 実行に対して実際に流した出力をそのまま載せます。数値はすべてこのスクリプトの出力から転記しています。

![積み上げられた薪の木口が多数並ぶ写真。大小さまざまな丸太の断面がぎっしり詰まっている。多数の実行ログ（英語でlog＝丸太）を1本ずつ数え、断面＝1回の実行結果を見比べて集計することのメタファー](/images/cc-headless-logs/hero-bbd2ed64f7.jpg)

対象は、同一マシンで動かした `*.jsonl` ログ 114 ファイル（Claude Code v2.1.197）。結論だけ先に:

- 集計に使う数値は、ログ末尾の `type:"result"` 行と先頭の `system/init` 行にほぼ集約されている。
- 完了判定は「`result` 行があり、かつ `num_turns >= 1`」で十分。114 ファイルのうち `result` 行が無い（実行中/中断）のが 62 ファイル。残り 52 ファイルが `result` 行を持ち、うち **42 ファイルが完了実行**（今回は 1 ファイル 1 実行）で成功 35・`is_error:true` 7、残り 10 ファイルは `num_turns == 0` の空 `result`（空 `result` 行自体は 19）だった。
- 成功 35 実行のキャッシュ読み出し率は中央値 98.59%（範囲 92.1〜99.8%）。入力トークンのほとんどがプロンプトキャッシュの読み出しだった。

料金そのものの集計（API換算額の内訳）は別記事（[AIみちしるべ: Claude Code 料金](https://ai.sanvia.co.jp/articles/claude-code-pricing)）でやっているので、ここでは**ログの読み方と集計の作り方**に絞ります。

## stream-json ログの構造

`--output-format stream-json` は、1行ごとに1つの JSON オブジェクトを吐く newline-delimited JSON です（[公式: Run Claude Code programmatically](https://code.claude.com/docs/en/headless)、確認日 2026-10-20）。各行は `type` フィールドで区別します。代表的な実行ログ1件（日次ジョブ（daily）の1実行）の行の内訳は次の通りでした。

| type | 件数 | 中身 |
|---|---|---|
| system | 425 | `subtype:"init"` のほか `thinking_tokens` や `task_started` 等 |
| assistant | 168 | モデルの応答・ツール呼び出し |
| user | 81 | ツール実行結果など |
| rate_limit_event | 1 | レート制限の状態 |
| result | 1 | 最後の1行。集計のキモ |

並びは「先頭に `system/init` が1行 → `assistant`／`user` のツール実行ループが本体 → 末尾に `result` が1行」という形です。公式ドキュメントも「The last line of the stream is a `result` message with the final response text, cost, and session metadata」「The `system/init` event ... is the first event in the stream」と書いています（[headless](https://code.claude.com/docs/en/headless)、確認日 2026-10-20）。

![stream-jsonの行の流れの模式図。左のsystem/init（model・version・permissionMode）から、中央のassistantとuserのツール実行ループ（systemやrate_limit_eventを含む）を経て、右のresult（num_turns・duration・usage・cost）へ矢印が伸びる。集計に使うのは先頭initと末尾resultの数値だけ](/images/cc-headless-logs/fig3-flow-13f61f2630.png)

### 先頭の `system/init` 行

セッションのメタデータが入ります。集計に使うのは次の3つくらいです（値はダミー、構造のみ）。

```json
{"type":"system","subtype":"init",
 "claude_code_version":"2.1.197",
 "model":"claude-opus-4-8",
 "permissionMode":"bypassPermissions",
 "session_id":"…","cwd":"…","tools":["…"],"slash_commands":["…"]}
```

- `model`: そのセッションの主モデル。
- `permissionMode`: 権限モード。今回の対象ログはすべて `bypassPermissions`（＝人の承認を挟まない無人実行）でした。
- `claude_code_version`: Claude Code の版。今回は全ログ `2.1.197`。手元の `claude --version` も `2.1.197 (Claude Code)` でした。

`model` は [CLI reference](https://code.claude.com/docs/en/cli-reference) の `--model`、`permissionMode` は権限モードに対応します。`system/init` が何を含むかは [headless の "Read session metadata"](https://code.claude.com/docs/en/headless)（確認日 2026-10-20）に説明があります。

### 末尾の `result` 行

実行全体のサマリです。Agent SDK の `SDKResultMessage` 型にフィールド定義があります（[Agent SDK: TypeScript リファレンス](https://code.claude.com/docs/en/agent-sdk/typescript)、確認日 2026-10-20）。構造だけ示すと（値はダミー）:

```json
{"type":"result","subtype":"success","is_error":false,
 "num_turns":"…","duration_ms":"…","duration_api_ms":"…",
 "total_cost_usd":"…",
 "usage":{"input_tokens":"…","cache_creation_input_tokens":"…",
          "cache_read_input_tokens":"…","output_tokens":"…"},
 "modelUsage":{"claude-opus-4-8":{"inputTokens":"…","outputTokens":"…",
     "cacheReadInputTokens":"…","cacheCreationInputTokens":"…","costUSD":"…"},
   "claude-haiku-4-5-20251001":{"…":"…"}},
 "permission_denials":[],"session_id":"…"}
```

主なフィールドの意味（出典はすべて [Agent SDK: TypeScript](https://code.claude.com/docs/en/agent-sdk/typescript) / [headless](https://code.claude.com/docs/en/headless) / [costs](https://code.claude.com/docs/en/costs)、確認日 2026-10-20）:

| フィールド | 意味 |
|---|---|
| `subtype` | `success` / `error_max_turns` / `error_during_execution` / `error_max_budget_usd` / `error_max_structured_output_retries` |
| `is_error` | エラー終了かどうかの真偽値 |
| `num_turns` | エージェントのターン数（ツール往復の回数） |
| `duration_ms` | 実行全体の実時間（ミリ秒） |
| `duration_api_ms` | うち API 呼び出しに費やした時間（ミリ秒） |
| `usage.input_tokens` | 新規の入力トークン |
| `usage.cache_creation_input_tokens` | キャッシュ書き込み分の入力トークン |
| `usage.cache_read_input_tokens` | キャッシュ読み出し分の入力トークン |
| `usage.output_tokens` | 出力トークン |
| `modelUsage` | モデル別の内訳（サブエージェントのモデル分を含む） |
| `total_cost_usd` | **API換算の推定額**（後述） |
| `permission_denials` | 却下されたツール呼び出し |

`modelUsage` の各モデルは `inputTokens` / `outputTokens` / `cacheReadInputTokens` / `cacheCreationInputTokens` / `costUSD` 等を持ちます。主モデルに加えて別のモデル（今回は `claude-haiku-4-5-20251001`）が入ることがあるので、主モデル以外の利用分を切り分けられます。docs によれば `modelUsage` は主ループに加えサブエージェントや compaction 等の内部呼び出しも含みます（[Agent SDK: TypeScript](https://code.claude.com/docs/en/agent-sdk/typescript)、確認日 2026-10-20）。

`result` 行にはこのほか診断用のフィールドがあり、[Agent SDK: TypeScript](https://code.claude.com/docs/en/agent-sdk/typescript) の「Several fields on the result carry diagnostic detail beyond `subtype`」節に定義があります（確認日 2026-10-20）。ログで観測したものを docs と突き合わせると:

- `terminal_reason`: ループが終わった理由の列挙（`completed` / `max_turns` / `api_error` など）。
- `api_error_status`: 会話を終了させた API エラーの HTTP ステータスコード。API エラーなく終わったときは `null` または省略。
- `ttft_ms`: 最初のトークンまでの時間（ミリ秒）。success アームのみ。`ttft_stream_ms` は最初の `message_start` までの時間。
- `stop_reason`: モデル側の停止理由（文字列 or `null`）。
- `fast_mode_state`: `on` / `off` / `cooldown`。
- `usage` 内の `service_tier`（`standard` / `priority` / `batch` / `null`）、`iterations`、`server_tool_use` もドキュメントに定義があります。

一方、ログに現れた `time_to_request_ms` は、上記ドキュメントで定義を確認できませんでした（`request_sent_wall_ms` 等は定義あり）。確認できないフィールドは意味を推測しません。

### `total_cost_usd` はあくまで推定値

`total_cost_usd` は請求額ではありません。公式の [Manage costs effectively](https://code.claude.com/docs/en/costs)（確認日 2026-10-20）は「The figure is an estimate, so for authoritative billing see the Usage page in the Claude Console」と明記し、さらに「Claude Max and Pro subscribers have usage included in their subscription, so the session cost figure isn't relevant for billing purposes」としています。headless 側も「Both figures are client-side estimates and can differ from your actual bill」と書いています（[headless](https://code.claude.com/docs/en/headless)）。この記事の金額は**API換算の目安**として扱い、詳しい内訳は[料金記事](https://ai.sanvia.co.jp/articles/claude-code-pricing)に回します。

## 集計スクリプト

やることは単純です。各 `.jsonl` を1行ずつ読み、`system/init` 行から主モデル・版・権限モードを、`result` 行から数値フィールドを拾う。**プロンプト本文・ツール入出力・ファイル内容は一切読みません**（読むのは `type` と数値フィールドだけ）。完了判定は「`result` 行があり `num_turns >= 1`」。標準ライブラリだけで動きます。

```python
#!/usr/bin/env python3
"""Claude Code の headless 実行ログ（stream-json, 1行1 JSON）を集計する。

使い方:
    python3 aggregate_cc_logs.py <ログの入ったディレクトリ or *.jsonl ...>

各ログの末尾付近にある `type":"result"` 行の**数値フィールド**と、
先頭の `system/init` 行の model / version / permissionMode だけを読む。
プロンプト本文・ツール入出力・ファイル内容などは一切読まない。
完了済み（result 行があり、num_turns>=1 ＝実作業が1ターン以上ある）実行だけを
対象にする。result 行が無いログ（＝実行中 or 中断）と、num_turns==0 の空の
result 行（docs ではキューされたバックグラウンドタスク完了で生じうると説明）は
件数だけ数えて除外する。

出力: 標準出力に要約、同じフォルダに runs.csv と summary.json。
"""
import csv
import glob
import json
import os
import statistics
import sys


def classify(stem: str) -> str:
    """ファイル名（拡張子なし）からジョブ種別をざっくり決める。
    ファイル名トークンだけで判断し、タスク内容までは推測しない。"""
    s = stem.lower()
    if "daily" in s:
        return "daily"            # 日次ジョブ(daily)
    if "note" in s:
        return "note"             # note 記事
    if "breaking" in s:
        return "breaking"         # 速報
    if "weekly" in s:
        return "weekly"           # 週次レビュー
    if "zenn" in s:
        return "zenn"             # Zenn 記事
    if "article" in s:
        return "article"          # サイト記事
    return "other"                # 種別不明（検証ランなど）


def iter_files(args):
    for a in args:
        if os.path.isdir(a):
            yield from sorted(glob.glob(os.path.join(a, "*.jsonl")))
        else:
            yield from sorted(glob.glob(a))


def pct(num, den):
    return (num / den) if den else 0.0


def collect(paths):
    runs = []
    n_files = n_empty = n_incomplete = n_zero_turn = 0
    for path in paths:
        n_files += 1
        init_model = init_ver = init_perm = None
        has_result = False
        parsed_any = False
        with open(path, encoding="utf-8") as fh:
            for line in fh:
                line = line.strip()
                if not line:
                    continue
                try:
                    obj = json.loads(line)
                except json.JSONDecodeError:
                    continue
                parsed_any = True
                t = obj.get("type")
                if t == "system" and obj.get("subtype") == "init":
                    init_model = obj.get("model")
                    init_ver = obj.get("claude_code_version")
                    init_perm = obj.get("permissionMode")
                elif t == "result":
                    has_result = True
                    if (obj.get("num_turns", 0) or 0) < 1:
                        n_zero_turn += 1   # 実作業ゼロの result は除外
                        continue
                    u = obj.get("usage", {}) or {}
                    inp = u.get("input_tokens", 0) or 0
                    cw = u.get("cache_creation_input_tokens", 0) or 0
                    cr = u.get("cache_read_input_tokens", 0) or 0
                    out = u.get("output_tokens", 0) or 0
                    mu = obj.get("modelUsage", {}) or {}
                    # primary = init の model。無ければ出力トークン最大のモデル。
                    primary = init_model
                    if primary not in mu and mu:
                        primary = max(mu, key=lambda m: mu[m].get("outputTokens", 0))
                    sub_out = sum(v.get("outputTokens", 0)
                                  for k, v in mu.items() if k != primary)
                    tot_out = sum(v.get("outputTokens", 0) for v in mu.values())
                    dur = obj.get("duration_ms", 0) or 0
                    api = obj.get("duration_api_ms", 0) or 0
                    runs.append({
                        "file": os.path.basename(path),
                        "job": classify(os.path.splitext(os.path.basename(path))[0]),
                        "model": primary or "unknown",
                        "is_error": bool(obj.get("is_error")),
                        "subtype": obj.get("subtype"),
                        "num_turns": obj.get("num_turns", 0) or 0,
                        "duration_ms": dur,
                        "duration_api_ms": api,
                        "api_ratio": round(pct(api, dur), 4),
                        "input_tokens": inp,
                        "cache_creation_input_tokens": cw,
                        "cache_read_input_tokens": cr,
                        "output_tokens": out,
                        "cache_read_rate": round(pct(cr, inp + cw + cr), 4),
                        "total_cost_usd": round(obj.get("total_cost_usd", 0) or 0, 4),
                        "n_models": len(mu),
                        "subagent_out_share": round(pct(sub_out, tot_out), 4),
                        "cc_version": init_ver or "unknown",
                        "permission_mode": init_perm or "unknown",
                    })
        if not parsed_any:
            n_empty += 1
        elif not has_result:
            n_incomplete += 1
    return runs, n_files, n_empty, n_incomplete, n_zero_turn


def med(xs):
    return round(statistics.median(xs), 2) if xs else 0


def main():
    args = sys.argv[1:] or ["*.jsonl"]
    paths = list(iter_files(args))
    runs, n_files, n_empty, n_incomplete, n_zero_turn = collect(paths)
    out_dir = os.path.dirname(os.path.abspath(__file__))

    ok = [r for r in runs if not r["is_error"]]
    err = [r for r in runs if r["is_error"]]

    print(f"scanned files        : {n_files}")
    print(f"  empty/unparsable   : {n_empty}")
    print(f"  no result (running): {n_incomplete}")
    print(f"  zero-turn results  : {n_zero_turn}  (excluded)")
    print(f"completed runs       : {len(runs)}  (success={len(ok)}, is_error={len(err)})")
    print()

    by_model = {}
    for r in runs:
        by_model[r["model"]] = by_model.get(r["model"], 0) + 1
    print("runs by primary model:")
    for m, c in sorted(by_model.items(), key=lambda kv: -kv[1]):
        print(f"  {m:<28} {c}")
    print()

    # success 実行だけでジョブ種別ごとに中央値を出す
    print("per-job medians (success runs only):")
    hdr = ("job", "n", "turns", "dur_min", "api/wall", "cache_read%", "out_tok", "cost_usd")
    print("  " + " ".join(f"{h:>11}" for h in hdr))
    jobs = sorted({r["job"] for r in ok})
    job_rows = []
    for job in jobs:
        g = [r for r in ok if r["job"] == job]
        row = {
            "job": job,
            "n": len(g),
            "turns": med([r["num_turns"] for r in g]),
            "dur_min": med([r["duration_ms"] / 60000 for r in g]),
            "api_wall": med([r["api_ratio"] for r in g]),
            "cache_read_pct": med([r["cache_read_rate"] * 100 for r in g]),
            "out_tok": int(med([r["output_tokens"] for r in g])),
            "cost_usd": med([r["total_cost_usd"] for r in g]),
        }
        job_rows.append(row)
        print("  " + " ".join(f"{v:>11}" for v in (
            row["job"], row["n"], row["turns"], row["dur_min"],
            row["api_wall"], row["cache_read_pct"], row["out_tok"], row["cost_usd"])))
    print()

    allc = [r["cache_read_rate"] * 100 for r in ok]
    print("overall (success runs):")
    print(f"  cache_read% median {med(allc)}  (min {round(min(allc),1)}, max {round(max(allc),1)})")
    subs = [r for r in ok if r["n_models"] > 1]
    print(f"  runs using a subagent model : {len(subs)}/{len(ok)}")
    if subs:
        sh = [r["subagent_out_share"] * 100 for r in subs]
        print(f"  subagent output share% median {med(sh)}  (min {round(min(sh),1)}, max {round(max(sh),1)})")
    costs = [r["total_cost_usd"] for r in ok]
    print(f"  total_cost_usd (estimate) median {med(costs)}  (min {round(min(costs),2)}, max {round(max(costs),2)})")

    # 保存
    with open(os.path.join(out_dir, "runs.csv"), "w", newline="") as fh:
        w = csv.DictWriter(fh, fieldnames=list(runs[0].keys()))
        w.writeheader()
        w.writerows(runs)
    summary = {
        "scanned_files": n_files,
        "empty_or_unparsable": n_empty,
        "no_result_incomplete": n_incomplete,
        "zero_turn_results_excluded": n_zero_turn,
        "completed_runs": len(runs),
        "success_runs": len(ok),
        "error_runs": len(err),
        "runs_by_model": by_model,
        "per_job_medians_success": job_rows,
        "cache_read_pct_median": med(allc),
        "subagent_runs": len(subs),
        "total_cost_usd_median_success": med(costs),
    }
    with open(os.path.join(out_dir, "summary.json"), "w") as fh:
        json.dump(summary, fh, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    main()
```

ジョブ種別は**ファイル名のトークン**（`daily` / `note` / `breaking` など）だけで決め、タスクの中身までは推測しません。判定は上から順（`daily`→`note`→`breaking`→`weekly`→`zenn`→`article`）で、最初に当たったトークンを採用します。したがって `cc_note_breaking_*` は `note` に分類されます。該当トークンが無いファイル（反復の検証ランなど）は `other`（種別不明）に寄せています。

注意として、このラベルは**ファイル名トークンでの粗い分類**であり、同じラベルでも中身は一様ではありません。たとえば `daily` には、命名の違う 2 系統（`cc_daily_*` と `claude_daily_*`）が混在します。以下では「日次ジョブ（daily）」のように、ファイル名トークンでまとめたものだと分かる呼び方をします。

## 実ログに流した結果

上のスクリプトを、同一マシンのログ置き場2か所（`*.jsonl` が 114 ファイル）に対して実行した生出力がこれです。以下の数値はすべてこの出力からの転記です（2026-10-20 時点のスナップショット。ログは実行のたびに増えるので、後から流すと `scanned files`・除外件数・完了実行数・各中央値のいずれも変わりえます）。

```text
scanned files        : 114
  empty/unparsable   : 0
  no result (running): 62
  zero-turn results  : 19  (excluded)
completed runs       : 42  (success=35, is_error=7)

runs by primary model:
  claude-sonnet-5              29
  claude-opus-4-8              12
  claude-opus-5                1

per-job medians (success runs only):
          job           n       turns     dur_min    api/wall cache_read%     out_tok    cost_usd
      article           1          98       36.45        0.66       98.24       87050       11.89
     breaking           3          95       27.48        0.62        97.8       72752        6.97
        daily           9          69       29.65        0.51       98.76       43881        8.12
         note           4        38.0        7.93        0.97       96.68       33540        5.26
        other          17          49       10.95        0.66       99.29       35104       10.15
       weekly           1          34        9.38        0.98       96.06       41306        3.43

overall (success runs):
  cache_read% median 98.59  (min 92.1, max 99.8)
  runs using a subagent model : 19/35
  subagent output share% median 0.24  (min 0.0, max 10.2)
  total_cost_usd (estimate) median 8.12  (min 0.78, max 31.52)
```

読み取れること（`article` と `weekly` は n=1 なので中央値ではなく1件の値です）:

- **何件が集計対象か**が明確になる。114 ファイルのうち 62 は `result` 行が無い（実行中か中断）。残り 52 ファイルが `result` 行を持ち、そのうち 42 ファイルは `num_turns>=1` の完了実行（今回は 1 ファイル 1 実行）、10 ファイルは `num_turns==0` の空 `result` しか持ちません（空 `result` 行の総数は 19＝同じファイルに複数出ることがある）。完了 42 のうち 35 が成功、7 が `is_error:true`。
- **主モデルは混在**。`claude-sonnet-5` 29・`claude-opus-4-8` 12・`claude-opus-5` 1。モデル間の厳密比較が目的ではなく、「同一環境のログをどう読むか」の例です。

### ジョブ種別ごとのターン数と実時間

`num_turns` が多い実行ほど実時間も長い、という素直な関係が出ました（n≥3 のジョブのみ図示）。

![ジョブ種別ごとのターン数と実時間の中央値の棒グラフ。日次ジョブ(daily)は69ターン・30分、速報(breaking)は95ターン・27分、種別不明の検証ランは49ターン・11分、note系(note)は38ターン・8分](/images/cc-headless-logs/fig1-jobs-87a1ff473c.png)

`api/wall`（＝`duration_api_ms` / `duration_ms`）は、実時間のうち API 呼び出しに費やした割合です（`duration_api_ms` の定義より）。note（0.97）や週次（0.98）はほぼ API 時間で占められ、日次ジョブ（daily, 0.51）は API 待ち以外の時間（ツール実行など）が実時間の約半分を占めていました。

### 入力トークンはほぼキャッシュ読み出し

一番はっきり出たのはこれです。`usage` の内訳を見ると、入力トークンの大半は `cache_read_input_tokens`（キャッシュ読み出し）でした。成功 35 実行の `cache_read` 率は中央値 98.59%（最小 92.1%・最大 99.8%）。下図の 4 種別（n≥3）で合計すると 97.5〜99.2% がキャッシュ読み出しです。

![入力トークンの内訳を100%積み上げ横棒で示した図。新規入力は各ジョブ0.1%未満・キャッシュ書込も数%で棒はほとんど見えず、日次ジョブ(daily)99.1%・種別不明99.2%・速報(breaking)97.8%・note系(note)97.5%がキャッシュ読み出し](/images/cc-headless-logs/fig2-tokens-bad224cd98.png)

長い自動実行では会話履歴が毎リクエスト送られ、その大半がプロンプトキャッシュから読み直されるため、こうなります（背景は [Prompt caching](https://code.claude.com/docs/en/prompt-caching)、確認日 2026-10-20）。新規入力トークンは全体の 0.1% 未満でした。

### 主モデル以外のモデル（サブエージェント等）

`modelUsage` に主モデル以外のモデルが現れた実行は 35 中 19。ただし、その主モデル以外のモデルが生成した**出力トークンの割合**は中央値 0.24%（最大でも 10.2%）でした。言えるのは「出力トークンの大半は主モデルのもの」というところまでで、そのモデルが具体的に何に使われたかはログの数値からは分かりません。公式は「choose a smaller model for a subagent」でサブエージェントに小さいモデルを割り当ててコストを抑える方法を案内しています（[costs](https://code.claude.com/docs/en/costs)、確認日 2026-10-20）。

## 自分のログで試すときのコツ

- ログの収集は `claude -p "<task>" --output-format stream-json --verbose > run.jsonl` のようにリダイレクトするだけ。CI やスクリプトでは `--bare` を付けると、hooks・skills・カスタムコマンド・サブエージェント・プラグイン・MCP サーバー・auto memory・CLAUDE.md の自動読み込みをスキップして起動が速くなります（スクリプト／SDK 呼び出しで推奨されるモード。[headless](https://code.claude.com/docs/en/headless)、確認日 2026-10-20）。
- 集計では**まず件数を数える**。「`result` 行がある」「`num_turns >= 1`」の2条件で、実行中・中断・空実行を落とすのが実務的です。`num_turns==0` の空 `result` を含めると、中央値がゼロ側に引っ張られます。
- 金額は `total_cost_usd` を鵜呑みにしない。推定値であること（前述の公式引用）を踏まえ、請求は Console の Usage ページで確認します。

## 限界

- サンプルは完了実行 42（成功 35）のみ。同じ環境・同じ Claude Code 版（v2.1.197）で集めたログですが、主モデルが混在（sonnet/opus）し、タスク内容もジョブ種別ごと・ファイルごとに違います。一般化はできません。
- `other`（17 件）はファイル名から種別を特定できない反復の検証ランで、タスクの中身は不明です（意味づけしていません）。`article`・`weekly` は各 1 件で、中央値ではなく単一の値です。
- `total_cost_usd` は API 換算の推定値で、請求額ではありません。サブスク（Pro/Max）では使用枠に対して消費され、ドル請求とは別です（[costs](https://code.claude.com/docs/en/costs)）。
- ログに現れても公式ドキュメントで確認できなかったフィールドは、意味を推測していません。
- プライバシーのため、この集計ではプロンプト本文・ツール入出力・ファイル内容・セッションID等は読まず、数値フィールドとフィールド名・モデル名・版だけを扱っています。

## よくある質問

### Q. 実行が完了したかどうかは、どのフィールドで判断すればいいですか

末尾に `type:"result"` 行があるかどうかです。SIGTERM で止めた実行は `result` を残さないと公式に明記されています（[headless の "Stop a run with SIGTERM"](https://code.claude.com/docs/en/headless)、確認日 2026-10-20）。また、`num_turns:0` の空の `result` 行が出ることがあります。ドキュメントには「複数のバックグラウンドタスク完了がまとめてキューされると、Claude Code はそれらを 1 ターンで応答でき、最後の 1 つを除く各完了は `num_turns: 0` の空の結果を順に出す」という記述があります（[Agent SDK: TypeScript](https://code.claude.com/docs/en/agent-sdk/typescript)、確認日 2026-10-20）。今回のログにも空 `result` が 19 行ありましたが、それがこのケースに当たるかは数値フィールドだけからは確認できません。いずれにせよ実務では「`result` があり `num_turns >= 1`」を完了条件にすると安全です。

### Q. `is_error` が true でも `subtype` が "success" のことがあるのはなぜですか

矛盾ではありません。`SDKResultMessage` 型では、`subtype: "success"` のアーム自体に `is_error: boolean` フィールドが定義されており（[Agent SDK: TypeScript](https://code.claude.com/docs/en/agent-sdk/typescript)、確認日 2026-10-20）、型定義上は両立しえます（`subtype` の値は `success` / `error_max_turns` / `error_during_execution` / `error_max_budget_usd` / `error_max_structured_output_retries`）。今回の `is_error:true` の 7 実行は、いずれも `subtype:"success"`・`terminal_reason:"completed"` で、`api_error_status` は 1 件が `429`、残り 6 件が `null` でした（列挙値・数値フィールドのみ集計。エラーメッセージ本文は読んでいません）。`api_error_status` は「会話を終了させた API エラーの HTTP ステータス」なので、`429` の 1 件は API エラー（HTTP 429）で終了したと読めますが、`null` の 6 件がなぜ `is_error:true` かはドキュメントから特定できないため推測しません。集計では `is_error` を成功判定の基準にしておくのが無難です。

### Q. `total_cost_usd` を合計すれば請求額になりますか

なりません。公式が「The figure is an estimate, so for authoritative billing see the Usage page in the Claude Console」と明記しています（[costs](https://code.claude.com/docs/en/costs)、確認日 2026-10-20）。API キー利用時の換算の目安であり、Pro/Max サブスクでは使用枠に対して消費されます。金額の内訳の見方は[Claude Code 料金](https://ai.sanvia.co.jp/articles/claude-code-pricing)にまとめています。

### Q. キャッシュ読み出しが 98% もあるのは異常ですか

いいえ。長い自動実行では会話履歴を毎リクエスト送るため、その大半がプロンプトキャッシュから読み直されます（[Prompt caching](https://code.claude.com/docs/en/prompt-caching) / [costs の "Why usage climbs in a long session"](https://code.claude.com/docs/en/costs)、確認日 2026-10-20）。今回の成功 35 実行でも 92〜99.8% の範囲に収まりました。むしろキャッシュ読み出し率が極端に低い実行があれば、キャッシュミス（セッションを空けすぎた等）を疑う手がかりになります。

## 参考資料

いずれも本文で参照した一次ソース。公式ドキュメントの各ページは公表日の表示がないため「公表日: 不明」。

- Run Claude Code programmatically（headless）— Anthropic（code.claude.com/docs）— https://code.claude.com/docs/en/headless — 公表日: 不明 — 確認日: 2026-10-20
- Manage costs effectively（costs）— Anthropic — https://code.claude.com/docs/en/costs — 公表日: 不明 — 確認日: 2026-10-20
- Agent SDK reference: TypeScript（`SDKResultMessage` 等の型定義）— Anthropic — https://code.claude.com/docs/en/agent-sdk/typescript — 公表日: 不明 — 確認日: 2026-10-20
- CLI reference — Anthropic — https://code.claude.com/docs/en/cli-reference — 公表日: 不明 — 確認日: 2026-10-20
- Prompt caching — Anthropic — https://code.claude.com/docs/en/prompt-caching — 公表日: 不明 — 確認日: 2026-10-20
- Claude Code 料金（自サイトの実測記事）— AIみちしるべ（株式会社SANVIA）— https://ai.sanvia.co.jp/articles/claude-code-pricing — 公表日: 不明 — 確認日: 2026-10-20

---

筆者: Kosuke。AIみちしるべ（[https://ai.sanvia.co.jp](https://ai.sanvia.co.jp)）を運営しています。[Claude Code の料金を実測した記事](https://ai.sanvia.co.jp/articles/claude-code-pricing)も書いています（[著者ページ](https://ai.sanvia.co.jp/authors/kosuke)）。この記事の集計スクリプトと生出力（数値のみの CSV/JSON）は手元に保存してあり、本文のスクリプトと同一です。
