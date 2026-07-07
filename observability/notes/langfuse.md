# Langfuse 정리 — LLM Observability 기초

## 1. Langfuse란

오픈소스 **LLM 엔지니어링 플랫폼**. 핵심은 LLM 애플리케이션의 **트레이싱(tracing)** 이고, 그 위에 프롬프트 관리·평가·데이터셋 기능이 얹혀 있다.

- **오픈소스 + 셀프호스팅 가능** (MIT 라이선스 코어, docker compose 배포)
- **프레임워크 중립** — LangChain, LlamaIndex, OpenAI SDK, 순수 Python 모두 지원
- **OpenTelemetry 기반** (Python SDK v3부터, 현재 v4) — OTel 생태계와 호환

> 버전 주의: **서버(docker image)는 v3, Python SDK는 v4**로 따로 versioning 된다. SDK v4에서 span/generation 생성 API가 `start_as_current_observation(as_type=...)` 하나로 통합되었다 — 웹 예제 중 v3 문법(`start_as_current_span` 등)은 그대로 동작하지 않음.

대안 비교:

| 도구             | 성격                          | 셀프호스팅         |
| ---------------- | ----------------------------- | ------------------ |
| **Langfuse**     | 오픈소스, 범용                | ✅ 무료            |
| **LangSmith**    | LangChain 공식, SaaS 중심     | 유료 플랜만        |
| **Phoenix**      | 오픈소스, 평가·실험에 강점    | ✅                 |
| **Helicone**     | 프록시 방식 (게이트웨이 겸용) | ✅                 |

## 2. 왜 일반 APM으로는 부족한가

일반 APM(Datadog, Aspire 대시보드 등)은 **HTTP 요청 단위**로 본다. LLM 서비스에서 알고 싶은 것은 다르다:

- 이 응답이 이상한데, **정확히 어떤 프롬프트**가 들어갔나?
- RAG 체인에서 **검색 → 재순위 → 생성** 중 어느 단계가 느리고 비싼가?
- 모델/프롬프트를 바꾼 뒤 **품질·비용·지연**이 어떻게 변했나?
- 이 유저의 **세션 전체 대화 흐름**은 어땠나?

즉 관측 단위가 "요청"이 아니라 **"LLM 호출 체인 + 그 내용물(프롬프트/응답/토큰)"** 이어야 하고, 이것이 LLM Observability가 별도 도구로 존재하는 이유다.

## 3. 데이터 모델 (핵심)

```
Session (선택)                  ← 대화 세션 단위 묶음
└── Trace                       ← 요청 1건의 전체 기록 (user_id, tags, input/output)
    └── Observation (트리 구조)
        ├── Span                ← 일반 작업 단위 (전처리, 검색, 후처리)
        ├── Generation          ← LLM 호출 전용 (model, 프롬프트, 응답, 토큰 usage, cost)
        └── Event               ← 시점성 이벤트
```

| 개념           | 설명                                                                 |
| -------------- | -------------------------------------------------------------------- |
| **Trace**      | 사용자 요청 1건. 트리의 루트. input/output, metadata, tags 부착      |
| **Span**       | 시간 구간이 있는 작업. 중첩 가능                                     |
| **Generation** | Span의 특수형. `model`, `usage`(토큰), `cost`가 있어 비용 집계의 근원 |
| **Session**    | 여러 trace를 대화 단위로 묶음 (챗봇의 한 대화창)                     |
| **User**       | trace에 `user_id`를 달아 유저별 사용량·비용 추적                     |
| **Score**      | trace/observation에 붙는 평가 점수 (유저 피드백, LLM-as-Judge 결과)  |

> **Generation에 usage를 기록해야 비용이 나온다.** 통합 SDK(LangChain 콜백 등)를 쓰면 자동 기록되고, 수동 계측이면 `usage_details`를 직접 넣어야 한다. 모델별 단가는 Langfuse가 내장 가격표 + 커스텀 모델 정의로 계산한다.

## 4. 계측(Instrumentation) 방법

낮은 침습 → 높은 제어 순:

1. **프레임워크 통합** — LangChain `CallbackHandler`, LlamaIndex, OpenAI SDK 래퍼. 코드 몇 줄로 체인 전체 자동 트레이스
2. **`@observe` 데코레이터** — 함수에 붙이면 자동으로 trace/span 생성, 중첩 호출은 부모-자식으로 연결
3. **Low-level SDK** — `start_as_current_observation(as_type="span"|"generation"|"tool"|...)`으로 구조를 직접 설계

전송은 **비동기 배치** — SDK가 백그라운드에서 모아 보내므로 앱 지연에 거의 영향 없음. 프로세스 종료 전 `flush()` 필요.

## 5. 트레이싱 위에 얹히는 기능

- **Prompt Management** — 프롬프트를 코드에서 분리해 버전 관리, 배포 없이 교체, 트레이스와 자동 연결 (어떤 버전이 어떤 결과를 냈는지)
- **Datasets & Experiments** — 실제 트레이스에서 테스트 케이스를 추출해 데이터셋 구축 → 프롬프트/모델 변경 시 일괄 실행·비교
- **Evaluation / Scores** — 유저 피드백(👍👎), 수동 라벨링, LLM-as-Judge 결과를 score로 축적
- **대시보드** — 토큰/비용/지연(TTFT 포함) 추이, 모델별·유저별·태그별 분해

→ 학습 순서상 트레이싱을 먼저 익히는 이유: **나머지 기능이 전부 트레이스 데이터를 원료로 삼는다.**

## 6. 셀프호스팅 아키텍처 (v3)

```
SDK ──HTTP──▶ langfuse-web (Next.js, UI + API)
                  │
                  ├─ Postgres    ← 트랜잭션 데이터 (유저, 프로젝트, 프롬프트)
                  ├─ Redis       ← 인제스트 큐, 캐시
                  ├─ S3/MinIO    ← 원본 이벤트, 대용량 input/output
                  └─ ClickHouse  ← 트레이스 분석 저장소 (OLAP)
                  ▲
              langfuse-worker    ← 큐 소비, ClickHouse 적재
```

- v2까지는 Postgres 단일 저장소였으나, v3에서 **쓰기(큐) / 분석(ClickHouse) / 원본(S3) 분리** — 대량 트레이스 처리를 위한 구조
- 인제스트가 비동기라 트레이스가 UI에 뜨기까지 수 초 지연이 정상
- db-notes에서 다룬 columnar(ClickHouse가 대표적) 개념이 여기서 실물로 등장 — "이벤트 대량 적재 + 집계 쿼리" 워크로드에 columnar를 쓰는 전형적 사례

## 7. 핵심 용어

| 용어     | 의미                                                          |
| -------- | ------------------------------------------------------------- |
| TTFT     | Time To First Token — 첫 토큰까지 시간, 체감 응답성           |
| Latency  | 전체 생성 완료까지 시간                                       |
| Usage    | input/output 토큰 수 — 비용 계산의 원천                       |
| Metadata | trace/observation에 붙이는 임의 컨텍스트 (환경, 버전, 파라미터) |
| Tags     | trace 필터링용 라벨                                           |

## 8. 학습 리소스

- 공식 문서: https://langfuse.com/docs
- 셀프호스팅 가이드: https://langfuse.com/self-hosting
- 데이터 모델: https://langfuse.com/docs/observability/data-model
- GitHub: https://github.com/langfuse/langfuse
