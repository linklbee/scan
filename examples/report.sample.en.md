# Linklbee diagnostic report — output tokens

scanner 1.0.2 · calls.sample.jsonl

## 0. What was read

- 248 calls (248 lines, 0 bad JSON, 0 rejected for missing ts) · period 2026-09-01 – 2026-10-08
- models: claude-sonnet-5-5 168, gpt-5.6-luna 80
- body: text 248 · token hashes 0 · none 0
- This script has zero lines of network code. Check: `grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py` -> 0. Answer text is read in memory only; this report and the JSON hold no text and no keys (counts and shares only).

## 1. Output share of the bill

- output $0.88 / total $1.20 = **73.1%** (248 priced calls, price table as of 2026-10-18)

## 2. Two-axis table (overlap x answer length)

138 pairs (110 keys; customer-key calls 248 · auto-found calls 0, auto pairs 0 · skipped for equal ts 0). Each pair = the latest earlier answer of the same key -> this answer. Overlap here is measured knowing the answer.

| overlap \ length | <200 | 200-500 | 500-2000 | 2000+ |
|---|---|---|---|---|
| >=0.95 | — | — | **100 · 63.4%** | — |
| 0.78-0.95 | — | — | **20 · 13.3%** | — |
| 0.60-0.78 | — | — | — | — |
| <0.60 | — | 18 · 5.7% | — | — |

cell = pairs · share of all output tokens. Bold = switch on (overlap >= 0.78 and answer >= 200 tokens). Pair overlap token-weighted 0.933, median 0.975.

## 3. Qualified share — the conclusion

- **Calling at every version leaves a large band where the call itself loses (23.4% of output at overlap >= 0.99), but calling once per few versions brings it into the qualifying range.** span 2 for every document (one call per that many versions): >= 0.99 4.7% · under 0.78 where diff loses 5.7% · the money band 0.85-0.99 89.6%. The span was fitted on this log (table below).

Span (how many versions back the draft is) — share of paired output at >= 0.99 / < 0.78 / 0.85-0.99:

This table is measured after folding answers equal to the previous answer (repeats: 0 pairs, 0.0% of span-1 paired output) and leaving out answers that went back to an earlier version (reverts: 0 pairs, 0.0%). Neither needs a call at any span, so neither counts as an effect of the span.

| span | pairs | >=0.99 | <0.78 | 0.85-0.99 |
|---|---|---|---|---|
| 1 | 138 | 23.4% | 6.9% | 69.7% |
| 2 | 108 | 4.7% | 5.7% | 89.6% |
| 3 | 78 | 0.0% | 3.9% | 96.1% |
| 4 | 48 | 0.0% | 0.0% | 97.8% |
| 6 | 0 | — | — | — |
| 8 | 0 | — | — | — |

Span picked per length tier (answer-token edges [532.0, 558.0], this log's tertiles): 1*, 1*, 1*. A tier with under 100 pairs keeps span 1. A span of 2 or more is picked only where it beats span 1 on the same answers (also at the 2.5th percentile of a document resample).

- **Output tokens in switch-on cells = 76.7% of all output** (120 pairs). Of paired output: 93.1%.
- Engine replay (switch decides before the answer, from the key's earlier pairs only): 62.6% of all output (96 pairs). A key's first pair has no history and stays off.
- Fact input (axis 3) **not checked**: switch-on pairs carry no request body (prompt). The 76.7% above comes from two axes only — if a request does not say what changed, the real saving can be lower.

## 4. Savings estimate (band)

**Not a saving measured on this log.** Each switch-on cell gets the range we measured (below) as a band; outside the switch it is 0.

- output tokens: 36.9% – 68.9% of all output (bands applied)
- with the engine replay switch: 29.8% – 56.0%
- dollars (cost-saving bands, measured without prompt cache): $0.22 – $0.65 = 18.4% – 54.5% of the bill · prices as of 2026-10-18 · switch-on pairs without a price 0

Ranges we measured (three public document sets (product copy, drug label revisions, statute revisions), real calls, failed-edit re-calls included (not for product copy: no call records), 2026-10):
- overlap >=0.95 · answer >= 200 tokens: output 55% – 95%, cost 30% – 75%
- overlap 0.78-0.95 · answer >= 200 tokens: output 15% – 65%, cost -5% – 50%

## 5. Zero-call candidates

- No prompt (or prompt_hash), so the gate could not run.
- Reference (counted with the answer known, an upper bound): pairs identical to the previous answer 0 · including digits-only differences 16

## 6. What could not be judged

- 248 calls without output_tokens — counted from the body with o200k (approximate for non-OpenAI models)
- Saving ranges were measured on other materials (two public document sets). This customer's own value comes from a holdout pilot

took 1.4s
