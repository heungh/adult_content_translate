# 2026-06-20 · Grok AI 토큰 사용량 · 비용 추적 방법

## 배경
Mantle 엔드포인트(Grok 4.3)에는 Application Inference Profile / cost allocation tag를 적용할 수 없어, 팀·사용자·일자별 비용 분해가 곤란함.

## 우회 방법 4가지

### A. 앱 레벨 usage 로깅 (즉시 적용 가능 · 권장)
응답의 `usage` 필드와 호출자 메타데이터를 함께 적재.

```python
resp_json = call_grok(...)             # SigV4 OpenAI compat 호출
usage = resp_json["usage"]
record = {
    "timestamp": datetime.utcnow().isoformat(),
    "team": current_user.team,
    "user_id": current_user.id,
    "model": "xai.grok-4.3",
    "prompt_tokens": usage["prompt_tokens"],
    "completion_tokens": usage["completion_tokens"],
    "input_cost": usage["prompt_tokens"] / 1000 * PRICE_IN,
    "output_cost": usage["completion_tokens"] / 1000 * PRICE_OUT,
}
# → DynamoDB / CloudWatch EMF / S3+Athena
```
- 저장소 추천: CloudWatch Logs(EMF) → Athena → QuickSight, 또는 DynamoDB(`team#date` 파티션).
- 장점: 팀/사용자/세션/문서ID 등 원하는 모든 차원으로 분해 가능, 가격표 변경 즉시 반영.
- 영향 범위: `app.py`의 모델 호출 직후에 logger 1줄.

### B. Bedrock Model Invocation Logging (보완)
Bedrock 콘솔 → Settings → "Model invocation logging" ON → CloudWatch Logs / S3 출력.
- 모든 호출이 input / output / usage / IAM identity와 함께 자동 기록.
- Athena로 `userIdentity.arn`, `modelId`, `inputTokenCount`, `outputTokenCount` 집계.
- ⚠️ Mantle 엔드포인트에서도 logging이 적용되는지는 별도 검증 필요 (2026-06 현재 명시되어 있지 않음).

### C. 사용자/팀별 Bedrock Long-term API Key 분리
콘솔에서 IAM user/role별로 long-term API key 발급 → 키별 sourceIdentity가 CloudTrail에 남음.
- CloudTrail + Model Invocation Logs join → 누구의 호출인지 정확히 분리.
- 팀별 IAM group + cost allocation tag 정책 적용 시 Cost Explorer에서도 일부 분해 가능.

### D. 계정/프로젝트 분리 (조직 차원 솔루션)
AWS Organization 내에 팀별 OU/account 분리 → Consolidated Billing에서 자연스럽게 팀별 split.
- 가장 확실하지만 운영 비용 큼.
- 팀 수가 적고 영구적일 때만 권장.

## 권장 조합: A + B
- A(앱 로그): 풍부한 비즈니스 차원(팀/사용자/문서/요청유형) 제공.
- B(Invocation Logging): 신뢰 가능한 ground-truth (감사·비용대조용).
- 두 로그를 `requestId` + `timestamp` 로 join하면 어떤 차원으로도 비용 분해 가능.

## 한계 (Invocation Logging 신뢰도)
- 토큰 × 단가 = **추정치**. 청구서와 100% 일치 보장 안 됨.
- 가격 변동, 프로모션 크레딧, Provisioned Throughput 약정 할인, 캐시 토큰 할인 등이 적용되면 drift 발생.
- 월말에 Cost Explorer 실제 청구액과 reconcile하는 프로세스 필요.
