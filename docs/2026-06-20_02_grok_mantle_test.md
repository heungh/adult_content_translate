# 2026-06-20 · Grok 4.3 호출 테스트 (bedrock-mantle, SigV4)

## 참고 문서
https://docs.aws.amazon.com/bedrock/latest/userguide/model-card-xai-grok-4-3.html

## 핵심 스펙 (Grok 4.3 model card 요약)
| 항목 | 값 |
|------|-----|
| Model ID | `xai.grok-4.3` |
| Launch | 2026-06-15 |
| Context | 1M tokens |
| Reasoning | 지원 (`{"effort":"none|low|medium|high"}`, 기본 low) |
| Endpoint | `bedrock-mantle` (NOT `bedrock-runtime`) |
| Base URL | `https://bedrock-mantle.us-west-2.api.aws/openai/v1` |
| API | OpenAI Chat Completions / Responses |
| Converse / InvokeModel | 미지원 |
| Region | **us-west-2 (Oregon)만** |
| Geo / Global cross-region inference ID | **둘 다 Not supported** |
| 인증 | Bedrock long-term API key (Bearer) **또는** IAM SigV4 (service name = `bedrock`) |
| 기본 파라미터 | `temperature` 0.7, `top_p` 0.95, `max_completion_tokens` 131072 |

## 테스트 코드 (SigV4)
```python
import boto3, json, requests
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest

creds = boto3.Session().get_credentials().get_frozen_credentials()
url = "https://bedrock-mantle.us-west-2.api.aws/openai/v1/chat/completions"

def call_grok(messages, max_tokens=512, temperature=0.7):
    body = json.dumps({
        "model": "xai.grok-4.3",
        "messages": messages,
        "max_tokens": max_tokens,
        "temperature": temperature,
    })
    req = AWSRequest(method="POST", url=url, data=body,
                     headers={"Content-Type": "application/json"})
    SigV4Auth(creds, "bedrock", "us-west-2").add_auth(req)  # ← service name 'bedrock'
    r = requests.post(url, data=body, headers=dict(req.headers))
    return r.status_code, r.json()
```

## 테스트 결과
| Test | 결과 | 응답시간 |
|------|------|---------|
| 영어 "pong" ping | HTTP 200, `"pong"` | 0.83s |
| 한→영 문학 번역 ("그는 창밖을 바라보며…") | HTTP 200, `"He gazed out the window and let out a deep sigh. The city's lights shone dimly."` | 0.84s |
| `us.xai.grok-4.3` / `global.xai.grok-4.3` / `xai.grok-4-v1:0` | HTTP 404 `model does not exist` | — |

## SigV4 service name 검증
| service name | HTTP | 비고 |
|--------------|------|------|
| `bedrock` | 200 | ✅ **정답** |
| `bedrock-mantle` | 400 | 인증은 통과, content safety check 트리거 |
| `bedrock-runtime` | 401 | `Credential should be scoped to correct service: 'bedrock-mantle'` (오류 메시지는 mantle이라 적혀있지만 실제 SigV4 서비스명은 `bedrock`) |

## 응답 포맷 (OpenAI 호환)
```json
{
  "choices": [{"message": {"role": "assistant", "content": "..."}}],
  "model": "xai.grok-4.3",
  "usage": {
    "prompt_tokens": 70,
    "completion_tokens": 24,
    "total_tokens": 94
  }
}
```

## 결론
- AWS 자격증명만 있으면 (long-term API key 발급 없이도) **SigV4 + service name `bedrock`** 로 Grok 4.3 호출 가능.
- `xai.grok-4.3`이 유일한 model ID. 어떤 inference profile 접두어도 동작하지 않음.
