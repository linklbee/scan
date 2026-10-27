# Linklbee LLM API Cost Diagnostic Report

scanner 1.4.7 · 2 files · 한국어: --lang ko

**Cache hit rate 50.8%** — the share of input tokens read from the cache (the vendor's usage totals). Details in section 3. By model it runs from 47.6% to 70.5%.

## 1. What was read — input grade D (vendor usage file)

- format: OpenAI console usage CSV (Activity) + OpenAI console cost CSV (Cost, daily totals) · 18 summary rows
- 991 calls (the vendor's own count, num_model_…)
- billed (cost CSV): $3.12 over 7 days (2026-10-01 to 2026-10-07), daily totals not split by model or key.
- 2026-10-01 to 2026-10-07 · 2 models
- These are totals counted by the vendor (1-minute buckets at the finest), not one row per call. No text, no prompts.
- The scanner has no network code.
- **What this file shows**: output's share of the bill · tokens and dollars by model · the cache-read share · busy hours · the least cache written and never read · a cache-minimum signal (files with request counts) · the most that could go to OpenAI Flex (section 4).
- **What a call log would add**: what is leaking (resends, cut-off answers, unstable prefixes, cache hits) and what additional savings opportunities (prompts under the cache minimum, shared prefixes, rewriting only the changed parts, answering without a call). Vendor totals have no single calls. To start keeping a call log, see "No log file yet?" in the README.

## 2. How much of the bill is output

- output $1.14 of $3.12 total = **36.4%** (2 of 2 models, prices fitted from the bill and the price list as of 2026-10-25)
- gpt-synth-mini: prices from the bill alone -- input $0.40, output $3.00 per 1M tokens, cache read 0.10x. Of 3 days on this model only, 2026-10-04 and 2026-10-06 solved them and the other 1 checked them (largest gap 0.0%).
- bill check (7-6): on the 7 days where every model has a price -- computed $3.12, billed $3.12, gap 0.0%.

## 3. Tokens by model

| model | input | cache read | cache write | output | usd | calls | hit rate |
|---|---|---|---|---|---|---|---|
| gpt-synth-mini | 4,316,168 | 3,918,362 | 0 | 368,303 | $2.99 | 897 | 47.6% |
| gpt-5.6-luna | 393,593 | 942,405 | 0 | 25,213 | $0.13 | 94 | 70.5% |

| API key | calls | input a call | hit rate |
|---|---|---|---|
| key_SYN…AAA | 897 | 9,180 | 47.6% |
| key_SYN…BBB | 94 | 14,212 | 70.5% |
- The hit rate differs by key. One overall number may fit no single key.

- input vs output: 9,570,528 input tokens (cache included) to 393,516 output tokens.
- cache read / input = **50.8%**. Single call lengths are unknown, so this file cannot show calls under the cache minimum.
- 2 models. The one with the most output has 93.6% of it.

## 4. What is leaking, what additional savings opportunities

### Additional savings opportunities

- [code 7-9] what could go to OpenAI Flex — **improvement found (upper bound)**
  - 4 summary rows on Flex-table models (gpt-5.6-luna): 1,335,998 input tokens (cache included) and 25,213 output tokens, $0.13 at list prices. Flex runs the same model at half the standard rate (Batch rates, price list 2026-10-21). Sending all of it at Flex would cost **at most $0.06** less.
  - An upper bound. This file cannot tell which calls can wait. A call log narrows it to calls that were not streamed.

## 5. With a call log

- The recording tool (two lines of code, no text kept) checks what is leaking and what additional savings opportunities, call by call. With answer and request hashes on, it also checks rewriting only the changed parts and answering without a call. Of what a full day would find to fix, the share seen after:
  - 5 min 55% · 30 min 74% · 1 h 87% · 1 day 100%
  - Measured on [synthetic] matrix cells (22 matrix cells, about 60 calls an hour). Less traffic takes longer.

took 0.0s

Send this report to bee@linklbee.com and we will look deeper, free of charge. The report holds no prompt or answer text — only numbers, patterns and model names. No need to send the log.