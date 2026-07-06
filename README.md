# LLMOps Notes

LLM 서비스를 **운영(Operations)** 관점에서 학습하고 기록하는 레포지토리입니다.

모델·프레임워크 자체(LangChain, RAG, Agent 등)는 [llm-notes](../llm-notes)에서 다루고,
이 레포는 그렇게 만든 LLM 애플리케이션을 **어떻게 띄우고, 관측하고, 평가하고, 운영하는가**에 집중합니다.

---

## 카테고리

### [Serving](./serving/)

모델을 추론 서버로 띄우고 성능을 끌어내는 영역입니다.

- vLLM — PagedAttention, Continuous Batching, 양자화, 배포 패턴
- Ollama — 로컬 모델 서버, Open WebUI 연동

### [Infra](./infra/)

LLM 워크로드의 기반 인프라를 다룹니다.

- GPU 서버 구성 — NVIDIA 드라이버, nvidia-container-toolkit, Docker GPU 활용

### [Gateway](./gateway/)

여러 모델·프로바이더를 단일 엔드포인트로 묶는 라우팅 계층입니다.

- LiteLLM — OpenAI 호환 프록시, 폴백, 레이트리밋, 비용 추적

### [Observability](./observability/)

LLM 호출 체인 단위의 트레이싱과 모니터링입니다.

- Langfuse — 셀프호스팅 LLM 트레이싱, 토큰/비용/지연 추적
- vLLM Prometheus 메트릭 — TTFT, TPOT, KV 캐시 사용률

### [Evaluation](./evaluation/)

프롬프트·모델 변경에 대한 품질 회귀 테스트입니다.

- LLM-as-Judge, RAG 평가 지표
- Ragas, promptfoo, DeepEval

---

## 디렉토리 구조

```
llmops-notes/
├── serving/          # 추론 엔진 (vLLM, Ollama)
│   ├── notes/
│   └── experiments/
├── infra/            # GPU, 컨테이너 기반 인프라
│   └── notes/
├── gateway/          # 모델 라우팅 (LiteLLM)
│   └── notes/
├── observability/    # 트레이싱, 메트릭 (Langfuse)
│   └── notes/
├── evaluation/       # 품질 평가, 회귀 테스트
│   └── notes/
└── README.md
```

각 카테고리는 `notes/`(개념 정리)와 필요 시 `experiments/`(compose 스택, 실습 코드)로 구성합니다.
