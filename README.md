[한국어 → README.ko.md](https://github.com/linklbee/scan/blob/main/README.ko.md)

# Linklbee scanner

**Identify LLM API cost drivers and optimization opportunities from actual call patterns.** The scanner analyzes logs locally without uploading raw logs. It reviews caching conditions, repeated requests, output limits and retries, shared prefixes, and potential response reuse. For document revisions, it identifies opportunities to generate only the changed portions of a previous response.

If essential fields are missing, the report explains the limitation rather than inventing estimates. **Estimated savings are not realized savings; actual results require measurement after integration.**

## Download and run

The whole tool is one file. Download `linklbee_scan.py` and run it:

```
python3 linklbee_scan.py <log file or folder> --lang en --out report.html
python3 linklbee_scan.py --demo --lang en          # see a report on a [synthetic] example log first
```

`pip install linklbee-scan` works too; it installs the `linklbee-scan` command, which takes the same arguments (the examples below use it).

Python 3.9+, no dependencies. Reports are in Korean by default; `--lang en` gives English. Run it with no arguments and it looks in the current folder for something to read.

Not sure how your log will be read? `--peek` reads the first 200 records, prints which field comes from where, and writes nothing:

```
linklbee-scan calls.jsonl --peek --lang en
```

## Log formats it reads

No conversion needed. JSONL, JSON arrays, CSV/TSV, `.log` and `.gz` files are read as they are; a folder is read file by file.

| Format | |
|---|---|
| OpenAI · Anthropic | raw SDK responses (Chat Completions · Responses · Messages · Batch) |
| CSV · TSV · JSON array | header row = field names / one element = one call |
| Langfuse | observation export |
| LangSmith | run export |
| Helicone | request record |
| LiteLLM | spend log |
| Amazon Bedrock | model invocation log · Converse response |
| Google Vertex AI · Gemini API | raw responses |
| Azure OpenAI | Chat Completions log |

Any other shape: map fields in one line, no script — `--map ts=created output=data.body.text output_tokens=meta.usage.out`.

No call log, only the vendor's usage or cost file? Pass it as downloaded. It reads the OpenAI Usage and Costs APIs, OpenAI console usage and cost CSVs, the Anthropic usage report (Admin API), the Anthropic console cost CSV, the AWS Cost and Usage Report, Azure cost details, Fireworks usage and the Together cost CSV. You get spend by model, input vs output, cache reads and busy hours.

A developer using Claude Code can try it on this machine's own records: `linklbee-scan --claude-code --lang en`.

## What the report finds

- Output's share of the bill — from `model` and token counts alone
- How much of the output could be saved by editing the previous answer instead of writing it again
- Calls that could be answered without a model call
- Failed calls that were billed
- Identical calls resent within seconds
- Answers cut off at the length limit
- Tool definitions and system text serialized inconsistently, which breaks the prompt cache
- Prompts under the vendor's cache minimum
- A billing check: billed output tokens against the answer text's own token count
- Cache hit rate
- Prefixes shared across calls that could be cached
- Calls that could use OpenAI Flex

Each item is reported as found, none, or needs more data — with the field that would show it.

## No log file yet?

Add these lines right after each call (OpenAI SDK; for Anthropic use `resp.content[0].text`, `u.input_tokens`, `u.output_tokens`). Once calls build up, run the scanner on `calls.jsonl`.

```python
import json, time
def log_call(resp, path="calls.jsonl"):  # resp = what client.chat.completions.create(...) returned
    u = resp.usage
    with open(path, "a", encoding="utf-8") as f:
        f.write(json.dumps({"ts": time.time(), "output": resp.choices[0].message.content, "model": resp.model, "input_tokens": u.prompt_tokens, "output_tokens": u.completion_tokens}, ensure_ascii=False) + "\n")
```

TypeScript (Node, OpenAI SDK):

```ts
import { appendFileSync } from "node:fs";
export function logCall(resp: any, path = "calls.jsonl") {  // resp = await client.chat.completions.create(...)
  const u = resp.usage ?? {};
  appendFileSync(path, JSON.stringify({ ts: Date.now() / 1000, output: resp.choices[0].message.content, model: resp.model, input_tokens: u.prompt_tokens, output_tokens: u.completion_tokens }) + "\n");
}
```

## Privacy

- **No network code.** No AI calls, no telemetry. Check it: `grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py` prints `0`.
- **Standard library only.** One file, nothing else to install or trust.
- **Your log never leaves your machine.** Answer text is read in memory only; the report holds counts, shares and dollars — no prompts, no answers, no document keys.

## License

MIT (`LICENSE`), published by NOTEGG Corp. Copyright (c) 2026 NOTEGG. The embedded o200k token table is under tiktoken's MIT license (`THIRD_PARTY_NOTICES`).

Full documentation — every field, `--flat`, `--hashify`, how to read the report, and its limits — is in [README.ko.md](https://github.com/linklbee/scan/blob/main/README.ko.md) (Korean, with the full English reference at the end under "English").
