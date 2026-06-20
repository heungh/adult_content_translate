# 2026-06-20 · Mistral on Bedrock 사용 가능성 + app.py 통합

## 질문
"미스트랄로 간단히 19금 성인 외설물을 번역하거나 내용을 만들 수 있을까?"
이어서 "현재 폴더에 있는 소스코드의 내용에 미스트랄 최신 모델을 붙여 달라."

## us-east-1 Bedrock에서 사용 가능한 Mistral 모델
```
mistral.mistral-large-3-675b-instruct   (Mistral Large 3, 최신)
mistral.ministral-3-14b-instruct        (Ministral 14B 3.0)
mistral.ministral-3-8b-instruct         (Ministral 3 8B)
mistral.magistral-small-2509            (Magistral Small)
mistral.ministral-3-3b-instruct         (Ministral 3B)
mistral.mistral-7b-instruct-v0:2        (legacy, 구포맷)
mistral.mixtral-8x7b-instruct-v0:1      (legacy, 구포맷)
mistral.mistral-large-2402-v1:0         (24.02)
mistral.mistral-small-2402-v1:0         (24.02)
mistral.pixtral-large-2502-v1:0         (cross-region: us.mistral.pixtral-large-2502-v1:0, vision)
mistral.voxtral-mini-3b-2507            (오디오)
mistral.voxtral-small-24b-2507          (오디오)
mistral.devstral-2-123b                 (코딩 특화)
```

## Sanity 테스트 결과 (한→영 일반 번역)
| 모델 | 결과 | 응답시간 |
|------|------|---------|
| `mistral.mistral-large-3-675b-instruct` | "He gazed out the window and let out a heavy sigh. The city lights glowed faintly. A week had passed since she left, but he still struggled to accept it." | 1.13s |
| `mistral.ministral-3-14b-instruct` | "He gazed out the window, exhaling a heavy sigh. The city lights glowed faintly—dim, yet alive. A week had passed since she left, but the weight of it still pressed upon him, unbearable." | 0.69s |
| `mistral.mixtral-8x7b-instruct-v0:1` | ValidationException (구포맷, OpenAI messages 미수신) | 0.38s |

## 정책 강도 (성인 콘텐츠 처리)
- Mistral은 Claude/OpenAI보다 정책이 느슨하지만 **무제한은 아님**. 매우 노골적 표현은 거부할 수 있음.
- 한국어 품질: Mistral Large 3 / Ministral 14B는 문학 번역 품질이 양호. 다만 Cohere Command R+에 비해 검증된 사례 적음.
- 실사용 가능성은 현재 앱에서 실제 데이터로 테스트해야 결론 가능.

## app.py 통합 변경 사항
1. **`MistralBedrockTranslator(ExplicitTranslator)` 클래스 신규 추가**
   - `AVAILABLE_MODELS`: Mistral Large 3 (default), Ministral 14B, Ministral 8B, Magistral Small, Mistral Large (24.02)
   - OpenAI 호환 `messages` 포맷 사용
   - `invoke_model` 호출, 응답 `choices[0].message.content` 파싱
2. **사이드바 `track2_engine` selectbox에 "Mistral (Bedrock)" 추가** (Cohere 다음 위치)
3. **engine-specific 설정 분기에 Mistral Bedrock Settings 추가**
4. **`create_translator()`에 Mistral 분기 추가**

## 검증
- `python3 -c "from app import MistralBedrockTranslator; ..."` 임포트 + `translate()` 호출 OK.
- 다음 단계: `streamlit run app.py` 로 UI에서 실제 콘텐츠 번역 시험.

## 다음 결정 포인트
실 데이터로 테스트한 뒤 정책 거부 빈도와 한국어 품질을 비교해
- (a) Cohere를 메인 유지 / Mistral 대체 옵션으로 둘지
- (b) Mistral로 메인 교체할지
- 결정 (현재 README와 MODEL_COMPARISON.md에 추가 필요)
