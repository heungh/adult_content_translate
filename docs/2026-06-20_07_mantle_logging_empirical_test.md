# 2026-06-20 · Mantle(Grok) Invocation Logging 실측 검증

## 질문
"Mantle에서 Invocation Logging이 되는지 확인 — 즉 Grok에서 invocation logging을 통해 비용추적이 가능한지."

## 검증 절차

### 1. 사전 상태 확인
- `aws bedrock get-model-invocation-logging-configuration --region us-west-2`
  → S3 destination: `bedrock-logs-181136804328-us-west-2/bedrock-logs/`
- `aws bedrock get-model-invocation-logging-configuration --region us-east-1`
  → CloudWatch `/aws/bedrock/modelinvocations` + S3 destination 둘 다 활성

### 2. 호출
| 호출 | UTC 시각 | request-id | 응답 |
|------|----------|------------|------|
| Mistral `bedrock-runtime` (us-east-1, 대조군) | 01:52:34 | 4292e3a3-6021-49bd-8795-28ae66f9e37f | HTTP 200, 한→영 번역 |
| Grok 4.3 `bedrock-mantle` (us-west-2) | 01:53:19 | req_6a6ioueepwszs252ongvfvmh76nre2w5wj4cynej27crxnzsyjaa | HTTP 200, 한→영 번역 |

### 3. Polling (30초 간격 · 최대 10분)
| t | us-west-2 (Grok 파일 수) | us-east-1 (Mistral 파일 수) |
|---|---|---|
| 3s | 0 | 1 (이미 도착) |
| 36s ~ 582s | 0 | 1 (변동 없음) |

### 4. CloudWatch 즉시 확인 (us-east-1)
```
2026-06-20T01:52:34+00:00
  model = mistral.mistral-large-3-675b-instruct
  req   = 4292e3a3-6021-49bd-8...
  ident = hheungsu
```
→ 대조군은 호출 즉시 CloudWatch에도 잡힘.

### 5. 추가 확인 (us-west-2 전체 일자)
- `aws s3 ls .../2026/06/20/ --recursive --region us-west-2` → 빈 결과
- 직전 4일간(2026-06-16 ~ 06-20)의 Grok 호출도 us-west-2 S3에 0건 (마지막 기록 시각 2026-06-16 05:10 UTC)

## 결론
**Bedrock Model Invocation Logging은 Mantle 엔드포인트(Grok 4.3) 호출을 잡지 않음.**
- 같은 계정·같은 logging 설정에서 `bedrock-runtime`(Mistral)은 1분 내 기록, `bedrock-mantle`(Grok)은 10분 0건.
- buffering 지연이 아니라 메커니즘 자체가 다른 것으로 판단.
- Mantle 전용 logging 메커니즘은 2026-06-20 현재 공개되지 않음.

→ **Grok 비용 추적은 앱 레벨 usage 로깅(응답 `usage.prompt_tokens`/`completion_tokens` 직접 적재)이 사실상 유일한 신뢰 가능 경로.**

상세 영향은 `docs/2026-06-20_06_invocation_logging.md` 와 `docs/2026-06-20_04_grok_cost_tracking.md` 에 반영됨.
