---
title: "codex execの--output-schemaで見積書・請求書から10項目を抜き出し、60回実測した精度と時間"
emoji: "🔎"
type: "tech"
topics: ["codex", "openai", "jsonschema", "python", "llm"]
published: true
---

架空の見積書・請求書10件から10項目を抜き出す作業を、Codex CLI の `codex exec` で3条件×2回（計60回）回しました。項目の正答率は98.0〜99.0%で、間違いは2件の書類の「期限」と「登録番号」だけでした。JSON Schema 指定（`--output-schema`）の有無では、今回の60回に限ると JSON の壊れも正答率の差もほとんど出ていません。

![古い木枠のふるいが掘り返した黒い土の上に置かれ、金網の上に小石がいくつかだけ残り、細かい土は下に落ちている写真。書類の文字の山から必要な項目だけを取り出して確かめる作業のメタファー](/images/codex-invoice-extract/hero-9c9d6fe9e5.jpg)

以下の数値は、すべてこの記事に載せた集計スクリプトの生出力から転記しています。書類・正解データ・生ログ・スクリプトは手元に保存してあります。

## 試した環境と条件

```text
$ codex --version
codex-cli 0.163.0
```

- 実行日時: 2026-10-23 10:46〜10:51（JST）、同じ Linux マシンで60回を1回ずつ順番に実行
- 認証: ChatGPT アカウントでのサインイン（`codex login`）
- 書類: この記事のために作った架空のダミー書類10件（テキスト）。会社名・金額・登録番号はすべて架空で、実在の事業者とは関係ありません
- 抽出する項目: 書類種別、書類番号、発行日、期限、発行元、宛先、登録番号、税抜合計、消費税額、税込合計の10項目
- 条件（各条件10件×2回＝20回）

| 条件名 | モデル | `--output-schema` |
|---|---|---|
| luna_schema | `gpt-6-luna` | あり |
| luna_noschema | `gpt-6-luna` | なし（同じ指示文で JSON を返すよう頼むだけ） |
| sol_schema | `gpt-6.1-sol` | あり |

モデルは公式ドキュメント「Models」で、GPT-6 Luna が要約・抽出などの定型の大量処理向け、GPT-6.1 Sol が複雑なコーディング向けの推奨として挙がっている2つを選びました（[Models](https://learn.chatgpt.com/docs/models)、確認日 2026-10-23）。推論レベルは指定していません。

なお、Claude Code にも `claude -p --output-format json --json-schema '<schema>'` で、結果を `structured_output` フィールドに受け取る同等の機能があります（[Run Claude Code programmatically](https://code.claude.com/docs/en/headless)、確認日 2026-10-23）。今回の環境では Claude Code の CLI がログインしていない状態だったため、計測は Codex CLI だけで行いました。

## ダミー書類：実務で出る揺れを1件ずつ入れた

d01 を基準にして、残りの9件には、請求書処理でよく見かける表記や計算の揺れを1〜3個ずつ入れています。

| 書類 | 種別 | 意図して入れた揺れ |
|---|---|---|
| d01 | 見積書 | 基準（西暦・半角・税抜表示・10%のみ） |
| d02 | 請求書 | 和暦（令和８年）、全角数字、全角ハイフンの請求番号 |
| d03 | 請求書 | 8%と10%の混在（軽減税率）、宛先が「様」 |
| d04 | 見積書 | 税込表示のみ（税抜合計は書面になく、税込−消費税で求まる） |
| d05 | 請求書 | 請求金額が大字（金壱拾参万弐阡円也）、日付が `2026.10.5`、登録番号に空白 |
| d06 | 請求書 | 発行元が `(株)` の略記、値引き行（マイナス） |
| d07 | 見積書 | 有効期限が「発行日より30日間」（日付が書かれていない） |
| d08 | 請求書 | 和暦の略記 `R8.10.1`、支払期限「10月末日」、登録番号が全角・ハイフン区切り |
| d09 | 請求書 | 単価に小数（12.5）、消費税の1円未満切り捨て |
| d10 | 見積書 | 書類番号なし、登録番号なし（免税事業者の注記）、「前回お見積額」という紛らわしい金額 |

一番短い d08 の全文はこうです。

```text:docs/d08.txt
ご請求書
請求番号 TL-808          発行日 R8.10.1
株式会社ダミーメディカル 御中
                               合同会社テストラボ
                               登録番号 Ｔ７－７７７７－００００－８８８８

検体分析     15件 × 4,000円 = 60,000円
レポート作成  1式            = 20,000円
小計 80,000円　消費税(10%) 8,000円　合計 88,000円
お支払期限：10月末日までに下記口座へお振込みください。
```

正解データ（`truth/d08.json`）はこうしました。

```json:truth/d08.json
{
  "doc_type": "請求書",
  "document_number": "TL-808",
  "issue_date": "2026-10-01",
  "due_date": "2026-10-31",
  "issuer_name": "合同会社テストラボ",
  "recipient_name": "株式会社ダミーメディカル",
  "registration_number": "T7777700008888",
  "subtotal_excl_tax": 80000,
  "tax_amount": 8000,
  "total_incl_tax": 88000
}
```

正解の決め方で迷ったのは期限です。d08 の「10月末日」は年が書かれていませんが、発行日（R8.10.1）から 2026-10-31 と決まるので日付を正解にしました。d07 の「発行日より30日間」は、初日を数えるかどうかで 10-02 とも 10-03 とも読めるため `null` を正解にしています。この決め方そのものが結果に効いたので、後で詳しく書きます。

また、正解との突き合わせでは、文字列を NFKC 正規化して空白を除いたうえで比べています。発行元の表記は「書面の表記どおり」と頼みましたが、d06 の `(株)テストシステムズ` を `株式会社テストシステムズ` と展開した場合と、d10 で「（個人事業）」まで含めた場合も正解として扱いました（正解 JSON の `_alt`）。金額は整数型での完全一致です。

## スキーマと指示文

`--output-schema` に渡したスキーマです。公式の例にならい、全項目を `required` にして `additionalProperties: false` を付け、無い場合は `null` を許す形にしました（[Non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)、確認日 2026-10-23）。

```json:schema.json
{
  "type": "object",
  "properties": {
    "doc_type": {"type": "string", "enum": ["見積書", "請求書"]},
    "document_number": {"type": ["string", "null"]},
    "issue_date": {"type": ["string", "null"]},
    "due_date": {"type": ["string", "null"]},
    "issuer_name": {"type": "string"},
    "recipient_name": {"type": "string"},
    "registration_number": {"type": ["string", "null"]},
    "subtotal_excl_tax": {"type": ["integer", "null"]},
    "tax_amount": {"type": ["integer", "null"]},
    "total_incl_tax": {"type": ["integer", "null"]}
  },
  "required": ["doc_type", "document_number", "issue_date", "due_date",
               "issuer_name", "recipient_name", "registration_number",
               "subtotal_excl_tax", "tax_amount", "total_incl_tax"],
  "additionalProperties": false
}
```

指示文は3条件とも同じです。スキーマなしの条件でも、項目の説明と「JSON だけを返す」指示は入っています。

```text:prompt.txt
<stdin> は日本の見積書または請求書をテキストにしたものです。次の10項目を抽出し、JSON オブジェクト1つだけを返してください（説明文やコードフェンスは付けない）。ファイルの読み書きやコマンドの実行は不要です。

- doc_type: "見積書" か "請求書"
- document_number: 見積番号・請求番号。無ければ null
- issue_date: 発行日。YYYY-MM-DD（和暦は西暦に直す）
- due_date: 見積書は有効期限、請求書は支払期限。YYYY-MM-DD。具体的な日付として特定できなければ null
- issuer_name: 発行元の事業者名（書面の表記どおり）
- recipient_name: 宛先の事業者名（「御中」「様」は除く）
- registration_number: 適格請求書発行事業者の登録番号。"T" + 半角数字13桁（空白・ハイフンなし）。無ければ null
- subtotal_excl_tax: 税抜の合計金額（円、整数）
- tax_amount: 消費税額の合計（円、整数）
- total_incl_tax: 税込の合計金額（円、整数）

キーはこの10個だけにしてください。
```

## 1回分の実行

書類のテキストは標準入力で渡します。`codex exec` は、プロンプト引数と標準入力の両方があると、標準入力を `<stdin>` ブロックとしてプロンプトの後ろに付けます（`codex exec --help` の記載）。

```bash
codex exec --json --ephemeral --skip-git-repo-check --ignore-user-config \
  -s read-only -m gpt-6-luna --output-schema schema.json \
  -o d05.last.txt "$(cat prompt.txt)" < docs/d05.txt > d05.jsonl
```

- `--json`: 標準出力が1行1イベントの JSONL になり、最後の `turn.completed` 行にトークン数（`usage`）が入ります。
- `-o`: 最後のメッセージ（ここでは抽出結果の JSON）をファイルにも書き出します。
- `--ephemeral`: セッションのファイルをディスクに残しません。
- `-s read-only`: 読み取り専用のサンドボックスです（`codex exec` の既定も読み取り専用）。
- `--skip-git-repo-check`: git 管理外のディレクトリで実行するために付けています。
- `--ignore-user-config`: 手元の `config.toml` を読まずに実行します（認証は使われます）。

試しに1回流したときの `d05.jsonl` は、次の4行でした（`thread_id` は伏せています）。

```text:d05.jsonl
{"type":"thread.started","thread_id":"<伏せ字>"}
{"type":"turn.started"}
{"type":"item.completed","item":{"id":"item_0","type":"agent_message","text":"{\"doc_type\":\"請求書\",\"document_number\":\"2026-K-05\",\"issue_date\":\"2026-10-05\",\"due_date\":\"2026-10-25\",\"issuer_name\":\"架空建設株式会社\",\"recipient_name\":\"ダミー不動産株式会社\",\"registration_number\":\"T3210987654321\",\"subtotal_excl_tax\":120000,\"tax_amount\":12000,\"total_incl_tax\":132000}"}}
{"type":"turn.completed","usage":{"input_tokens":14847,"cached_input_tokens":0,"cache_write_input_tokens":0,"output_tokens":99,"reasoning_output_tokens":0}}
```

大字の「金壱拾参万弐阡円也」も 132000 になっています。この試走は下の60回には含めていません。

## 60回を回すスクリプト

条件×書類×反復を1回ずつ順番に実行し、生ログ（`.jsonl`）、最後のメッセージ（`.last.txt`）、標準エラー、実時間と終了コード（`.meta.json`）を `runs/` に保存します。実時間は `codex` プロセスの起動から終了までを Python の `time.monotonic()` で測った値です。

```python:run_all.py
# ダミー書類 × 条件 × 反復 で codex exec を回し、生ログを runs/ に保存する。
# 使い方: python3 run_all.py [反復回数=2]
import json
import os
import subprocess
import sys
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime, timezone, timedelta

HERE = os.path.dirname(os.path.abspath(__file__))
DOCS = sorted(f[:-4] for f in os.listdir(os.path.join(HERE, "docs")) if f.endswith(".txt"))
PROMPT = open(os.path.join(HERE, "prompt.txt"), encoding="utf-8").read()
SCHEMA = os.path.join(HERE, "schema.json")
RUNS = os.path.join(HERE, "runs")
WORK = os.path.join(HERE, "work")  # 空の作業ディレクトリ（書類は stdin で渡す）

CONDITIONS = {
    "luna_schema":   ["-m", "gpt-6-luna",  "--output-schema", SCHEMA],
    "luna_noschema": ["-m", "gpt-6-luna"],
    "sol_schema":    ["-m", "gpt-6.1-sol", "--output-schema", SCHEMA],
}
JST = timezone(timedelta(hours=9))


def run_one(cond, doc, rep):
    name = f"{cond}__{doc}__r{rep}"
    base = os.path.join(RUNS, name)
    if os.path.exists(base + ".meta.json"):
        return name, "skip"
    cmd = ["codex", "exec", "--json", "--ephemeral", "--skip-git-repo-check",
           "--ignore-user-config", "-s", "read-only",
           *CONDITIONS[cond], "-o", base + ".last.txt", PROMPT]
    with open(os.path.join(HERE, "docs", f"{doc}.txt"), "rb") as stdin, \
         open(base + ".jsonl", "wb") as out, open(base + ".stderr.txt", "wb") as err:
        started = datetime.now(JST).isoformat(timespec="seconds")
        t0 = time.monotonic()
        try:
            rc = subprocess.run(cmd, stdin=stdin, stdout=out, stderr=err,
                                cwd=WORK, timeout=600).returncode
        except subprocess.TimeoutExpired:
            rc = "timeout"
        wall_ms = int((time.monotonic() - t0) * 1000)
    meta = {"name": name, "condition": cond, "doc": doc, "rep": rep,
            "started_at": started, "wall_ms": wall_ms, "exit_code": rc,
            "args": [a if a != PROMPT else "<prompt.txt>" for a in cmd[2:]]}
    with open(base + ".meta.json", "w", encoding="utf-8") as fh:
        json.dump(meta, fh, ensure_ascii=False, indent=2)
    return name, rc


def main():
    reps = int(sys.argv[1]) if len(sys.argv) > 1 else 2
    os.makedirs(RUNS, exist_ok=True)
    os.makedirs(WORK, exist_ok=True)
    jobs = [(c, d, r) for r in range(1, reps + 1) for d in DOCS for c in CONDITIONS]
    with ThreadPoolExecutor(max_workers=1) as ex:
        for name, rc in ex.map(lambda j: run_one(*j), jobs):
            print(name, rc, flush=True)


if __name__ == "__main__":
    main()
```

## 集計スクリプト

`runs/` の生ログと `truth/` の正解を突き合わせます。最後のメッセージをそのまま `json.loads` できるか（だめならコードフェンスを外して再試行）、`jsonschema` でスキーマに合うか、項目ごとに正解と一致するかを数えます。あわせて、正解を見ずに出力だけで機械的にできる検査（税抜＋税＝税込の検算、登録番号が `T`＋13桁か、`null` になっていないか、日付の形式）が、実際の誤りをどれだけ拾えるかも数えます。

```python:score.py
# codex exec の生ログ（runs/*.jsonl）と正解 JSON（truth/*.json）を突き合わせて集計する。
# 使い方: python3 score.py [runs ディレクトリ] [truth ディレクトリ] [schema.json]
import csv
import json
import os
import re
import statistics
import sys
import unicodedata

import jsonschema

HERE = os.path.dirname(os.path.abspath(__file__))
RUNS = sys.argv[1] if len(sys.argv) > 1 else os.path.join(HERE, "runs")
TRUTH = sys.argv[2] if len(sys.argv) > 2 else os.path.join(HERE, "truth")
SCHEMA = json.load(open(sys.argv[3] if len(sys.argv) > 3 else os.path.join(HERE, "schema.json")))
FIELDS = list(SCHEMA["properties"])


def norm_text(v, field):
    """文字列項目の比較用正規化: NFKC・空白除去。宛先は敬称を除く。登録番号は英数字だけ。"""
    if v is None:
        return None
    if not isinstance(v, str):
        return ("<non-str>", repr(v))
    s = re.sub(r"\s+", "", unicodedata.normalize("NFKC", v))
    if field == "recipient_name":
        s = re.sub(r"(御中|様)$", "", s)
    if field == "registration_number":
        s = re.sub(r"[^0-9A-Za-z]", "", s).upper()
    return s


def same(field, got, want, alts):
    if field in ("subtotal_excl_tax", "tax_amount", "total_incl_tax"):
        # 金額は型も含めて完全一致（"135000" のような文字列は不一致）
        return type(got) is int and got == want if want is not None else got is None
    cands = [want] + alts.get(field, [])
    return any(norm_text(got, field) == norm_text(c, field) for c in cands)


def read_run(path):
    """JSONL から最後の agent_message・usage・コマンド実行数・エラーを取り出す。"""
    msg, usage, n_cmd, errors = None, None, 0, []
    with open(path, encoding="utf-8") as fh:
        for line in fh:
            try:
                ev = json.loads(line)
            except json.JSONDecodeError:
                continue
            t = ev.get("type")
            item = ev.get("item") or {}
            if t == "item.completed" and item.get("type") == "agent_message":
                msg = item.get("text")
            elif t == "item.completed" and item.get("type") == "command_execution":
                n_cmd += 1
            elif t == "turn.completed":
                usage = ev.get("usage")
            elif t in ("turn.failed", "error"):
                errors.append(t)
    return msg, usage, n_cmd, errors


def parse(msg):
    """そのまま json.loads できるか。だめならコードフェンスを外して再試行。"""
    if msg is None:
        return None, "no_message"
    try:
        return json.loads(msg), "raw"
    except json.JSONDecodeError:
        pass
    m = re.search(r"```(?:json)?\s*(\{.*\})\s*```", msg, re.S)
    if m:
        try:
            return json.loads(m.group(1)), "fenced"
        except json.JSONDecodeError:
            pass
    return None, "unparsable"


def med(xs):
    return statistics.median(xs) if xs else 0


def main():
    rows, mistakes = [], []
    for fn in sorted(os.listdir(RUNS)):
        if not fn.endswith(".meta.json"):
            continue
        meta = json.load(open(os.path.join(RUNS, fn), encoding="utf-8"))
        msg, usage, n_cmd, errors = read_run(os.path.join(RUNS, meta["name"] + ".jsonl"))
        truth = json.load(open(os.path.join(TRUTH, meta["doc"] + ".json"), encoding="utf-8"))
        alts = truth.get("_alt", {})
        obj, how = parse(msg)
        schema_ok = False
        if isinstance(obj, dict):
            try:
                jsonschema.validate(obj, SCHEMA)
                schema_ok = True
            except jsonschema.ValidationError:
                pass
        row = {"name": meta["name"], "condition": meta["condition"], "doc": meta["doc"],
               "rep": meta["rep"], "exit_code": meta["exit_code"], "wall_ms": meta["wall_ms"],
               "parse": how, "schema_ok": schema_ok, "n_commands": n_cmd,
               "turn_errors": len(errors)}
        u = usage or {}
        for k in ("input_tokens", "cached_input_tokens", "output_tokens", "reasoning_output_tokens"):
            row[k] = u.get(k, 0) or 0
        n_ok = 0
        for f in FIELDS:
            ok = isinstance(obj, dict) and same(f, obj.get(f), truth[f], alts)
            row[f"ok_{f}"] = ok
            n_ok += ok
            if not ok:
                mistakes.append({"condition": meta["condition"], "doc": meta["doc"],
                                 "rep": meta["rep"], "field": f,
                                 "want": truth[f],
                                 "got": obj.get(f) if isinstance(obj, dict) else f"<{how}>"})
        row["fields_ok"] = n_ok
        # 正解を見ずに出力だけでできる検査（実務での自動チェック相当）
        flags = []
        if isinstance(obj, dict):
            a, b, c = obj.get("subtotal_excl_tax"), obj.get("tax_amount"), obj.get("total_incl_tax")
            if not all(isinstance(x, int) for x in (a, b, c)) or a + b != c:
                flags.append("sum_mismatch")
            reg = obj.get("registration_number")
            if reg is None:
                flags.append("reg_null")
            elif not re.fullmatch(r"T\d{13}", reg):
                flags.append("reg_format")
            for k in ("issue_date", "due_date"):
                v = obj.get(k)
                if v is None:
                    flags.append(f"{k}_null")
                elif not re.fullmatch(r"\d{4}-\d{2}-\d{2}", v):
                    flags.append(f"{k}_format")
        row["flags"] = " ".join(flags)
        rows.append(row)

    conds = list(dict.fromkeys(r["condition"] for r in rows))
    print(f"runs: {len(rows)}  fields per run: {len(FIELDS)}")
    print()
    print("per condition:")
    hdr = ("condition", "runs", "exit0", "json_raw", "schema_ok", "field_acc%",
           "all10_ok", "wall_s_med", "in_tok_med", "cached_med", "out_tok_med", "cmds")
    print("  " + " ".join(f"{h:>13}" for h in hdr))
    summary = {"runs": len(rows), "per_condition": {}, "per_field": {}}
    for c in conds:
        g = [r for r in rows if r["condition"] == c]
        acc = sum(r["fields_ok"] for r in g) / (len(g) * len(FIELDS)) * 100
        s = {
            "runs": len(g),
            "exit0": sum(r["exit_code"] == 0 for r in g),
            "json_raw": sum(r["parse"] == "raw" for r in g),
            "schema_ok": sum(r["schema_ok"] for r in g),
            "field_acc_pct": round(acc, 1),
            "all_fields_ok": sum(r["fields_ok"] == len(FIELDS) for r in g),
            "wall_s_median": round(med([r["wall_ms"] for r in g]) / 1000, 1),
            "wall_s_min": round(min(r["wall_ms"] for r in g) / 1000, 1),
            "wall_s_max": round(max(r["wall_ms"] for r in g) / 1000, 1),
            "input_tokens_median": int(med([r["input_tokens"] for r in g])),
            "cached_input_tokens_median": int(med([r["cached_input_tokens"] for r in g])),
            "output_tokens_median": int(med([r["output_tokens"] for r in g])),
            "reasoning_output_tokens_median": int(med([r["reasoning_output_tokens"] for r in g])),
            "runs_with_commands": sum(r["n_commands"] > 0 for r in g),
        }
        summary["per_condition"][c] = s
        print("  " + " ".join(f"{v:>13}" for v in (
            c, s["runs"], s["exit0"], s["json_raw"], s["schema_ok"], s["field_acc_pct"],
            s["all_fields_ok"], s["wall_s_median"], s["input_tokens_median"],
            s["cached_input_tokens_median"], s["output_tokens_median"],
            s["runs_with_commands"])))
    print()
    print("per field (correct / runs):")
    print("  " + f"{'field':<22}" + " ".join(f"{c:>14}" for c in conds))
    for f in FIELDS:
        cells = []
        summary["per_field"][f] = {}
        for c in conds:
            g = [r for r in rows if r["condition"] == c]
            k = sum(r[f"ok_{f}"] for r in g)
            summary["per_field"][f][c] = k
            cells.append(f"{k}/{len(g)}")
        print("  " + f"{f:<22}" + " ".join(f"{x:>14}" for x in cells))
    print()
    print("output-only checks (runs flagged, all conditions):")
    check_field = {"sum_mismatch": "total_incl_tax", "reg_format": "registration_number",
                   "reg_null": "registration_number", "due_date_null": "due_date",
                   "due_date_format": "due_date", "issue_date_null": "issue_date",
                   "issue_date_format": "issue_date"}
    for fl, f in check_field.items():
        hit = [r for r in rows if fl in r["flags"].split()]
        wrong = sum(1 for r in hit if not r[f"ok_{f}"])
        print(f"  {fl:<18} flagged={len(hit):>3}  of which actually wrong={wrong}")
    print()
    print(f"mistakes ({len(mistakes)}):")
    for m in mistakes:
        print(f"  {m['condition']:<14} {m['doc']} r{m['rep']} {m['field']:<20} "
              f"want={json.dumps(m['want'], ensure_ascii=False)} "
              f"got={json.dumps(m['got'], ensure_ascii=False)}")

    with open(os.path.join(HERE, "scored_runs.csv"), "w", newline="", encoding="utf-8") as fh:
        w = csv.DictWriter(fh, fieldnames=list(rows[0].keys()))
        w.writeheader()
        w.writerows(rows)
    summary["mistakes"] = mistakes
    with open(os.path.join(HERE, "summary.json"), "w", encoding="utf-8") as fh:
        json.dump(summary, fh, ensure_ascii=False, indent=2)


if __name__ == "__main__":
    main()
```

`jsonschema` は標準ライブラリではないので、`python3 -m venv .venv && .venv/bin/pip install jsonschema` で入れてから `.venv/bin/python score.py` で実行します（今回は jsonschema 4.26.0）。

## 生出力

```text
runs: 60  fields per run: 10

per condition:
      condition          runs         exit0      json_raw     schema_ok    field_acc%      all10_ok    wall_s_med    in_tok_med    cached_med   out_tok_med          cmds
  luna_noschema            20            20            20            20          98.0            18           3.8         14319         11520            92             0
    luna_schema            20            20            20            20          98.5            18           3.7         14063             0            97             0
     sol_schema            20            20            20            20          99.0            18           5.9         15549             0            98             0

per field (correct / runs):
  field                  luna_noschema    luna_schema     sol_schema
  doc_type                       20/20          20/20          20/20
  document_number                20/20          20/20          20/20
  issue_date                     20/20          20/20          20/20
  due_date                       18/20          18/20          18/20
  issuer_name                    20/20          20/20          20/20
  recipient_name                 20/20          20/20          20/20
  registration_number            18/20          19/20          20/20
  subtotal_excl_tax              20/20          20/20          20/20
  tax_amount                     20/20          20/20          20/20
  total_incl_tax                 20/20          20/20          20/20

output-only checks (runs flagged, all conditions):
  sum_mismatch       flagged=  0  of which actually wrong=0
  reg_format         flagged=  1  of which actually wrong=1
  reg_null           flagged=  8  of which actually wrong=2
  due_date_null      flagged=  6  of which actually wrong=3
  due_date_format    flagged=  0  of which actually wrong=0
  issue_date_null    flagged=  0  of which actually wrong=0
  issue_date_format  flagged=  0  of which actually wrong=0

mistakes (9):
  luna_noschema  d08 r1 due_date             want="2026-10-31" got=null
  luna_noschema  d08 r1 registration_number  want="T7777700008888" got=null
  luna_noschema  d08 r2 due_date             want="2026-10-31" got=null
  luna_noschema  d08 r2 registration_number  want="T7777700008888" got="T777700008888"
  luna_schema    d07 r1 due_date             want=null got="2026-10-03"
  luna_schema    d08 r2 due_date             want="2026-10-31" got=null
  luna_schema    d08 r2 registration_number  want="T7777700008888" got=null
  sol_schema     d07 r1 due_date             want=null got="2026-10-03"
  sol_schema     d07 r2 due_date             want=null got="2026-10-03"
```

列の意味: `exit0` は終了コード0の回数、`json_raw` は最後のメッセージをそのまま `json.loads` できた回数、`schema_ok` はスキーマ検証に通った回数、`all10_ok` は10項目すべて正解だった回数、`wall_s_med` は実時間の中央値（秒）、`in_tok_med`・`cached_med`・`out_tok_med` は `turn.completed` の `input_tokens`・`cached_input_tokens`・`output_tokens` の中央値、`cmds` はコマンド実行（`command_execution`）が出た回数です。

## 結果

| 条件 | 項目正答率 | 10項目すべて正解 | JSON として読めた | 実時間 中央値（範囲） | 入力トークン 中央値 | 出力トークン 中央値 |
|---|---|---|---|---|---|---|
| GPT-6 Luna・スキーマあり | 98.5%（197/200） | 18/20回 | 20/20回 | 3.7秒（3.3〜5.3） | 14,063 | 97 |
| GPT-6 Luna・スキーマなし | 98.0%（196/200） | 18/20回 | 20/20回 | 3.8秒（3.2〜5.3） | 14,319 | 92 |
| GPT-6.1 Sol・スキーマあり | 99.0%（198/200） | 18/20回 | 20/20回 | 5.9秒（5.1〜8.7） | 15,549 | 98 |

60回すべてが終了コード0で、最後のメッセージはそのまま JSON として読め、スキーマ検証にも通りました。コードフェンス付きや説明文付きの出力は、スキーマなしの条件でも1回もありませんでした。また、コマンド実行（`command_execution`）は60回とも0回でした。

![条件別の誤り項目数の横棒グラフ。GPT-6 Luna スキーマ指定ありは期限2・登録番号1の計3項目（正答率98.5%）、GPT-6 Luna スキーマ指定なしは期限2・登録番号2の計4項目（98.0%）、GPT-6.1 Sol スキーマ指定ありは期限2の計2項目（99.0%）](/images/codex-invoice-extract/fig1-accuracy-ba4fb19b84.png)

誤りは3条件合わせて9項目で、すべて d07 と d08 の「期限」「登録番号」に集中しました。金額（税抜合計・消費税額・税込合計）、日付の和暦変換、大字、内税からの逆算、軽減税率の合算、小数単価と端数切り捨ては、60回すべて正解です。

![誤りが出た書類・項目と条件別の誤り回数の表。d07の期限（発行日より30日間）はLunaスキーマあり1/2回・Lunaスキーマなし0/2回・Solスキーマあり2/2回、d08の期限（10月末日）は1/2回・2/2回・0/2回、d08の登録番号（全角・ハイフン区切り）は1/2回・2/2回・0/2回。他の書類・項目は全条件で誤りなし](/images/codex-invoice-extract/fig2-errors-9f1af34d91.png)

### 誤りの中身

**d07 の期限（正解 `null`）**: 誤った3回はいずれも `"due_date":"2026-10-03"` を返しました。発行日 2026-09-03 に30日を足した日付です。GPT-6.1 Sol は2回とも、GPT-6 Luna（スキーマあり）は2回中1回がこの値で、残りの3回は `null` でした。

**d08 の期限（正解 `2026-10-31`）**: 誤った3回はいずれも `null` でした。GPT-6.1 Sol は2回とも `2026-10-31` を返しています。

この2つは、指示文の「具体的な日付として特定できなければ null」を、モデルと私とで違う向きに解釈した結果とも読めます。d07 は私が「特定できない」とした相対表現を日付に直し、d08 は私が「特定できる」とした年なしの表現を `null` にしています。どちらが妥当かは業務のルール次第なので、実務では「相対表現は `null` にして原文を別項目に入れる」のように、指示文側でルールを1つに決めておく必要があります。

**d08 の登録番号（正解 `T7777700008888`）**: 書面には `Ｔ７－７７７７－００００－８８８８`（全角・ハイフン区切り）と書いてあります。誤った3回の出力は次のとおりです（該当行だけ抜粋）。

```text
luna_noschema  d08 r1  "registration_number": null
luna_noschema  d08 r2  "registration_number":"T777700008888"
luna_schema    d08 r2  "registration_number":null
```

2回は書面にある番号を `null` にし、1回は `7` が1つ少ない12桁を返しています。理由はログからは分かりません。同じ書類・同じ条件でも、2回のうち1回は正しく `T7777700008888` を返しており（luna_schema の r1）、同じ入力でも結果が揺れる点は押さえておく必要があります。

### 出力だけでできる検査は、9件中6件を拾った

正解データは本番運用では手に入らないので、出力だけで機械的に判定できる検査を足したときに誤りをどれだけ拾えるかも数えました（生出力の `output-only checks`）。

| 検査 | 引っかかった回数 | うち実際に誤り |
|---|---|---|
| 税抜＋消費税≠税込 | 0 | 0 |
| 登録番号が `T`＋13桁でない | 1 | 1 |
| 登録番号が `null` | 8 | 2 |
| 期限が `null` | 6 | 3 |

12桁の登録番号は形式チェックで1件、`null` になった登録番号2件と期限3件は「`null` なら人が見る」ルールで拾えます。9件中6件です。ただしこのルールでは、正しく `null` だった回（d10 の登録番号6回、d07 の期限3回）も人の確認に回ります。残りの3件（d07 で `2026-10-03` を返した回）は、形式も検算も正しいため、出力だけでは検出できませんでした。

金額の検算（税抜＋税＝税込）は60回とも矛盾がありませんでした。ただし、今回は金額の誤りそのものが0件だったため、この検算が実際に誤りを拾えるかはこの実験では確かめられていません。

### 実行時間とトークン

![1回あたりの実行時間のドットプロット。GPT-6 Luna スキーマ指定ありは中央値3.7秒、スキーマ指定なしは3.8秒、GPT-6.1 Sol スキーマ指定ありは5.9秒。各20回](/images/codex-invoice-extract/fig3-time-8a9e2c8c28.png)

1件あたりの実時間は、GPT-6 Luna が中央値3.7〜3.8秒、GPT-6.1 Sol が5.9秒でした。CLI の起動から終了までを含んだ時間です。スキーマの有無による差は、中央値で0.1秒でした。

入力トークンは1回あたり13,858〜15,618でした。書類のテキストは長いものでも628文字（空白・改行込み）で、書類だけでは説明できない量ですが、入力の内訳は `--json` のイベントには出てこないため、ここでは分けていません。出力トークンの中央値は 92〜98 でした。

`cached_input_tokens` が0より大きかった回は、スキーマなしの条件で20回中20回（11,520〜13,824）、GPT-6 Luna のスキーマありで20回中5回、GPT-6.1 Sol で20回中1回でした。この差がなぜ出たのかは、ログからは分かりません。

## 実務で使うときの注意

- **最終確認は人が行う**。今回、金額は60回とも正解でしたが、架空の書類10件での結果です。誤りは「書面にある登録番号を `null` にする」「桁を1つ落とす」といった形で出ました。
- **検算と形式チェックを先に機械で行い、引っかかったものを人に回す**。今回の例では、`税抜＋税＝税込`、登録番号の `^T\d{13}$`、必須にしたい項目の `null` を見るだけで、9件中6件の誤りが人の確認に回りました。
- **期限のような「解釈」が入る項目は、ルールを指示文に書き切る**。「30日間」「末日」の扱いを決めないと、モデルごと・回ごとに答えが分かれました。
- **機密書類の扱い**。`codex exec` は既定で読み取り専用のサンドボックスで動き、`--ephemeral` を付けるとセッションのファイルをディスクに残しません。CI では、リポジトリのコードを実行するジョブに `OPENAI_API_KEY` や `CODEX_API_KEY` をジョブ全体の環境変数として置かないよう公式が注意しています（[Non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)、確認日 2026-10-23）。ChatGPT では、入力が学習に使われるかどうかがプランと設定で異なります。プラン別の初期設定と学習オフの手順は[ChatGPTの情報漏洩対策：学習オフの設定手順とプラン別の違い](https://ai.sanvia.co.jp/articles/chatgpt-data-leak-settings)にまとめています。Codex で処理した内容の扱いについては、OpenAI のヘルプセンターのページが執筆時に筆者の環境から HTTP 403 を返して開けなかったため、この記事では確認できていません（未確認）。社内で使う前に、自社のプランでの扱いを公式の案内で確認してください。

## 限界

- 書類は筆者が作った架空のダミー10件で、テキストです。PDF・スキャン画像・手書き・複数ページの書類は試していません。
- 各条件20回（10件×2回）だけです。98.0% と 99.0% の差は誤り項目数で2つ分で、条件間の優劣を言える量ではありません。
- 正解データの決め方（特に期限）に筆者の判断が入っています。d07・d08 の期限の誤り6件は、正解の決め方を変えれば数が変わります。
- 同じマシン・同じ日の5分間の実行で、時間やキャッシュは混み具合などで変わりえます。推論レベルは指定していません。
- トークン数は `turn.completed` の `usage` の値をそのまま使っています。公式ドキュメントには各フィールドの例はありますが、内訳の定義の記載は見当たらなかったため、意味づけはしていません。

## よくある質問

### Q. `--output-schema` を付けないと JSON が壊れますか

今回の60回では、付けなかった20回も含めて全回がそのまま JSON として読め、スキーマ検証にも通りました。ただし指示文で「JSON オブジェクト1つだけを返す」「説明文やコードフェンスは付けない」と明示しています。公式ドキュメントでは、後続の処理で安定した項目が必要な場合に `--output-schema` を使うよう案内されています（[Non-interactive mode](https://learn.chatgpt.com/docs/non-interactive-mode)、確認日 2026-10-23）。今回の結果は、付けなくても壊れなかった、というところまでです。

### Q. 1件あたりどれくらい時間がかかりますか

今回の環境では、GPT-6 Luna で中央値3.7〜3.8秒、GPT-6.1 Sol で5.9秒でした（CLI の起動から終了まで、1件ずつ順番に実行）。60回の実時間の合計は276.0秒（約4分36秒）でした。

### Q. Claude Code でも同じことができますか

公式ドキュメントには、`claude -p` に `--output-format json` と `--json-schema` を付けると、スキーマに合った出力が `structured_output` フィールドに入ると書かれています。有効な構造化出力がないまま終わった場合はこのフィールドが無いので、使う前に存在を確認するよう案内されています（[Run Claude Code programmatically](https://code.claude.com/docs/en/headless)、確認日 2026-10-23）。この記事では Claude Code での実測はしていません。

## 参考資料

公式ドキュメントの各ページには公表日の表示がないため「公表日: 不明」としています。

- Non-interactive mode（`codex exec`、`--output-schema`、`--json`、`--ephemeral`）— OpenAI — https://learn.chatgpt.com/docs/non-interactive-mode — 公表日: 不明 — 確認日: 2026-10-23
- Models（GPT-6.1 Sol / GPT-6 Luna の推奨用途）— OpenAI — https://learn.chatgpt.com/docs/models — 公表日: 不明 — 確認日: 2026-10-23
- Run Claude Code programmatically（`--json-schema` と `structured_output`）— Anthropic — https://code.claude.com/docs/en/headless — 公表日: 不明 — 確認日: 2026-10-23
- ChatGPTの情報漏洩対策：学習オフの設定手順とプラン別の違い — AIみちしるべ — https://ai.sanvia.co.jp/articles/chatgpt-data-leak-settings — 公表日: 2026-10-15（更新 2026-10-20） — 確認日: 2026-10-23
- Codex CLIの使い方｜インストールからexec・料金まで実機検証 — AIみちしるべ — https://ai.sanvia.co.jp/articles/codex-cli-usage — 公表日: 2026-10-21 — 確認日: 2026-10-23

---

筆者: Kosuke。AIみちしるべ（[https://ai.sanvia.co.jp](https://ai.sanvia.co.jp)）を運営しています。Codex CLI のインストールや `codex exec` の基本、料金の目安は[Codex CLIの使い方の実機検証記事](https://ai.sanvia.co.jp/articles/codex-cli-usage)にまとめています（[著者ページ](https://ai.sanvia.co.jp/authors/kosuke)）。この記事のダミー書類・正解データ・生ログ・集計スクリプトの生出力は手元に保存してあり、本文のスクリプトはそれと同一です。
