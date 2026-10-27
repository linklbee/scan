# Linklbee LLM API Cost Diagnostic Report

scanner 1.4.7 · 1 file · 한국어: --lang ko

**Cache hit rate 0.0%** — The share of input tokens read from the cache (322 calls, the vendor's usage) · 0.0% on calls over the minimum. Details in 7-7.

## Summary

- Additional costs identified: **$0.00** — the sum of section 2 'Additional costs identified', 3 items (1 items without enough data for an amount are left out of the sum; they are listed in section 2)
- Additional savings opportunities: **$0.00** — the sum of section 2 'Additional savings opportunities', 3 items, using the lower bound of each range (2 items without enough data for an amount are left out of the sum; they are listed in section 2)
- Of that, setting improvements: $0.00 · ongoing optimization: $0.00. Generating only the changed parts (section 4) and reusing earlier answers (section 6) are ongoing optimization items too: the effect depends on the similarity to the earlier answer and on answer length, the thresholds have to be adjusted when the request pattern changes, and applying it to low-similarity calls can raise cost.
- The synthetic example has a total cost of $1.06. Identified additional costs are $0.004, so the value is displayed as $0.00 at two decimal places. Your report uses amounts from your own log.

The amounts shown are calculated by the diagnostic's rules. Actual savings are confirmed by measurement after integration.

## 1. What was read

- **🧪 [synthetic] This is the example log.** A call log built for the example report: 322 of 322 calls are synthetic. It does not represent any company's usage. To scan your own log: `python3 linklbee_scan.py --logs <file>`.
- 322 calls (322 lines, 0 not JSON, 0 rejected for missing ts) · 2026-09-01 to 2026-10-09
- models: claude-sonnet-5-5 168, gpt-5.6-luna 154
- body: text 308 · token hashes 0 · none 14
- input grade A — with answer text. 7 of 7 checklist items can be checked. Items 7-1 (billed failed calls) and 7-6 (billing check) are fixed on the service side, so they are not in the count or the sums; section 2 gives their result and the direction of the fix.
- Billing: per token on the API — dollars are API usage x the price list. On a flat rate or a plan, add `--flat all`.
- The scanner has no network code. To check: `grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py` -> 0. Answer text stays in memory. This report and the JSON hold counts and shares only — no text, no document ids.
- 14 calls have no body, so they are not in the answer pairs or similarity

## 2. Checklist — what is leaking, what additional savings opportunities

### Additional costs identified — money going out

- [code 7-2] the same request sent again shortly after — **improvement found**
  - Fix: when an identical request (same full body) comes back within a set time, the engine can return the stored answer without calling the vendor. Streams and irreversible actions are excluded. A regenerate would also get the stored answer, so the engine proposes this with the amount first, and you decide whether to turn it on.
  - Setting improvement: applied by changing a setting or how calls are built. You can apply it yourself with the method below.
  - How: find where the same request is sent again shortly after (retries, double clicks) and stop it from going out again before the first one has finished.
  - Calls paid for and not used, two kinds (told apart in the same log): **discarded output** $0.00 (14 failed calls still billed + 0 OK answers the same request replaced — a quality check, a broken format or a user's regenerate cannot be told apart here) · **retries** $0.00 (14 repeats after a failed or timed-out call — the service's retry setting). Repeats after a cut answer are counted in 7-3.
  - Within 60 s, 14 calls repeated the request just sent (within the same document when the log has document ids; out of 322 calls whose request is known). The earlier call had: failed 14 · been cut 0 · succeeded 0 · unknown 0. Median gap 30.5 s. input 2,379 · output 306 tokens, $0.00 (14 priced calls, API usage x price list). A repeat after a successful answer may be a user regenerating on purpose.
  - Split by the earlier call: after a successful answer none — the engine's stored answer can fix this · after a failure $0.00 — fixed by the service's retry settings · after a cut none — fixed by 7-3's larger cap.
  - The retry pattern and its cost after failures are analyzed and the settings to improve are shown. The engine does not change this item itself; it takes an adjustment of the service's retry policy (interval, maximum retries).
  - Engine: the Linklbee server records each response's status and request signature. The engine's usage view shows retry chains and gaps after failures, with a suggested retry policy (backoff interval, maximum retries, switching route after repeated failures).
- [code 7-3] answers cut off at the length limit — **improvement found**
  - Fix: if the same request is often cut and then resent with a higher cap, the engine can send it with that cap from the start, so you stop paying for the cut first attempt. Answers may get longer, so the engine proposes this with the amount first, and you decide whether to turn it on.
  - Setting improvement: applied by changing a setting or how calls are built. You can apply it yourself with the method below.
  - How: send the calls that are resent with a bigger length limit after a cut with that limit from the start.
  - Of 322 calls that can be checked, 1 were cut off (0.3%). Their output was 300 tokens, 0.3% of all output, and they cost $0.00. max_tokens of the cut calls: 300×1.
  - Engine: the Linklbee server records each answer's finish reason and the request's max_tokens. The engine's usage view shows the cut share per limit, with a suggested limit.
- [code 7-4] request prefix serialized inconsistently (tool definitions, system instructions) — **improvement found**
  - Fix: the engine sends tool definitions and system blocks in one key order. It can turn this on when usage records show a gain, and turns it off when they show a loss (judged on Anthropic and OpenAI GPT-5.6+ records). The current prefix normalization adjusts key order; changes to values such as timestamps and ids are not included.
  - Setting improvement: applied by changing a setting or how calls are built. You can apply it yourself with the method below.
  - How: send the tool definitions and system instructions in the same order and the same text every time, and move changing values such as times and IDs to the end of the call.
  - 322 calls have a prefix. 3 kinds of block were compared by their as-sent hash and key-sorted hash.
  - **Block `tools`: 2 serializations with the same content in a different key order, so the vendor caches it as 2 separate entries.** Of 154 calls with this block, 29 were not in the most common form. The forms first differ at character 4 (the block is 409 characters, 87 o200k tokens).
  - Source (vendor documents): Anthropic's prompt caching guide says "some languages (for example, Swift, Go) randomize key order during JSON conversion, breaking caches". OpenAI's guide, in its table of settings that affect the cached prefix, says of tools: "Changes tool names, descriptions, schemas, ordering, or tool-specific instructions."
  - Engine: the Linklbee engine sends tool definitions and system blocks with sorted keys. List order and strings stay the same, so the model reads the same content (a structured-output schema keeps its field order; only its other keys are sorted). It can turn this on when usage records show a gain, and turns it off when they show a loss. Set `"prefix_normalize": false` in the engine settings file `waste.json` to keep it off.
- [code 7-7] prompt cache hit rate — **nothing found**
  - Cache-read tokens / prompt tokens = **0.0%** (322 calls). For calls above the cache minimum only (the 120 that pass 7-5): **0.0%**. No threshold can be set from this log, so the rate alone is not judged (next line).
  - The threshold comes from the cache-usable share computed on this log. No input of the calls above the minimum is a prefix shared with an earlier call of the same model within 5 minutes (0 %), so no threshold can be set. Cache savings would need a different request pattern.
  - claude-sonnet-5-5 (anthropic): 0.0%; above the minimum 0.0% (120 calls); 168 calls with no cache read.
  - gpt-5.6-luna (openai): 0.0%; above the minimum — (0 calls); 154 calls with no cache read.
  - Why calls above the minimum still miss: caching was not turned on (the Anthropic API caches only with `cache_control` in the request; Bedrock tries without it, best effort, with no hit guaranteed), the prefix is not byte-identical (7-4), the cache expired, or the vendor sent the request to another server (OpenAI's guide says traffic over about 15 a minute on one key can overflow on models before GPT-5.6; from GPT-5.6 on OpenAI routes the cache itself).
  - Engine: the engine's usage view reports the same rate from your usage records. Below the line it suggests `cache_control` markers (Anthropic, Bedrock), prefix normalization (`"prefix_normalize": true`), putting fixed parts first, and OpenAI `prompt_cache_key`.

### Additional savings opportunities — cheaper ways, even with nothing wrong

- [code 7-5] prompts shorter than the cache minimum — **improvement found**
  - Fix: the engine counts how many tokens each request is short of the minimum. It can pad a short fixed prefix up to the minimum, but only with text you provide. It can also send short calls that share a prefix as one call (OpenAI and Anthropic, single calls without tools). The answers can change, so the engine never turns this on itself; you decide.
  - Setting improvement: applied by changing a setting or how calls are built. You can apply it yourself with the method below.
  - How: for short calls that share a start, lengthen the fixed instructions past the cache minimum or send several of them together as one request.
  - claude-sonnet-5-5 (anthropic): 48 of 168 calls under 512 tokens (28.6%), 48 of them with no cache read.
  - gpt-5.6-luna (openai): 154 of 154 calls under 1,024 tokens (100.0%), 154 of them with no cache read.
  - by threshold (322 calls checked): 64 -> 0.0% under · 512 -> 62.7% under · 1,024 -> 100.0% under · 2,048 -> 100.0% under · 4,096 -> 100.0% under
  - Anthropic runs a prompt under the minimum without caching and without an error, so nothing in the log shows it.
  - Engine: the engine's usage view reports the same share from your usage records, using the same constants, with a suggested fix for those calls (grow the prefix past the threshold, or drop the cache marker).
  - Thresholds as of 2026-10-20 (vendor documents). Gemini prices change on 2026-12-01.
- [code 7-8] prefix shared between calls — **improvement found**
  - Ongoing optimization: whether and at what threshold it applies follows the request content and usage pattern, so the saving is checked and the threshold managed during operation.
  - Share of prompt tokens in a prefix already sent by an earlier call (same model): within 5 minutes **3.2%** · within 1 hour 17.7% · across the whole log 30.1% (322 calls, 22 of them sharing a prefix within 5 minutes).
  - Shared prefixes that came back after more than 5 minutes, once the cache had expired: **26.9%**.
  - Shared within 5 minutes but shorter than the vendor's cache minimum (cache_constants.json), so it cannot be cached: **3.2%** (22 of 22 calls checked).
  - Compare: actual cache reads (7-7) 0.0% vs prefixes shared within 5 minutes 3.2%.
  - Method: only whole items count, in the order tools → system → conversation items (without a request body, the prompt in 64-token blocks). Partial matches inside an item are not counted. Where the vendor actually caches (`cache_control` markers, its own block sizes) is decided separately, so this share is how much a cache could read, not a saving.
- [code 7-9] calls that could have used OpenAI Flex (not streamed) — **improvement found**
  - Fix: the engine works out which calls could have used Flex and how much that saves, and proposes it first. Once on, it sends the chosen traffic at Flex. If Flex has no capacity, it retries once at standard (the refusal is not billed). Flex answers come later than standard ones, so it is applied only to work that can wait; which traffic goes to Flex is chosen from the engine's proposed list.
  - Setting improvement: applied by changing a setting or how calls are built. You can apply it yourself with the method below.
  - How: add `service_tier: "flex"` to the calls no one is waiting on (overnight jobs, batch work).
  - 154 calls used models that Flex offers, and 91 of them were not streamed: input 15,476 · output 1,799 tokens, $0.01 (91 priced calls, API usage x price list). Flex runs the same model at half the standard rate (Batch rates, price list 2026-10-21).
  - What it would save: a share of the bill, **0 – 0.2%**. It is 0 if no call can wait, and 0.2% if every call that was not streamed can wait. The log cannot tell which calls can wait.


The results in this section are based on the data provided and the calculation conditions shown.

## 3. Reducible share — the conclusion

- **Reducible calls produce 75.6% of all output tokens** (120 answer pairs), and 93.0% of paired output.
- If the Linklbee engine had decided each call as it came (before the answer, from that document's history only): 61.6% of all output (96 answer pairs). A document's first pair has no history, so it is not reduced.
- Of these, 0 answer pairs resent the same prompt (out of 120 reducible pairs with a prompt), 0.0% of all output. The user may have wanted a different answer. 0 of them came within 600 s. The Linklbee engine treats those as a regenerate and rewrites from scratch, so they are left out of the 'if the engine had decided' and 'without a call' figures. This log has 0 same-prompt resends (200 are needed to fit a window), so the default 600 s window is used.
- Facts in the request: 119 reducible answer pairs have a request body (out of 120). Share of an answer's new tokens that the request gave in advance: token-weighted 97.6% · median 100.0% (p10 96.2%, p90 100.0%) · pairs at 0.5+ 100.0% · control with a different request 18.2% -> **unique share +0.79**. **[no line yet]** No line decides yet whether this value makes a request reducible. For reference (Linklbee's measurements on other materials, not this log — fc3_1018, four real-call materials, token-weighted — pass: drug-label revisions 45 · statute revisions 317; miss: patent notices 60 · product bullets 322; four materials are not a line): materials that passed on quality 74.1% – 75.3% · materials that did not 4.9% – 16.3%.
  - Only pairs with 50+ new tokens (18; a one- or two-token fix scores 100% for free): token-weighted 96.0% · median 96.2% · control 19.5% -> unique share +0.77.

- **Calling at every version wastes many calls (23.4% of output at similarity 0.99+). Calling once every few versions makes them reducible.** With a span of 2 for every document (one call per that many versions): 0.99+ 4.7% · under 0.78, where rewriting the changed parts costs more, 5.8% · 0.85-0.99, where it saves money, 89.5%. The span was fitted on this log (table below).

Span (how many versions back the previous answer is taken from) — share of paired output at 0.99+ / under 0.78 / 0.85-0.99:

Before measuring, answers identical to the previous one were folded (repeats: 0 pairs, 0.0% of span-1 paired output), and answers that went back to an earlier version were left out (reverts: 0 pairs, 0.0%). Neither needs a call at any span, so neither counts toward the span's effect.

| span | answer pairs | >=0.99 | <0.78 | 0.85-0.99 |
|---|---|---|---|---|
| 1 | 138 | 23.4% | 7.0% | 69.6% |
| 2 | 108 | 4.7% | 5.8% | 89.5% |
| 3 | 78 | 0.0% | 3.9% | 96.1% |
| 4 | 48 | 0.0% | 0.0% | 97.8% |
| 6 | 0 | — | — | — |
| 8 | 0 | — | — | — |

Span chosen per length tier (answer-token edges [532.0, 558.0], this log's tertiles): 1*, 1*, 1*. A tier with fewer than 100 answer pairs keeps span 1. A span of 2 or more is chosen only where it beats span 1 on the same answers, including at the 2.5th percentile of a per-document resample.

## 4. Estimated savings (expected range)

This section covers only calls that can be answered by rewriting the changed parts. On a log whose documents do not resemble each other, 0 is the expected result; see section 2.

**This is not a saving measured on this log.** The ranges Linklbee measured on other public documents (below) are applied to this log's reducible calls. Everything else counts as 0.

- output tokens: 32.7% – 61.1% of all output (ranges applied)
- if the engine had decided: 29.4% – 55.1%
  - Counted again with the engine's own rules. On public material (laws, eCFR; logs without a document key) it came to 0.92~1.12 times what the engine actually turned on. Judgments only the engine makes (such as a request with two equally likely previous answers) are not replayed.
- dollars (cost ranges measured without a prompt cache): $0.18 – $0.54 = 17.2% – 50.9% of the bill · prices as of 2026-10-25 · reducible pairs without a price: 0
  - Linklbee sends 10.0% of the reducible pairs in full as its comparison group. The range above is what the rest saves.

Ranges Linklbee measured (three public document sets (product copy, drug label revisions, statute revisions), real calls, failed-edit re-calls included (not for product copy: no call records), 2026-10):
- similarity >=0.95 · answer 200+ tokens: output 55% – 95%, cost 30% – 75%
- similarity 0.78-0.95 · answer 200+ tokens: output 15% – 65%, cost -5% – 50%
- The estimated savings in section 4 were measured on other materials (the public documents named at the end of section 4). The real value for this traffic comes from a pilot that holds back some calls for comparison

## 5. Answer pairs by similarity and length

138 answer pairs from 170 documents. Each pair compares an answer with the previous answer for the same document. Similarity here is measured with the answer known. How documents were found: 308 calls by the document key in the log · 0 calls grouped by the scanner (0 pairs) · 0 skipped for an identical timestamp.

| similarity \ length | <200 | 200-500 | 500-2000 | 2000+ |
|---|---|---|---|---|
| >=0.95 | — | — | **100 · 62.4%** | — |
| 0.78-0.95 | — | — | **20 · 13.1%** | — |
| 0.60-0.78 | — | 2 · 0.7% | — | — |
| <0.60 | — | 16 · 5.0% | — | — |

Each cell: answer pairs · share of all output tokens. Bold = reducible (similarity 0.78+ and answer 200+ tokens). Pair similarity, token-weighted 0.934, median 0.975.

## 6. Answering without a call

- The rule (a prompt equal to the previous call's except for digits gets no call) picked 0 answer pairs, 0.0% of all output. Of those, the answer really matched the previous one in: exactly — · allowing digit-only differences —. **The rule looked only at prompts, never at answers.**
- For reference, counted with the answer known (an upper bound): pairs identical to the previous answer 0 · allowing digit-only differences 16

## 7. How much of the bill is output

- output $0.88 of $1.06 total = **82.8%** (322 priced calls, prices as of 2026-10-25)
- Dollars assume per-token billing. If some traffic is flat-rate or committed capacity (PTU), mark it with `--flat <model name>`, or `--flat all` for everything. Marked traffic gets no dollar figure.

- On a plan, **as a usage limit** — how much more work fits in the same limit, counted on this log's 205,562 tokens.
  - Without the 7-2 resends: 1.3% of tokens → about **1.3% more** work in the same limit
  - Without the output of the 7-3 cut first attempts: 0.1% of tokens → about **0.1% more** work in the same limit

## 8. Items that came out 0 — why, and what to look at next

One line per item: (a) the reason, counted on this log · (b) what would change it, or what to look at instead. Every figure is either counted on this log or a line Linklbee set (its name in brackets).

- Section 6, answering without a call — **0%**. (a) 138 answer pairs, 138 of them with a prompt; none match the previous call's prompt apart from digits. For reference, counted with the answer known (an upper bound): pairs identical to the previous answer 0 · allowing digit-only differences 16. (b) The rule picks traffic that repeats the same request (fixed templates, FAQs).
- [code 7-7] prompt cache hit rate — none. (a) hit rate above the cache minimum 0.0%, at or above the 0.0% line (the hit-rate floor). (b) nothing to cut here within this log (no waste).

These items came out at 0 on this analysis. That is not an analysis error; each line above gives the reason and what more would be needed.

Send this report to bee@linklbee.com and we will look deeper, free of charge. The report holds no prompt or answer text — only numbers, patterns and model names. No need to send the log.

took 2.2s