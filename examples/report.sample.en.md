# Linklbee diagnostic report — output tokens

scanner 1.1.1 · calls.sample.jsonl

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

- **Caution — on this material the LLM call itself may be the loss.** 23.4% of paired output is at overlap >= 0.99 (the draft already is the next answer). On such material calling, diff or full, scored below the draft as is (admin rules, 277 real calls: draft 0.9635 > diff 0.9611 > full 0.9545). What should shrink here is the call itself, and that is not Linklbee's product -- Linklbee does not turn on removing the call (zero-call) by default. The figures below are 'if you call'. The line (20.0% of output at >= 0.99) was set on three other materials (two that passed, 8-16%; one that lost, 27-28%), not on this log.

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

With 248 calls (under 600) the span is not fitted to this log (span 1 for every tier; the table above is for reference). A span picked on a small log rests on chance.

- **Output tokens in switch-on cells = 76.7% of all output** (120 pairs). Of paired output: 93.1%.
- Engine replay (switch decides before the answer, from the key's earlier pairs only): 62.6% of all output (96 pairs). A key's first pair has no history and stays off.
- Fact input (axis 3) **not checked**: switch-on pairs carry no request body (prompt). The 76.7% above comes from two axes only — if a request does not say what changed, the real saving can be lower.

## 4. Savings estimate (band)

This section looks at the document-diff cells only. On a log whose documents do not resemble each other 0 is the normal result; see section 7.

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

took 1.2s

## 7. Waste checklist — items read without the answer body

- 7-1 billed failed calls — **[cannot judge: input missing]** needs: status
- 7-2 the same request sent again shortly after — **[cannot judge: input missing]** needs: request (or prompt / prompt_hash)
- 7-3 answers cut at the token limit — **[cannot judge: input missing]** needs: finish_reason (or max_tokens + output_tokens)
- 7-4 unstable serialization of the request prefix (tool definitions, system instructions) — **[cannot judge: input missing]** needs: request (tools / system)
- 7-5 prompts under the cache minimum — **found**
  - Fix: the engine counts each request's tokens short of the minimum but does not fix it yet; for now fixing it is in-house work.
  - claude-sonnet-5-5 (anthropic): under 512 tokens 0 / 168 calls (0.0%), 0 of them with no cache read.
  - gpt-5.6-luna (openai): under 1,024 tokens 80 / 80 calls (100.0%), 80 of them with no cache read.
  - by threshold (248 calls judged): 64 -> 0.0% under · 512 -> 32.3% under · 1,024 -> 85.9% under · 2,048 -> 100.0% under · 4,096 -> 100.0% under
  - Engine: `/admin/usage` `waste.cache_min` reports the tenant ledger's share under the same constants, with a correction for those calls (merge the prefix past the threshold, or drop the cache marker).
  - Constants as of 2026-10-20 (vendor documents). Gemini prices change on 2026-12-01.
- 7-6 billing check — billed output tokens vs the answer body — **[cannot judge: input missing]** needs: answer body (output) + the vendor's output_tokens
- 7-7 prompt cache hit rate — **[cannot judge: input missing]** needs: input_tokens + the vendor's cache-read field (cached_tokens / cache_read_input_tokens / cachedContentTokenCount)
  - not judged: no cache-read field 248 — not counted as 0
- 7-8 prefix shared between calls — **[cannot judge: input missing]** needs: request body (messages / system / tools) or prompt, + input_tokens + ts

This section covers the log given. It is not a guarantee.

## 8. Axes that came out 0 — why, and where to look next

One line per axis: (a) the reason, counted on this log · (b) what would change it, or which axis to look at instead. Every figure is counted on this log or is a named constant in the code (named in brackets).

- Axis 3, fact input — cannot be measured on this log (no field prompt). (b) With the request body (prompt, or --hashify's prompt_token_hashes) it is measured.
- Section 5, the zero-call gate — cannot be measured on this log (no field prompt). (b) With prompt or prompt_hash it runs. Upper bound counted with the answer known (reference): pairs identical to the previous answer 0 · with digits-only differences 16.
- 7-1 billed failed calls — cannot be measured on this log (no field status).
- 7-2 the same request sent again shortly after — cannot be measured on this log (no field request (or prompt / prompt_hash)).
- 7-3 answers cut at the token limit — cannot be measured on this log (no field finish_reason (or max_tokens + output_tokens)).
- 7-4 unstable request prefix serialization — cannot be measured on this log (no field request (tools / system)).
- 7-6 billing check — cannot be measured on this log (no field answer body (output) + the vendor's output_tokens).
- 7-7 prompt cache hit rate — cannot be measured on this log (no field input_tokens + the vendor's cache-read field (cached_tokens / cache_read_input_tokens / cachedContentTokenCount)).
