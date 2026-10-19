# 링클비 스캐너 (Linklbee scanner)

**LLM 호출 로그를 읽어, 같은 문서의 다음 판을 쓸 때 이전 답을 초안으로 고쳐 쓰게 하면 출력 토큰이 몇 % 줄어드는 자리인지를 자사 기계에서 리포트 한 장으로 냅니다.**

파일 하나(`linklbee_scan.py`), 파이썬 3.9 이상, 표준 라이브러리만. `pip install` 도 가입도 없습니다.

```
python3 linklbee_scan.py --logs calls.jsonl --out report.md            # 한국어 리포트
python3 linklbee_scan.py --logs calls.jsonl --lang en --out report.md  # English report
```

로그가 없어도 바로 돌려볼 수 있습니다: `examples/` 의 **[합성]** 예시 로그와 그 리포트를 보십시오(아래 "예시").

**답 본문을 안 남기는 회사도 1절은 나옵니다.** `model`·`input_tokens`·`output_tokens` 만 있으면 "청구서에서 출력이 몇 %인가"가 본문 없이 계산됩니다. 링클비가 줄이는 것이 그 출력입니다.

## 먼저 `--peek` — 돌리기 전에 "이렇게 읽겠습니다"를 봅니다

```
python3 linklbee_scan.py --logs calls.jsonl --peek
```

앞 200줄만 읽고, 어느 필드를 어디서 읽을지 예시와 함께 보여 줍니다. 아무것도 쓰지 않고 1초 안에 끝납니다. 못 찾은 필드는 후보 경로를 보여 줍니다.

```
ts                  ← created                            (epoch 초)  예: 2026-10-17 09:12:33
output              ← choices.0.message.content          예: "제12조 신청인은…" (1,240 토큰)
output_tokens       ← usage.completion_tokens            예: 900
input_tokens        ← usage.prompt_tokens                예: 3,000
→ 60줄 중 60줄을 읽을 수 있습니다(ts 있음), 그중 본문 있음 60줄.
```

## 로그 모양 — OpenAI·Anthropic 원형은 그대로 읽습니다

SDK 가 돌려준 응답 객체를 한 줄에 하나씩 JSON 으로 저장했다면 **변환 없이 됩니다.** 아래 경로만 읽고, 추측해서 아무 문자열이나 본문으로 삼지 않습니다. 무엇을 무엇으로 읽었는지는 리포트 0절에 적힙니다.

| 진단기 필드 | 이 이름이 없으면 읽는 곳 |
|---|---|
| `ts` | `created` (OpenAI) · `created_at` (OpenAI Responses) · `timestamp` · `time` · `start_time` — epoch 초·밀리초 또는 ISO-8601 |
| `output` | `choices.0.message.content` · `choices.0.text` · `content` (Anthropic, text 조각을 이어 붙임) · `output[].content[].text` (Responses) · `completion` · `response` · `text` |
| `output_tokens` | `usage.completion_tokens` · `usage.output_tokens` |
| `input_tokens` | `usage.prompt_tokens` · `usage.input_tokens` (Anthropic 은 `cache_read_input_tokens`·`cache_creation_input_tokens` 를 더함 — 그쪽은 캐시분이 input_tokens 밖에 있습니다) |
| `cached_input_tokens` | `usage.prompt_tokens_details.cached_tokens` · `usage.input_tokens_details.cached_tokens` · `usage.cache_read_input_tokens` |

OpenAI Batch API 결과 줄(`response.body` 안)과 `{"request": …, "response": {…}}` 로 감싼 줄도 같은 경로로 읽습니다.

**표에 없는 모양이면 `--map`** — 스크립트를 쓰지 않고 한 줄로 맞춥니다. 점 표기, 배열은 번호(`choices.0.message.content`).

```
python3 linklbee_scan.py --logs export.jsonl --map ts=created output=data.body.text output_tokens=meta.usage.out --peek
```

**CSV 도 읽습니다.** 확장자(`.csv`·`.tsv`)나 첫 줄로 판별하고, 머리 줄을 필드 이름으로 봅니다. `--map` 은 열 이름에도 걸립니다(`--map ts=When output=AnswerBody`).

## 이 파일이 아직 없다면

- **답 본문까지 남기고 있다** → 위 표대로 바로 됩니다. 모양이 다르면 `--peek` 를 보고 `--map` 한 줄.
- **토큰 수만 남긴다** → 1절(청구서에서 출력이 몇 %인가)은 본문 없이 나옵니다. 2~5절(겹침·절감 밴드)은 본문 또는 토큰 해시(`--hashify`)가 있어야 나옵니다.
- **아무것도 안 남긴다** → `ts` 와 `output` 두 개만 남기기 시작하면 됩니다. 기존 호출 바로 뒤에 아래 다섯 줄을 붙이십시오(의존성 0). `model`·토큰 수까지 넣으면 금액이 나옵니다. 같은 문서의 판들을 묶는 id 가 있으면 `"group_key"` 로 넣으십시오.

Python (OpenAI SDK; Anthropic 이면 `resp.content[0].text`, `u.input_tokens`, `u.output_tokens`):

```python
import json, time
def log_call(resp, path="calls.jsonl"):  # resp = client.chat.completions.create(...) 의 반환값
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

## 아무것도 보내지 않습니다 — 직접 확인하십시오

이 파일에는 네트워크 코드가 0줄입니다. AI 호출 0, 텔레메트리 0.

```
grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py      # -> 0
```

빌드할 때 같은 검사를 하고, 한 줄이라도 걸리면 빌드가 실패합니다. 파일 끝에 든 o200k 토큰 표는 불러올 때 sha256 으로 검사합니다(`RANK_SHA256`).

## 필요한 로그 필드는 두 개입니다

한 줄 = LLM 호출 한 번 (JSONL 또는 CSV). 아래 이름이 아니어도 위 표의 이름이면 읽습니다.

```json
{"ts": "2026-10-17T09:12:33Z", "output": "그때 모델이 낸 답 본문"}
```

| 필드 | | 없으면 |
|---|---|---|
| `ts` | **필수** — 시각 (ISO-8601 또는 epoch 초) | 그 줄을 거부합니다 (시간 순서 없이 짝을 지으면 미래가 섞입니다) |
| `output` | **필수** — 답 본문 | 겹침을 못 잽니다. 답 길이만 재고 상한으로만 적습니다 |
| `group_key` | 권장 — 같은 문서·같은 건의 묶음 키. **호출 종류(분류·답변 등)가 아니라 문서 하나**(그 문서의 판들이 같은 키) | 내용 유사도로 스스로 묶고 리포트에 그렇게 적습니다 |
| `model`, `input_tokens`, `output_tokens` | 권장 | 금액으로 못 바꿉니다 / 토큰을 본문에서 셉니다 |
| `prompt` | 선택 | 요청에 "무엇이 바뀌었는지"가 들어 있는지(3축)를 못 봅니다 |

## 본문을 파일에도 남기기 싫으시면 — `--hashify`

먼저 본문을 회사만 아는 값(솔트)을 붙인 토큰 해시로 바꾼 뒤 그 파일로 돌립니다. 겹침은 토큰이 같은지만 보므로 같은 값이 나옵니다.

```
python3 linklbee_scan.py --hashify calls.jsonl --out hashed.jsonl --salt <회사만 아는 값>
python3 linklbee_scan.py --logs hashed.jsonl --out report.md
```

## 리포트에는 숫자만 들어갑니다

본문은 메모리에서만 읽습니다. 리포트(`--out`), 통계 JSON(`--json`), 요약 파일(`--share`)에는 **본문도, 묶음 키도, 프롬프트도 들어가지 않고** 개수·비율·금액만 들어갑니다. 리포트 머리에는 입력 파일 이름과 모델 이름이 찍힙니다.

확인한 방법: 예시 로그의 모든 답·키·프롬프트에 표지 문자열을 심어 돌린 뒤 네 산출물(한국어·영어 리포트, JSON, 요약)에서 그 문자열과 키·문서 제목을 찾았고, 0건이었습니다(2026-10-18).

리포트를 저희에게 보낼지는 직접 정하시면 됩니다. 보내고 싶으시면 `--share summary.json` 으로 떨어지는 통계 파일만 직접 골라 보내시면 됩니다.

## 리포트를 읽는 법

- **켜짐 칸 비중**(2·3절) — 이전 답과 다음 답의 토큰 겹침이 0.78 이상이고 답이 200토큰 이상인 칸에 든 출력이 전체 출력의 몇 %인가. **이 로그에서 잰 값입니다.**
- **절감 범위**(4절) — 이 로그에서 잰 값이 **아닙니다.** 저희가 공개 문서로 실호출해 잰 범위를 켜짐 칸에 대입한 밴드이고, 리포트에 그렇게 적힙니다. 자사 트래픽의 실제 값은 홀드아웃 파일럿에서 나옵니다.
- 겹침 0.78, 길이 200 은 다른 데이터에서 읽은 기본선입니다. 데이터마다 맞는 선이 다르고, 리포트의 폭(몇 판에 한 번 부르는가) 표처럼 이 로그로 다시 맞춘 값은 그렇다고 적힙니다.

## 안 되는 것

- **1회성 호출만 하는 서비스는 절감 0 으로 나오는 것이 정상입니다.** 이전 판이 있는 문서를 다시 쓰는 호출이 없으면 줄일 자리가 없습니다.
- **겹침 0.78 미만에서 켜면 돈을 더 씁니다.** 그 칸의 법령 개정 207건 실호출에서 출력 +19%, 품질(실제 다음 판 대비 토큰 F1) 0.9175 → 0.6865, 207건 중 100건이 잘렸습니다. 리포트가 이 칸을 따로 셉니다.
- **겹침 0.99 초과면 부르지 않는 것이 맞습니다.** 이전 판이 이미 정답에 가깝습니다 — 101건에서 이전 판 그대로 0.9938, 고쳐 쓰게 한 답 0.9815. 이 칸이 출력의 20% 이상이면 리포트가 "호출 자체가 손해" 경고를 냅니다.
- **법적 효력이 있는 문서에는 LLM을 아예 부르지 않는 방식(호출 0)을 쓰지 마십시오.** 특허 통지 재료에서 호출 없이 낸 334건 중 19건이 실제와 달랐고, 그중 17건이 심사관 이름·법조항 번호·청구항 번호 차이였습니다. 토큰 겹침으로는 안 잡힙니다. 리포트 5절의 호출 0 후보는 참고용 상한입니다.

## 예시 — [합성]

`examples/calls.sample.jsonl` 은 **지어낸** 로그입니다(고객 데이터·공개 말뭉치 원문 없음, 줄마다 `"synthetic": true`). 가상의 서비스 약관 24종 × 6판, 대부분이 새 글인 문서 6종 × 4판, 이전 판이 없는 1회성 답 80건. `examples/report.sample.ko.md` · `report.sample.en.md` 는 이 파일로 돌린 실제 출력입니다. 숫자는 지어낸 로그의 값이고, 어떤 서비스의 절감률도 아닙니다.

```
python3 linklbee_scan.py --logs examples/calls.sample.jsonl --out my_report.md
```

## 라이선스

MIT (`LICENSE`). Copyright (c) 2026 NOTEGG. 포함된 o200k 토큰 표는 tiktoken 의 MIT 라이선스를 따릅니다(`THIRD_PARTY_NOTICES`).

---

## English

**Reads an LLM call log on your machine and reports what share of your output tokens sits where writing the next version of a document from the previous answer, as a draft, cuts output.**

One file, Python 3.9+, standard library only. No install, no sign-up.

```
python3 linklbee_scan.py --logs calls.jsonl --lang en --out report.md
```

**Sends nothing.** Zero lines of network code, zero AI calls, zero telemetry. Check it yourself:
`grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py` → `0`. The build fails if this finds a single line.

**Two required fields per line**: `ts` (time) and `output` (the answer text). `group_key` (one document or case across its versions -- not the kind of call), `model`, token counts and `prompt` make it more precise.

**No answer bodies kept? Section 1 still comes out.** With `model`, `input_tokens` and `output_tokens` only, the report says what share of your bill is output -- the part Linklbee shortens.

**Raw vendor logs are read as they are.** One SDK response object per line works without conversion: `created` / `created_at` / `timestamp` / `time` / `start_time` for `ts`; `choices.0.message.content`, `choices.0.text`, Anthropic `content` (text blocks joined), Responses `output[].content[].text`, `completion`, `response`, `text` for `output`; `usage.completion_tokens` / `usage.output_tokens`, `usage.prompt_tokens` / `usage.input_tokens` (Anthropic: plus its cache read and write tokens, which it counts outside input_tokens), and the cached-token fields of both. OpenAI Batch output lines (`response.body`) too. Only these paths are read; section 0 of the report lists what was read as what.

- `--peek` reads the first 200 records and prints what would be read as what, with an example, and candidate paths for a field it cannot find. Nothing is scanned or written.
- `--map ts=created output=data.body.text output_tokens=meta.usage.out` points a field at any path (dots, list items by number) or CSV column.
- CSV is read too (by extension `.csv`/`.tsv` or by the first line; the header row gives the field names).

**No log file yet?** Append one line per call right after it. Python (OpenAI SDK; for Anthropic use `resp.content[0].text`, `u.input_tokens`, `u.output_tokens`):

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

**Keep text out of files too**: `--hashify calls.jsonl --out hashed.jsonl --salt <secret>`, then scan `hashed.jsonl`; overlap comes out the same.

**The report holds numbers only** — no answer text, no keys, no prompts (checked by planting marker strings in every field of the example log: 0 hits in all four outputs). The header shows the input file name and model names.

### Where the savings numbers come from

The scanner measures **your** share from **your** logs. The savings band it applies to that share comes from the runs below — real paid calls against public revision datasets, each item called twice (once through Linklbee, once as a full rewrite), with quality scored against the version that was actually published next.

| Dataset (public) | Calls | Output tokens | Cost | Quality: Linklbee | Quality: full rewrite |
|---|---|---|---|---|---|
| Korean statute amendments | 317 | −57.1% | −38.2% | 0.9777 | 0.9811 |
| Korea MFDS drug-label revisions | 45 | −75.1% | −57.0% | 0.971 | 0.9628 |
| Product manual revisions (Arduino docs) | 327 | −87.5% | −68.7% | 0.9995 | 0.988 |
| Knowledge-base revisions (Wikipedia sections) | 308 | −91.0% | −64.1% | 0.9993 | 0.997 |

The statute, manual and wiki runs are **update work where the change is given** — the diff carrying the new wording is part of the request, as it is in a statute amendment. Reasoning tokens were **off on both arms** in all four runs.

Model `claude-sonnet-5-5`, 2026-10-17 to 19. **Sample sizes are where each run's budget ended, not a design choice.**

One further run (79 US eCFR sections) is held back while we re-check how it was measured.

**The cost figures include the cost of re-calling as a full rewrite when an edit fails to apply.** We did not take that out.

These come from public data. They are not a promise about your traffic — which is the entire reason this scanner exists.

**What it does not do**
- A service that only makes one-off calls should get 0. That is the correct result.
- Below 0.78 overlap the edit call costs more (207 real calls: output +19%, quality 0.9175 → 0.6865, 100 of 207 truncated).
- Above 0.99 overlap, don't call at all — the previous version is already closer (101 pairs: 0.9938 vs 0.9815).
- Don't skip the model entirely (zero-call) for legally binding documents: 19 of 334 zero-call patent notices differed from the real one, 17 of them in names and article or claim numbers.

**Example [synthetic]**: `examples/calls.sample.jsonl` is an invented log (every line `"synthetic": true`); `examples/report.sample.en.md` is the scanner's actual output on it.

MIT (`LICENSE`). Copyright (c) 2026 NOTEGG. The embedded o200k table is MIT (tiktoken), see `THIRD_PARTY_NOTICES`.
