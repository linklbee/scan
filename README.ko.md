# 링클비 스캐너 (Linklbee scanner)

**LLM API 호출 패턴을 분석해 비용 발생 요인과 최적화 기회를 확인합니다.** 진단기는 개발·운영팀의 로컬 환경에서 로그를 분석하며 원본 로그를 외부로 전송하지 않습니다. 캐시 적용 조건, 반복 요청, 출력 제한에 따른 재요청, 공통 접두부, 이전 응답의 재사용 가능성 등을 검토합니다. 문서 개정 작업에서는 이전 응답에서 변경된 부분만 생성할 기회도 확인합니다.

분석에 필요한 필드가 없는 항목은 임의의 수치를 산정하지 않고 제한 사항과 필요한 정보를 표시합니다. **예상 절감액은 실제 절감액이 아니며, 최종 효과는 도입 후 비교 측정합니다.**

Python 3.9 이상에서 표준 라이브러리만으로 실행할 수 있습니다. 회원가입·API 키가 필요하지 않습니다. 파일 하나(`linklbee_scan.py`)를 내려받아 바로 실행합니다. `pip install linklbee-scan` 도 됩니다(같은 인자를 받는 `linklbee-scan` 명령이 생깁니다).

### 로그가 있으신가요?

| | 하는 일 | 단계 |
|---|---|---|
| 있다 | `python3 linklbee_scan.py <파일 또는 폴더> --out report.html` — 변환 없이 넣으세요. JSONL · JSON 배열 · CSV · .log · .gz, 벤더 응답 원형과 관측 도구 내보내기를 그대로 읽고, 폴더면 안의 파일을 전부 읽습니다 | 2 (받는다 · 돌린다) |
| 로그는 없지만 공급사 콘솔·청구 파일은 받을 수 있다 | 받은 파일을 그대로 넣으세요: `python3 linklbee_scan.py usage.json`. OpenAI 사용량(Usage API) · OpenAI 비용(Costs API) · OpenAI 콘솔 사용량 CSV(Activity) · OpenAI 콘솔 비용 CSV · Anthropic 사용량 보고(Admin API) · Anthropic 콘솔 비용 CSV · AWS 비용·사용 보고서(CUR) · Azure 비용 내역 · Fireworks 사용량(CSV·API) · Together 비용 CSV 를 읽습니다. 콘솔에서 내려받은 OpenAI 사용량 CSV 와 비용 CSV 를 한 폴더에 두면 가격표에 없는 모델의 단가를 청구서에서 거꾸로 풀고, 계산한 금액을 청구액과 대조합니다([합성] 예시: `examples/bill/`, 그 출력 `examples/report.bill.ko.md`). 모델별 지출, 입력 대비 출력, 캐시 읽기, 시간대 쏠림이 나옵니다. 호출 하나하나가 없어서 바뀐 부분만 고쳐 쓰기·재전송·잘린 답 같은 항목은 호출 로그가 있어야 봅니다 | 3 (받는다 · 콘솔에서 내려받는다 · 돌린다) |
| 아직 없다 | 아래 "이 파일이 아직 없다면" 의 몇 줄을 호출 직후에 넣고, 쌓인 뒤 같은 명령으로 돌리세요 | 3 (받는다 · 몇 줄 넣는다 · 돌린다) |
| 개발자이고 Claude Code 를 쓴다 | `python3 linklbee_scan.py --claude-code` — 이 컴퓨터의 Claude Code 기록(~/.claude/projects)을 그대로 읽습니다. 설치·변환·업로드가 없습니다. 체험용 입구이고, 회사 서비스의 호출 로그로도 같은 리포트가 나옵니다 | 1 |
| 먼저 보고 싶다 | `python3 linklbee_scan.py --demo` — 동봉한 [합성] 예시 로그로 리포트가 바로 나옵니다 | 1 |

인자 없이 돌리면 이 세 갈래를 화면에 보여 줍니다. 읽을 것을 못 찾아도 오류로 끝내지 않고 같은 안내를 냅니다.

```
python3 linklbee_scan.py calls.jsonl --out report.html                   # 한국어 리포트
python3 linklbee_scan.py calls.jsonl --lang en --out report.html         # English report
```

답 본문을 남기지 않아도 출력 비중은 나옵니다. `model`·`input_tokens`·`output_tokens` 만 있으면 "청구서에서 출력이 몇 %인가"가 본문 없이 계산됩니다. 링클비가 줄이는 것이 그 출력입니다.

## 먼저 `--peek` — 실행 전에 로그를 어떻게 읽을지 확인하세요

```
python3 linklbee_scan.py --logs calls.jsonl --peek
```

앞 200줄만 읽고, 어느 필드를 어디서 읽을지 예시와 함께 보여 줍니다. 아무것도 쓰지 않고 1초 안에 끝납니다. 못 찾은 필드는 후보 경로를 보여 줍니다.

```
ts                  ← created                            (epoch 초)  예: 2026-10-17 09:12:33
output              ← choices.0.message.content          예: "제12조 신청인은…" (1,240 토큰)
output_tokens       ← usage.completion_tokens            예: 900
input_tokens        ← usage.prompt_tokens                예: 3,000
→ 60줄 중 60줄을 읽을 수 있습니다, 그중 ts 있음 60줄 · 본문 있음 60줄.
```

## 로그 모양 — OpenAI·Anthropic 원형은 그대로 읽습니다

SDK 가 돌려준 응답 객체를 한 줄에 하나씩 JSON 으로 저장했다면 변환 없이 됩니다. 아래 경로만 읽고, 다른 값을 짐작해서 답으로 읽지 않습니다. 어느 필드를 무엇으로 읽었는지는 리포트 맨 앞에 나옵니다.

| 진단기 필드 | 이 이름이 없으면 읽는 곳 |
|---|---|
| `ts` | `created` (OpenAI) · `created_at` (OpenAI Responses) · `timestamp` · `time` · `start_time` — epoch 초·밀리초 또는 ISO-8601 |
| `output` | `choices.0.message.content` · `choices.0.text` · `content` (Anthropic, text 조각을 이어 붙임) · `output[].content[].text` (Responses) · `completion` · `response` · `text` |
| `output_tokens` | `usage.completion_tokens` · `usage.output_tokens` |
| `input_tokens` | `usage.prompt_tokens` · `usage.input_tokens` (Anthropic 은 `cache_read_input_tokens`·`cache_creation_input_tokens` 를 더함 — 그쪽은 캐시분이 input_tokens 밖에 있습니다) |
| `cached_input_tokens` | `usage.prompt_tokens_details.cached_tokens` · `usage.input_tokens_details.cached_tokens` · `usage.cache_read_input_tokens` |

OpenAI Batch API 결과 줄(`response.body` 안)과 `{"request": …, "response": {…}}` 로 감싼 줄도 같은 경로로 읽습니다.

**관측 도구·클라우드 내보내기도 그대로 넣으면 됩니다** — `--peek` 를 돌리면 어느 도구의 형식인지 알려 줍니다.

| 읽는 로그 형식 | |
|---|---|
| OpenAI · Anthropic | SDK 응답 원형(Chat Completions · Responses · Messages · Batch) |
| CSV · TSV · JSON 배열 | 머리 줄 = 필드 이름 / 원소 하나 = 호출 하나 |
| Langfuse | observation 내보내기 |
| LangSmith | run 내보내기 |
| Helicone | request 기록 |
| LiteLLM | spend 로그 |
| Amazon Bedrock | 모델 호출 로그(invocation log) · Converse 응답 |
| Google Vertex AI · Gemini API | 응답 원형 |
| Azure OpenAI | Chat Completions 로그 |

필드 경로는 각 도구의 공식 문서에 적힌 것만 씁니다. 모양마다 [합성] 예시 한 파일이 `examples/adapters/` 에 있습니다.

표에 없는 모양이면 `--map` — 스크립트를 쓰지 않고 한 줄로 맞춥니다. 점 표기, 배열은 번호(`choices.0.message.content`).

```
python3 linklbee_scan.py --logs export.jsonl --map ts=created output=data.body.text output_tokens=meta.usage.out --peek
```

CSV 도 읽습니다. 확장자(`.csv`·`.tsv`)나 첫 줄로 판별하고, 머리 줄을 필드 이름으로 봅니다. `--map` 은 열 이름에도 걸립니다(`--map ts=When output=AnswerBody`).
JSON 배열 내보내기도 읽습니다. 파일이 `[` 로 시작하면 한 줄이든 들여쓰기든 원소 하나를 호출 하나로 봅니다(파일 전체를 메모리에 올리지 않습니다).

## 이 파일이 아직 없다면

- 답 본문까지 남기고 있다 → 위 표대로 바로 됩니다. 모양이 다르면 `--peek` 를 보고 `--map` 한 줄.
- 토큰 수만 남긴다 → 1절(청구서에서 출력이 몇 %인가)은 본문 없이 나옵니다. 문서끼리 얼마나 닮았는지와 예상 절감 범위는 답 본문이나 토큰 해시(`--hashify`)가 있어야 나옵니다.
- 아무것도 안 남긴다 → `ts` 와 `output` 두 개만 남기기 시작하면 됩니다. 기존 호출 바로 뒤에 아래 다섯 줄을 붙이세요(의존성 0). `model`·토큰 수까지 넣으면 금액이 나옵니다. 같은 문서의 판들을 묶는 id 가 있으면 `"group_key"` 로 넣으세요.

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

## 아무것도 보내지 않습니다 — 직접 확인하세요

이 파일에는 네트워크 코드가 0줄입니다. AI 호출 0, 텔레메트리 0.

```
grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py      # -> 0
```

빌드할 때 같은 검사를 하고, 한 줄이라도 걸리면 빌드가 실패합니다. 파일 끝에 든 o200k 토큰 표는 불러올 때 sha256 으로 검사합니다(`RANK_SHA256`).

## 로그 필드 — 이 필드가 있으면 이것을 봅니다

한 줄 = LLM 호출 한 번 (JSONL 또는 CSV; JSON 배열이면 원소 하나). 아래 이름이 아니어도 위 표의 이름이면 읽습니다. 로그를 준비할 때 이 표를 체크리스트로 쓰세요. 있는 만큼 보고, 없는 필드는 리포트가 "로그에 이것이 있으면 봅니다"로 적습니다.

```json
{"ts": "2026-10-17T09:12:33Z", "model": "gpt-5.6-luna", "group_key": "약관-17", "prompt": "…", "output": "…",
 "input_tokens": 3000, "output_tokens": 900, "cached_input_tokens": 1024, "status": "completed", "finish_reason": "stop"}
```

| 필드 | 있으면 보는 것 |
|---|---|
| `model` · `input_tokens` · `output_tokens` | 청구서에서 출력이 차지하는 몫(1절), 금액 환산, 캐시 최소 길이 미달(7-5) |
| `ts` — 시각 (ISO-8601 또는 epoch 초) | 같은 문서의 판을 시간 순서로 잇기(2~5절), 재전송 간격(7-2), 접두가 캐시에 남는 시간(7-8). 없어도 7-1·7-3·7-5·7-6·7-7 은 봅니다 |
| `output` — 답 본문 (또는 `--hashify` 의 토큰 해시) | 이전 답과 닮은 정도(2~4절), 과금 대조(7-6) |
| `group_key` — 같은 문서·같은 건의 묶음 키. 호출 종류가 아니라 문서 하나 | 답 쌍을 정확히 묶기. 없으면 내용이 닮은 것끼리 스스로 묶고 리포트에 그렇게 적습니다 |
| `prompt` — 요청 본문 (또는 `prompt_hash`) | 요청이 바뀐 내용을 알려 주는가(3절), 호출 없이 답하기 후보(5절), 같은 요청 재전송(7-2) |
| `status` (또는 HTTP 상태 코드·`error`) | 실패했는데 청구된 호출(7-1) |
| `finish_reason` (또는 `max_tokens`) | 길이 상한에서 잘린 답(7-3) |
| `request` — 요청 원형 (`tools` · `system` · `messages`) | 도구 정의·시스템 지시의 직렬화 흔들림(7-4), 호출끼리 같은 앞부분(7-8), Flex 후보(7-9) |
| `cached_input_tokens` (또는 벤더 원형의 캐시 읽기 값) | 캐시 적중률(7-7) |

## 정액제·약정 용량 트래픽 — `--flat`

정액제(전 직원 구독 등)나 약정 용량(Azure PTU·Vertex 약정 처리량)은 토큰을 줄여도 청구액이 바뀌지 않습니다. 그런 트래픽에 가격표를 곱하면 없는 절감액이 나옵니다. 모델 이름으로 표시하세요. 그 호출은 금액에서 빠지고, 줄이는 출력은 같은 정액으로 더 처리할 수 있는 여유(토큰)로 적힙니다.

```
python3 linklbee_scan.py --logs calls.jsonl --flat gpt-4o            # 이 모델만 정액
python3 linklbee_scan.py --logs calls.jsonl --flat all               # 전부 정액
```

로그에 `billing` 필드(`flat` · `subscription` · `ptu` · `provisioned` 등)가 있거나 Vertex 응답의 `usageMetadata.trafficType` 이 `PROVISIONED_THROUGHPUT` 이면 표시 없이도 그 줄을 정액으로 봅니다.

## 본문을 파일에도 남기기 싫으시면 — `--hashify`

답 본문을 직접 정한 값(솔트)과 섞어 토큰 해시로 먼저 바꾸고, 그 파일로 진단하세요. 닮음은 토큰이 같은지만 보므로 같은 값이 나옵니다.

벤더 응답을 그대로 적은 로그도 됩니다. 결과 파일에는 숫자·모델 이름·시각과 해시만 남습니다. 글이 남은 줄이 있으면 파일을 쓰지 않고 멈춥니다.

콘솔의 사용량·청구 내보내기도 됩니다. 키·프로젝트 id 는 해시로 바꾸고, 이름·메일·설명 칸은 지웁니다.

```
python3 linklbee_scan.py --hashify calls.jsonl --out hashed.jsonl --salt <회사만 아는 값>
python3 linklbee_scan.py --logs hashed.jsonl --out report.md
```

## 리포트에는 숫자만 들어갑니다

본문은 메모리에서만 읽습니다. 리포트(`--out`), 통계 JSON(`--json`), 요약 파일(`--share`)에는 **본문도, 묶음 키도, 프롬프트도 들어가지 않고** 개수·비율·금액만 들어갑니다. 리포트 머리에는 읽은 파일 수와 모델 이름이 찍힙니다(파일·폴더·필드 이름은 찍지 않습니다).

확인한 방법: 예시 로그의 모든 답·키·프롬프트에 표지 문자열을 심어 돌린 뒤 네 산출물(한국어·영어 리포트, JSON, 요약)에서 그 문자열과 키·문서 제목을 찾았고, 0건이었습니다(2026-10-18). 2026-10-25 에는 시스템 지시·도구 이름·도구 설명·필드 이름·사용자 ID·파일 이름까지 표지를 넓혀 다시 찾았고, 0건이었습니다.

`--share summary.json` 을 붙이면 화면에 공유용 요약 몇 줄도 나옵니다(호출 수 · 줄일 여지 추정 · 출력 몫 · 발견한 낭비 항목). 본문·파일 이름·회사명·키는 들어 있지 않으니 그대로 복사해서 붙이면 됩니다.

링클비에 무엇을 보낼지는 직접 정하시고, 보내실 때도 직접 보내세요 — 진단기는 아무것도 보내지 않습니다. 리포트는 위와 같이 프롬프트·답 원문이 없어 그대로 보내셔도 되고, 가장 작게 보내려면 `--share summary.json` 이 쓴 통계 파일만 보내셔도 됩니다.

리포트를 링클비에 보낼지는 직접 정하시면 됩니다. 보내고 싶으시면 `--share summary.json` 으로 떨어지는 통계 파일만 직접 골라 보내시면 됩니다.

## 리포트가 보는 것 — 실린 순서대로 한 줄씩

- 요약 — 새고 있는 돈과 더 줄일 수 있는 돈의 합(7절 금액, 범위는 아래쪽 값). 최소 기준이라는 한 줄
- 0절 읽은 것 — 호출 수·기간·모델, 어느 필드를 어디서 읽었는지, 본문 없는 호출처럼 숫자에서 빠진 몫
- 7절 낭비 점검서 — 7-1 실패 호출 과금 · 7-2 짧은 간격 재전송 · 7-3 길이 상한에서 잘린 답 · 7-4 요청 접두 직렬화 불안정 · 7-5 캐시 최소 길이 미달 · 7-6 과금 대조 · 7-7 캐시 적중률 · 7-8 호출끼리 같은 접두 · 7-9 OpenAI Flex 로 보낼 수 있었던 호출. 항목마다 발견 / 없음 / 더 보려면(로그에 있으면 보는 필드)
- 3절 줄일 수 있는 요청의 비중 — 이전 답을 고쳐 써서 줄일 수 있는 출력이 몇 %인가
- 4절 예상 절감 범위 — 링클비가 잰 범위를 이 로그의 해당 구간에 대입한 값(이 로그에서 잰 절감이 아님)
- 2절 답 쌍 분포 — 3절의 근거 표(문서 닮음 × 답 길이)
- 5절 호출 없이 답하기 후보 — 직전 호출과 프롬프트가 숫자만 빼고 같은 자리
- 1절 청구서에서 출력이 차지하는 몫 — 본문 없이 `model`·토큰 수만으로
- 8절 0이 나온 항목 — 왜 0인지, 무엇이 바뀌면 달라지는지(0이 나온 항목이 있을 때만)

## 리포트를 읽는 법

- 줄일 수 있는 비중 — 이전 답과 다음 답이 토큰 기준 0.78 이상 같고 답이 200토큰 이상인 요청의 출력이 전체 출력의 몇 %인가. 이 로그에서 잰 값입니다.
- 절감 범위 — 이 로그에서 잰 값이 **아닙니다.** 링클비가 공개 문서로 실호출해 잰 범위를 해당 구간에 대입한 예상 범위이고, 리포트에 그렇게 적힙니다. 실제 값은 일부 요청을 비교용으로 남겨 두는 파일럿에서 확인합니다.
- 닮음 0.78, 길이 200 은 다른 데이터에서 읽은 기본선입니다. 데이터마다 맞는 선이 다르고, 리포트의 폭(몇 판에 한 번 부르는가) 표처럼 이 로그로 다시 맞춘 값은 그렇다고 적힙니다.

## 안 되는 것

- 1회성 호출만 하는 서비스는 바뀐 부분만 고쳐 쓰기 칸(2~4절)이 0 으로 나오는 것이 정상입니다. 이전 판을 다시 쓰는 호출이 없으면 이 항목은 0입니다. 다른 낭비 항목은 따로 봅니다.
- **닮음 0.78 미만에서 켜면 돈을 더 씁니다.** 그 칸의 법령 개정 207건 실호출에서 출력 +19%, 품질(실제 다음 판 대비 토큰 F1) 0.9175 → 0.6865, 207건 중 100건이 잘렸습니다. 리포트가 이 칸을 따로 셉니다.
- 닮음 0.99 초과면 부르지 않는 것이 맞습니다. 이전 판이 이미 정답에 가깝습니다 — 101건에서 이전 판 그대로 0.9938, 고쳐 쓰게 한 답 0.9815. 이 칸이 출력의 20% 이상이면 리포트가 3절 수치 뒤에 그 몫을 따로 적습니다.
- **법적 효력이 있는 문서에는 LLM을 아예 부르지 않는 방식을 쓰지 마세요.** 특허 통지 재료에서 호출 없이 낸 334건 중 19건이 실제와 달랐고, 그중 17건이 심사관 이름·법조항 번호·청구항 번호 차이였습니다. 토큰 닮음으로는 안 잡힙니다. 리포트의 "호출 없이 답하기 후보"는 참고용 상한입니다.

## 예시 — [합성]

`examples/calls.sample.jsonl` 은 지어낸 로그입니다(고객 데이터·공개 말뭉치 원문 없음, 줄마다 `"synthetic": true`, 리포트 0절에도 [합성]으로 찍힙니다). 가상의 서비스 약관 24종 × 6판, 대부분이 새 글인 문서 6종 × 4판, 이전 판이 없는 1회성 지원 답 140건(그중 몇 건은 실패 뒤 다시 보냄) — 모두 322건. 위 표의 필드가 전부 들어 있어 리포트의 모든 항목이 나옵니다. `examples/report.sample.ko.md` · `report.sample.en.md` 는 이 파일로 돌린 실제 출력입니다. 숫자는 지어낸 로그의 값이고, 어떤 서비스의 절감률도 아닙니다.

```
python3 linklbee_scan.py --logs examples/calls.sample.jsonl --out my_report.md
```

## 라이선스

MIT (`LICENSE`). Copyright (c) 2026 NOTEGG. 포함된 o200k 토큰 표는 tiktoken 의 MIT 라이선스를 따릅니다(`THIRD_PARTY_NOTICES`).

---

## English

**Reads an LLM call log on your own server, with nothing sent out, and shows on one page where token spend leaks.** It checks for prompts too short for the vendor's cache, the cache hit rate, prefixes repeated across calls, identical calls resent seconds apart, and failed or cut-off calls that were still billed. These can happen in any service. If a service writes new versions of the same documents, it also shows how much output could be saved by rewriting only the parts that changed.

If the log lacks a field an item needs, the report does not guess. It names the field that would show the item. Every number comes from the log provided.

One file (`linklbee_scan.py`). Python 3.9+, standard library only. No `pip install`, no sign-up. Reports are in Korean by default; add `--lang en` for English.

### Have a log?

| | What to do | Steps |
|---|---|---|
| Yes | `python3 linklbee_scan.py <file or folder> --lang en --out report.html`. No conversion: JSONL, JSON arrays, CSV, .log and .gz, raw vendor responses and observability exports are read as they are. A folder is read file by file | 2 (download, run) |
| No, but the vendor's console has a usage or cost file | Pass the file as downloaded: `python3 linklbee_scan.py usage.json --lang en`. Reads the OpenAI Usage API, the OpenAI Costs API, the OpenAI console usage CSV (Activity) and cost CSV, the Anthropic usage report (Admin API), the Anthropic console cost CSV, the AWS Cost and Usage Report, Azure cost details, Fireworks usage (CSV or API) and the Together cost CSV. Put the OpenAI console usage CSV and cost CSV in one folder and it works out the prices of models missing from the price table from the bill, then checks the computed amount against the bill ([synthetic] example: `examples/bill/`, its output `examples/report.bill.en.md`). Shows spend by model, input vs output, cache reads and busy hours. Items that need single calls (rewriting only the changed parts, resends, cut-off answers) need a call log | 3 (download, export from the console, run) |
| Not yet | Add the few lines under "No log file yet?" right after each call. Once calls build up, run the same command | 3 (download, add a few lines, run) |
| A developer using Claude Code | `python3 linklbee_scan.py --claude-code --lang en` reads this machine's Claude Code records (~/.claude/projects) as they are. Nothing to install, convert or upload. It is a way to try the scanner; a company service's call log gives the same report | 1 |
| Just want to see a report | `python3 linklbee_scan.py --demo --lang en` runs on the bundled [synthetic] example log | 1 |

Run it with no arguments to see these options. If it finds nothing to read, it shows the same options instead of an error.

No answer text in the log? Output's share of the bill still comes out. `model`, `input_tokens` and `output_tokens` are enough. That output is what Linklbee shortens.

### Check first with `--peek`

```
python3 linklbee_scan.py --logs calls.jsonl --peek --lang en
```

Reads the first 200 records and prints which field will be read from where, with an example. Writes nothing and finishes in about a second. For a field it cannot find, it lists candidate paths.

```
ts                  ← created                            (epoch s)  e.g. 2026-10-17 09:12:33
output              ← choices.0.message.content          e.g. "Article 12. The applicant…" (1,240 tokens)
output_tokens       ← usage.completion_tokens            e.g. 900
input_tokens        ← usage.prompt_tokens                e.g. 3,000
-> 60 of 60 records can be read: 60 with ts, 60 with an answer body.
```

### Log shapes read without conversion

Raw OpenAI and Anthropic responses work as they are: one SDK response object per line, saved as JSON. The scanner reads only the paths below and never guesses at other values. Section 0 of the report lists what was read from where.

| Scanner field | Read from, when the field itself is missing |
|---|---|
| `ts` | `created` (OpenAI) · `created_at` (OpenAI Responses) · `timestamp` · `time` · `start_time` — epoch seconds or milliseconds, or ISO-8601 |
| `output` | `choices.0.message.content` · `choices.0.text` · `content` (Anthropic, text blocks joined) · `output[].content[].text` (Responses) · `completion` · `response` · `text` |
| `output_tokens` | `usage.completion_tokens` · `usage.output_tokens` |
| `input_tokens` | `usage.prompt_tokens` · `usage.input_tokens` (for Anthropic, plus `cache_read_input_tokens` and `cache_creation_input_tokens`, which it counts outside input_tokens) |
| `cached_input_tokens` | `usage.prompt_tokens_details.cached_tokens` · `usage.input_tokens_details.cached_tokens` · `usage.cache_read_input_tokens` |

OpenAI Batch API output lines (inside `response.body`) and lines wrapped as `{"request": …, "response": {…}}` are read the same way.

Observability and cloud exports work as they are, too. `--peek` names the tool whose format it recognizes.

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

Field paths come only from each tool's official documents. `examples/adapters/` has one [synthetic] example file per format.

For any other shape, use `--map` — one line, no script. Dotted paths, list items by number (`choices.0.message.content`).

```
python3 linklbee_scan.py --logs export.jsonl --map ts=created output=data.body.text output_tokens=meta.usage.out --peek --lang en
```

CSV is detected by extension (`.csv`, `.tsv`) or by its first line, and the header row gives the field names. `--map` works on column names too (`--map ts=When output=AnswerBody`). A JSON-array export is read too: a file starting with `[`, on one line or indented, is one call per element, streamed rather than loaded whole.

### No log file yet?

- If the log already keeps answer text, it works as is (table above). For a different shape, run `--peek` and add one `--map`.
- If it keeps token counts only, output's share of the bill still comes out. How alike the documents are, and the expected savings, need answer text or token hashes (`--hashify`).
- If it keeps nothing, start with just `ts` and `output`: add the five lines below right after the existing call (no dependencies). Add `model` and the token counts to get dollars. If an id ties together the versions of one document, log it as `"group_key"`.

Python (OpenAI SDK; for Anthropic use `resp.content[0].text`, `u.input_tokens`, `u.output_tokens`):

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

### Sends nothing — check it yourself

The file has no network code. No AI calls, no telemetry.

```
grep -cE "ht[t]p|sock[e]t|url[l]ib" linklbee_scan.py      # -> 0
```

The build runs the same check and fails on a single hit. The o200k token table at the end of the file is verified by sha256 when loaded (`RANK_SHA256`).

### Log fields — what each one adds

One line = one LLM call (JSONL or CSV; in a JSON array, one element). The names in the shape tables above work too. Use this table as a checklist when preparing a log. The report uses whatever is there and, for a missing field, says what it would add.

```json
{"ts": "2026-10-17T09:12:33Z", "model": "gpt-5.6-luna", "group_key": "terms-17", "prompt": "…", "output": "…",
 "input_tokens": 3000, "output_tokens": 900, "cached_input_tokens": 1024, "status": "completed", "finish_reason": "stop"}
```

| Field | What it adds |
|---|---|
| `model` · `input_tokens` · `output_tokens` | output's share of the bill (section 1), dollars, prompts under the cache minimum (7-5) |
| `ts` — time (ISO-8601 or epoch seconds) | a document's versions in order (sections 2-5), resend gaps (7-2), how long a prefix stays cached (7-8). Without it, 7-1, 7-3, 7-5, 7-6 and 7-7 still run |
| `output` — answer text (or token hashes from `--hashify`) | how much an answer resembles the previous one (sections 2-4), billing check (7-6) |
| `group_key` — the id of one document or case across its versions, not the kind of call | exact answer pairs. Without it, the scanner groups similar answers itself and says so in the report |
| `prompt` — the request text (or `prompt_hash`) | whether the request says what changed (section 3), answering without a call (section 5), resent calls (7-2) |
| `status` (or an HTTP status code or `error`) | failed calls that were billed (7-1) |
| `finish_reason` (or `max_tokens`) | answers cut off at the length limit (7-3) |
| `request` — the raw request (`tools` · `system` · `messages`) | inconsistent serialization of tool definitions and system text (7-4), prefixes shared across calls (7-8), Flex candidates (7-9) |
| `cached_input_tokens` (or the vendor's cache-read value) | cache hit rate (7-7) |

### Flat-rate or committed capacity — `--flat`

On a flat rate (such as a company-wide subscription) or committed capacity (Azure PTU, Vertex provisioned throughput), fewer tokens do not change the bill. Multiplying that traffic by list prices would show savings that do not exist. Mark it by model name. Those calls get no dollar figure, and output cut there is reported as spare capacity (tokens) on the same plan.

```
python3 linklbee_scan.py --logs calls.jsonl --flat gpt-4o --lang en      # this model only
python3 linklbee_scan.py --logs calls.jsonl --flat all --lang en         # everything
```

A line is treated as flat-rate without the flag if it has a `billing` field (`flat`, `subscription`, `ptu`, `provisioned`, …) or, for Vertex, `usageMetadata.trafficType` = `PROVISIONED_THROUGHPUT`.

### Keep answer text out of files too — `--hashify`

Turn the answer text into token hashes mixed with a value of your choosing (a salt) first, then scan that file. Similarity only checks whether tokens match, so the result is the same.

Logs that keep the vendor's raw responses work too. The output file keeps only numbers, model names, times and hashes. If any line would still hold text, nothing is written and the run stops.

Console usage and billing exports work too. Key and project ids become hashes; name, e-mail and description columns are dropped.

```
python3 linklbee_scan.py --hashify calls.jsonl --out hashed.jsonl --salt <a secret only you know>
python3 linklbee_scan.py --logs hashed.jsonl --lang en --out report.md
```

### The report holds numbers only

Answer text is read in memory only. The report (`--out`), the stats JSON (`--json`) and the summary file (`--share`) hold **no answer text, no document keys and no prompts** — only counts, shares and dollars. The report header shows how many files were read and the model names (no file, folder or field names).

How this was checked: marker strings were planted in every answer, key and prompt of the example log, and the four outputs (Korean and English reports, JSON, summary) were searched for them and for the keys and document titles. 0 hits (2026-10-18). On 2026-10-25 the markers were widened to system instructions, tool names, tool descriptions, field names, user IDs and the file name, and searched again: 0 hits.

With `--share summary.json`, the scanner also prints a few lines to share: call count, estimated room to cut, output's share of the bill, waste items found. No text, file name, company name or keys, so they can be pasted as they are.

Whether to send anything to Linklbee is up to you, and you send it by hand — the scanner sends nothing. The report itself can be sent as it is (it holds no prompt or answer text, as above); the statistics file that `--share summary.json` writes is the smallest choice.

### What the report covers — one line per section

- 0 What was read — calls, period, models, and which field was read from where
- 1 How much of the bill is output — from `model` and token counts alone, no text needed
- 2-3 Similarity x answer length — what share of output could be saved by editing the previous answer
- 4 Estimated savings — ranges Linklbee measured, applied to this log's matching calls (not a saving measured on this log)
- 5 Answering without a call — calls whose prompt matches the previous one apart from digits
- 6 Notes — what is left out of the numbers, such as calls without text
- 7 Waste checklist — 7-1 billed failed calls · 7-2 quick resends · 7-3 answers cut off at the length limit · 7-4 inconsistent prefix serialization · 7-5 prompts under the cache minimum · 7-6 billing check · 7-7 cache hit rate · 7-8 prefixes shared across calls · 7-9 calls that could use OpenAI Flex. Each one is found / none / needs more data (with the field that would show it)
- 8 Items that came out 0 — why, and what would change it (only when an item is 0)

### How to read the report

- Reducible share: the share of all output that comes from calls whose answer matches the previous answer at 0.78 or more (by tokens) and runs 200+ tokens. Measured on this log.
- Savings range: **not** measured on this log. Linklbee measured these ranges with real calls on public documents, and the report applies them to the matching calls and labels them that way. The real value comes from a pilot that holds back some calls for comparison.
- The 0.78 similarity and 200-token lines are defaults taken from other data. The right line differs by data. Where the report refits a value to this log, such as the span table (how many versions between calls), it says so.

### Limits

- A service that only makes one-off calls should get 0 for rewriting only the changed parts (sections 2-4). Without calls that rewrite an earlier version, this item is 0. The waste checklist is judged separately.
- **Below 0.78 similarity, rewriting the changed parts costs more.** On 207 real calls to statute amendments in that range: output +19%, quality (token F1 against the real next version) 0.9175 → 0.6865, and 100 of the 207 were cut off. The report counts that range separately.
- Above 0.99 similarity, don't call at all. The previous version is already closer: on 101 pairs, the previous version scored 0.9938 and the rewritten answer 0.9815. When that range is 20% or more of output, the report notes its share right after the section 3 figures.
- **For legally binding documents, never skip the model call entirely.** On patent notices, 19 of 334 answers produced without a call differed from the real one, 17 of them in examiner names, statute numbers or claim numbers. Token similarity does not catch this. The report's "answering without a call" figure is a reference upper bound.

### Example [synthetic]

`examples/calls.sample.jsonl` is a made-up log: no customer data or public corpus text, `"synthetic": true` on every line, and section 0 of the report labels it [synthetic]. It has fictional terms of service, 24 documents × 6 versions; 6 mostly-new documents × 4 versions; and 140 one-off support answers with no earlier version, a few resent after a failure — 322 calls in all. Every field in the table above is present, so every item in the report comes out. `examples/report.sample.en.md` and `report.sample.ko.md` are the scanner's actual output on it. The numbers belong to a made-up log and are no service's savings.

```
python3 linklbee_scan.py --logs examples/calls.sample.jsonl --lang en --out my_report.md
```

### License

MIT (`LICENSE`). Copyright (c) 2026 NOTEGG. The embedded o200k token table is under tiktoken's MIT license (`THIRD_PARTY_NOTICES`).
