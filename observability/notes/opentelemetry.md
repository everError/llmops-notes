# OpenTelemetry(OTel) 정리

## 1. OpenTelemetry란

**관측성 데이터(트레이스, 메트릭, 로그)를 생성·수집·전송하는 방법의 표준**. CNCF 프로젝트로, 특정 벤더 도구가 아니라 **스펙 + SDK + 프로토콜의 묶음**이다.

핵심 문제의식: 예전에는 관측 도구마다 계측 방식이 달랐다. Datadog을 쓰다 Jaeger로 바꾸면 계측 코드를 전부 다시 짜야 했고, 라이브러리들은 어느 벤더를 지원할지 각자 선택해야 했다. OTel은 이걸 표준화해서:

- **계측은 한 번만** — 코드에는 OTel API로 계측하고, 데이터를 어디로 보낼지는 설정으로 바꾼다
- **백엔드는 자유** — Jaeger, Prometheus, Grafana, Datadog, Langfuse 등 OTel을 받는 곳이면 어디든
- **라이브러리 생태계 공유** — HTTP 클라이언트, DB 드라이버 등은 OTel 계측을 내장하기 시작 (자동 계측)

> 비유하면: 관측 데이터계의 **USB 표준**. 장비(앱)와 충전기(백엔드)가 서로를 몰라도 규격만 맞으면 연결된다.

## 2. 세 가지 신호(Signals)

| 신호        | 무엇인가                                   | 예                                      |
| ----------- | ------------------------------------------ | --------------------------------------- |
| **Traces**  | 요청 하나가 거친 경로의 시간 기록          | "이 API 호출이 DB→캐시→외부API를 거침" |
| **Metrics** | 시간에 따라 집계되는 숫자                  | 초당 요청 수, GPU 사용률, 대기 큐 길이  |
| **Logs**    | 시점성 텍스트 이벤트                       | 에러 메시지, 디버그 출력                |

셋은 상호 보완이다: **메트릭으로 이상을 감지**하고("p99 지연이 튀었다"), **트레이스로 위치를 좁히고**("검색 span이 느리다"), **로그로 원인을 확인**한다("timeout 에러"). OTel의 강점은 세 신호가 **같은 컨텍스트(trace_id)로 연결**된다는 것 — 메트릭 이상 지점에서 해당 트레이스로, 트레이스에서 그 시점 로그로 바로 넘어갈 수 있다.

## 3. 트레이스의 구조

```
Trace  (trace_id 하나로 묶인 전체 여정)
└── Span: HTTP POST /chat          [────────────────── 820ms]
    ├── Span: retrieval             [──── 210ms]
    │   └── Span: vector-db query    [── 180ms]
    └── Span: llm-generation             [─────────── 590ms]
```

- **Span** = 이름 + 시작/종료 시각 + attributes(키-값) + 부모 span 참조
- **Trace** = 같은 trace_id를 공유하는 span들의 트리
- **Context Propagation** = 서비스 경계를 넘을 때 trace_id를 전달하는 메커니즘 (HTTP 헤더 `traceparent` — W3C Trace Context 표준). 이게 있어서 **BFF → gRPC 서비스 → DB**로 이어지는 분산 호출이 하나의 트레이스로 이어진다

Langfuse의 Trace/Span/Generation이 익숙하게 느껴졌다면 맞다 — **Langfuse의 데이터 모델은 OTel 트레이스 모델의 LLM 특화판**이고, Python SDK v3는 실제로 OTel SDK 위에 구현되어 있다. Generation은 "LLM 속성(model, usage)을 가진 span"이다.

## 4. Semantic Conventions (시맨틱 규약)

span attribute의 **이름을 표준화**한 규약. `http.request.method`, `db.system`처럼 정해진 키를 쓰면 어떤 백엔드든 의미를 해석할 수 있다.

LLM 쪽에는 **GenAI Semantic Conventions**가 정의되는 중:

- `gen_ai.system` (openai, anthropic, ...)
- `gen_ai.request.model`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` 등

이 규약 덕분에 OTel로 계측된 LLM 호출을 Langfuse가 받아서 Generation으로 해석할 수 있다 — Langfuse는 OTLP 엔드포인트(`/api/public/otel`)로 **OTel 네이티브 트레이스도 직접 수신**한다.

## 5. 구성 요소 (파이프라인)

```
[앱 코드]
  API ──▶ SDK ──▶ Exporter ──OTLP──▶ Collector (선택) ──▶ 백엔드(들)
```

| 구성 요소     | 역할                                                                              |
| ------------- | --------------------------------------------------------------------------------- |
| **API**       | 계측 인터페이스. 라이브러리는 API에만 의존 (SDK 없으면 no-op → 오버헤드 없음)     |
| **SDK**       | API의 실제 구현. 샘플링, 배치, 리소스 정보 부착                                   |
| **Exporter**  | 데이터를 특정 형식으로 내보내는 플러그인 (OTLP, Prometheus, console 등)           |
| **OTLP**      | OTel 표준 전송 프로토콜 (gRPC :4317 / HTTP :4318)                                 |
| **Collector** | 독립 프로세스. 수신 → 가공(필터, 샘플링, 민감정보 마스킹) → 여러 백엔드로 분배    |

**Collector는 선택이지만 운영에서는 거의 필수** — 앱은 로컬 Collector로만 보내고, 백엔드 교체·다중화·샘플링 정책은 Collector 설정에서 처리한다. 앱 재배포 없이 관측 파이프라인을 바꿀 수 있게 하는 완충 지대.

## 6. 계측 방식

1. **Zero-code (자동 계측)** — 코드 수정 없이 에이전트/후킹으로 유명 라이브러리(HTTP, DB 등)를 자동 계측. Python은 `opentelemetry-instrument` 래퍼, .NET/Java는 런타임 에이전트
2. **라이브러리 내장** — 프레임워크가 OTel 계측을 자체 포함 (ASP.NET Core가 대표적)
3. **수동 계측** — 비즈니스 로직 단위 span을 직접 생성

```python
from opentelemetry import trace

tracer = trace.get_tracer("my-service")

with tracer.start_as_current_span("process-order") as span:
    span.set_attribute("order.id", order_id)
    ...
```

Langfuse SDK의 `start_as_current_observation`과 모양이 같은 이유 = 같은 OTel 패턴이다.

## 7. 이미 만나본 OTel

- **.NET Aspire** — Aspire 대시보드가 받는 트레이스/메트릭/로그가 전부 OTLP다. `AddServiceDefaults()`가 OTel SDK 설정을 켜는 것이고, 대시보드는 OTLP 수신 백엔드다
- **Langfuse** — Python SDK v3가 OTel 기반, OTLP 수신 엔드포인트 제공
- **vLLM** — `/metrics`는 Prometheus 형식이지만, OTel 트레이싱 연동도 지원 (요청 단위 trace)

즉 "Aspire로 서비스 관측, Langfuse로 LLM 관측"을 해도 **밑바닥 표준은 하나**다. 언젠가 둘을 잇고 싶으면(BFF 트레이스 안에 LLM generation까지) 같은 trace context를 전파하면 된다.

## 8. 핵심 용어

| 용어                | 의미                                                       |
| ------------------- | ---------------------------------------------------------- |
| OTLP                | OTel 표준 전송 프로토콜 (gRPC/HTTP)                        |
| Resource            | 신호의 출처 정보 (`service.name`, 호스트, 버전)            |
| Context Propagation | 경계를 넘어 trace_id를 전달 (`traceparent` 헤더)           |
| Sampling            | 트레이스 일부만 수집 (head/tail sampling)                  |
| Instrumentation     | 계측 — 코드에서 신호를 생성하게 만드는 행위                |

## 9. 학습 리소스

- 공식 문서: https://opentelemetry.io/docs/
- 개념 가이드: https://opentelemetry.io/docs/concepts/
- GenAI Semantic Conventions: https://opentelemetry.io/docs/specs/semconv/gen-ai/
- Langfuse OTel 연동: https://langfuse.com/integrations/native/opentelemetry
