# 2026-06-20 · Mantle 엔드포인트 vs 기존 Bedrock Inference Profile

## 질문
"맨틀로 가면서 기존 베드락의 추론 프로파일이 안된다는거죠? 맞나요?"

## 답: 예, 정확합니다

| 항목 | 기존 Bedrock (Claude, Mistral, Nova 등) | Mantle (Grok 4.3) |
|------|----------------------------------------|-------------------|
| 엔드포인트 | `bedrock-runtime.{region}.amazonaws.com` | `bedrock-mantle.{region}.api.aws` |
| 호출 API | `InvokeModel` / `Converse` | OpenAI Chat Completions / Responses |
| Cross-Region Inference Profile (`us.*`, `global.*`) | ✅ 지원 | ❌ Not supported |
| Application Inference Profile (AIP) + 태그 | ✅ 가능 | ❌ 사용 불가 |
| Cost allocation tag로 팀별 비용 split | ✅ AIP 태그로 가능 | ❌ 불가 |
| In-Region 호출 | ✅ | ✅ (단, us-west-2 only) |
| SigV4 service name | `bedrock` | `bedrock` (동일) |

## 실무적 의미
- 기존에 `us.anthropic.claude-sonnet-4-5-...` ID에 Application Inference Profile을 씌우고 cost allocation tag를 걸어 팀별 비용 분리하던 패턴을 **Grok에는 그대로 적용 불가**.
- Grok 비용 분리는 별도 문서(`2026-06-20_04_grok_cost_tracking.md`)의 우회 방식 사용 필요.
- 향후 AWS가 `bedrock-mantle`에도 AIP/태그 기능을 붙일 가능성은 있으나 2026-06-20 현재 미지원.
