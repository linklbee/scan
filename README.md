# Linklbee scan

**Find out how much of your LLM bill is the model re-writing a document it already wrote.**

One command, your own log file. No account, no signup, nothing uploaded.

```bash
python3 linklbee_scan.py --logs calls.jsonl --out report.md --lang en
```

You get one report that answers one question: **of the output tokens you paid for, what share was the next version of a document the model had already produced** — and of that share, how much would actually be cheaper to handle differently.

---

## Zero network calls. Check it yourself before you run it.

```bash
grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py
# 0
```

- **Single file.** Standard library only. Python 3.9+. Nothing to install.
- Your answer text is read **in memory only**. The report and the JSON contain counts and shares — **no text, no keys**.
- There is no telemetry, because there is no code that could send any.

---

## What the report contains

```
0. What was read
1. Output share of the bill
2. Two-axis table (overlap × answer length)
3. Qualified share — the conclusion
4. Savings estimate (band)
5. Zero-call candidates
6. What could not be judged
```

Section 6 is not filler. Calls with no `output_tokens` and no body, models missing from the price table, logs too small to fit a window — the scanner lists them instead of quietly folding them into an average.

**If your traffic does not qualify, the report says so.** That is a normal outcome and it is the honest one.

---

## Input

One JSON object per line. The minimum:

```json
{"ts": "2026-10-17T09:12:33Z", "output": "..."}
```

More accurate with `model`, `input_tokens`, `output_tokens`, `group_key`.

| Flag | What it does |
|---|---|
| `--logs FILE` | the JSONL call log |
| `--out FILE` | write the report here |
| `--lang en\|ko` | report language |
| `--json FILE` | also write the stats as JSON |
| `--share FILE` | statistics-only summary — no text, no names, no keys. Safe to paste in a team channel |
| `--hashify FILE` | convert a log to token-hash form first, if you need to hand the log itself to someone |
| `--pricing FILE` | your own price table |

### Speed (measured, 16 GB MacBook)

| Log size | Time |
|---|---|
| 100,000 lines | 83 s |
| 500,000 lines | 7 min |

(The 500k run was done once.)

---

## Where the savings numbers come from

The scanner measures **your** share from **your** logs. The savings band it applies to that share comes from the runs below — real paid calls against public revision datasets, each item called twice (once through Linklbee, once as a full rewrite), with quality scored against the version that was actually published next.

| Dataset (public) | Calls | Output tokens | Cost | Quality: Linklbee | Quality: full rewrite |
|---|---|---|---|---|---|
| Korea MFDS drug-label revisions | 45 | −78.5% | −61.7% | 0.9719 | 0.9346 |
| Korean statute amendments | 317 | −57.1% | −38.2% | 0.9777 | 0.9811 |
| US eCFR amendments | 79 | −74.3% | −56.9% | 0.9875 | 0.9439 |

Model `claude-sonnet-5-5`, 2026-10-17 to 18. **Sample sizes are where each run's budget ended, not a design choice.**

**The cost figures include the cost of re-calling as a full rewrite when an edit fails to apply.** We did not take that out.

These come from public data. They are not a promise about your traffic — which is the entire reason this scanner exists.

---

## There is a cheaper way. We don't ship it.

Sending the previous answer back unchanged cuts the cost by more than half. We measured it, and it really does.

The problem is **four in ten**. The date changed, the article number changed, and last time's answer goes out unchanged. It reads almost identically, so automated checks don't catch it either. The end user gets that answer and nobody ever knows.

That is why we left it out. The savings above are what is left after taking it away.

---

## Who made this, and why it is free

We are [NOTEGG](https://linklbee.com). We sell **Linklbee** — middleware you run on your own servers that keeps the previous answer per document, asks the model for only the edit, and applies it locally. Requests go from your server straight to your vendor; they never pass through ours.

This scanner is how you find out whether that would do anything for you, **before** talking to us. It is MIT, it runs entirely on your machine, and it will tell you when the answer is no.

Questions, or want to go through a report together: **bee@linklbee.com**. Please don't send us your logs.

---

## License

MIT © 2026 NOTEGG — see [LICENSE](LICENSE).

The Linklbee middleware is a separate commercial product and is **not** covered by this licence.
