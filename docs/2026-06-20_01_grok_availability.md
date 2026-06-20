# 2026-06-20 · AWS Bedrock에서 Grok 사용 가능 여부 조사

## 질문
README와 소스코드를 읽고, AWS Bedrock에서 Grok 모델을 사용한 API 호출이 가능한지, 그리고 inference profile(`us.*`, `global.*`)로 Grok을 호출할 수 있는지 확인.

## 검증 절차 & 결과

| 항목 | 결과 |
|------|------|
| `list-foundation-models` providerName 확인 (us-east-1/us-west-2/us-east-2) | xAI **제공자 자체 없음**. 보이는 제공자: AI21 Labs, Amazon, Anthropic, Cohere, DeepSeek, Google, Meta, MiniMax, Mistral AI, Moonshot AI, NVIDIA, OpenAI, Qwen, Stability AI, TwelveLabs, Writer, Z.AI |
| `list-inference-profiles` (us / global / eu / ap 다수 region) | `xai.*`, `grok` 패턴 **0건** |
| `bedrock-runtime` 의 `InvokeModel` 직접 시도 — `xai.grok-4-v1:0`, `us.xai.grok-4-v1:0`, `global.xai.grok-4-v1:0`, `grok-4` 등 | 모두 `ValidationException: The provided model identifier is invalid` |
| Marketplace Model Endpoints | 비어있음 |

## 결론
- 기존 `bedrock-runtime` 경로로는 **Grok 호출 불가**.
- Cross-Region Inference Profile에도 Grok 항목 없음.
- → 사용자 문서(다음 토픽) 안내에 따라 **신규 `bedrock-mantle` 엔드포인트**를 거쳐야 함이 확인됨.
