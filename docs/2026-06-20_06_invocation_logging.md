# 2026-06-20 · Bedrock Invocation Logging의 비용측정 실효성

## 질문
"Invocation Logging을 활용하면 정말로 비용 측정이 가능한가요?"

## 한 줄 답
**사용량(토큰 수) 측정은 가능. 거기에 가격표를 곱해 "비용 추정"은 가능. 그러나 청구서와 100% 일치하는 "cost allocation" 수준은 아님.**

## Bedrock Model Invocation Logging이 기록하는 항목
| 필드 | 내용 | 비용 측정에 유효 |
|------|------|:--:|
| `timestamp` | 호출 시각 | ✅ 일자별 집계 |
| `accountId`, `region` | 계정/리전 | ✅ |
| `identity.arn` | 호출자 IAM principal | ✅ 사용자별 집계 |
| `modelId` | 호출된 모델 | ✅ 모델별 가격 적용 |
| `input.inputTokenCount` | 입력 토큰 수 | ✅ |
| `output.outputTokenCount` | 출력 토큰 수 | ✅ |
| `input.inputBodyJson` / `output.outputBodyJson` | 본문 (옵션, 감사용) | — |
| `requestId` | 요청 ID | ✅ CloudTrail join key |

→ 토큰 수 × Bedrock 공식 단가 = **호출별 비용 산출 가능**. Athena 한 줄 쿼리로 사용자/일자/모델별 그룹화.

## 주의 (실제 청구서와 어긋날 수 있는 이유)

### 1. Mantle 엔드포인트(Grok)는 Invocation Logging 대상이 아님 (실측 확정)
- Bedrock Model Invocation Logging은 `bedrock-runtime` (`InvokeModel`/`Converse`) 대상으로 설계되었고, Mantle은 포함되지 않음.

**실측 (2026-06-20, us-west-2 S3 logging 활성 상태):**

| 호출 | UTC 시각 | request-id | S3 도착 | 결과 |
|------|---------|------------|---------|:----:|
| Mistral `bedrock-runtime` (us-east-1, 대조군) | 01:52:34 | 4292e3a3-... | ~01:53:30 (1분 내) | ✅ |
| Grok 4.3 `bedrock-mantle` (us-west-2) | 01:53:19 | req_6a6io... | **10분 동안 0건** | ❌ |

- 추가로 그 직전 4일간(2026-06-16 ~ 06-20) Mantle 호출이 다수 있었음에도 us-west-2 S3 prefix는 06-16 05:10 UTC 이후 비어있음.
- → 결론: **Mantle 호출은 같은 계정·같은 logging 설정으로도 잡히지 않음.** 별도 logging 메커니즘은 2026-06-20 현재 공개되지 않음.
- → Grok 비용 추적은 본 문서 'A. 앱 레벨 usage 로깅'이 사실상 유일한 신뢰 가능 경로.

### 2. 가격 ≠ 청구서 (drift 가능성)
- 가격 변동, 프로모션 크레딧, 약정 할인(Provisioned Throughput, Reserved), 캐시 토큰 할인 등이 적용되면 단순 "토큰×단가" 계산과 청구서가 어긋남.
- Invocation logging 기반 비용은 **추정치(estimate)**로 봐야 함.
- 월말에 Cost Explorer 실제 청구 금액과 reconcile하는 프로세스 필요.

### 3. IAM identity = 사람이 아님
- `identity.arn`은 IAM user / role ARN. 앱이 단일 IAM role로 동작하면 모든 호출이 같은 principal로 찍힘.
- 사용자/팀 차원으로 분해하려면:
  - (a) 사용자별 IAM role + STS AssumeRole (앱 인증과 연동)
  - (b) 또는 앱 레벨 로깅에서 user/team을 같이 적재한 뒤 `requestId`로 invocation log와 join.

### 4. Mantle 측 logging이 없다면 — 앱 레벨 로깅이 유일한 옵션
- Grok에 한해서는 응답 본문의 `usage.prompt_tokens` / `usage.completion_tokens`를 앱이 직접 기록하는 게 사실상 표준.

## 실무 권장 구성
| 모델군 | 1차 측정 | 2차 보정 |
|--------|----------|----------|
| Claude / Mistral / Cohere (`bedrock-runtime`) | Invocation Logging → S3 → Athena | Cost Explorer 월말 대조 |
| Grok (`bedrock-mantle`) | 앱 레벨 usage 로깅 (응답 `usage` 필드) — **유일 신뢰 가능 경로** (Mantle은 Bedrock Invocation Logging 미적용 실측 확인) |
| 팀/사용자 차원 부여 | 앱이 `requestId` + `team` + `user_id`를 사이드 테이블에 기록 | invocation log와 `requestId`로 join |

## 결론
Invocation Logging은 "사용량 측정"엔 충분히 신뢰할 만하고 Athena 한 번이면 사용자/일자/모델별 토큰·추정비용 대시보드 제작 가능.
다만 Grok(Mantle)에는 아직 적용되지 않았고(향후 어떻게 될지는 아직 예측 어려움), 모든 모델에서 청구서와 100% 일치는 아니므로 **앱 레벨 usage 로깅 + Invocation Logging + 월말 Cost Explorer 대조**의 3중 운영이 가장 안전.
