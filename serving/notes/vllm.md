# vLLM 정리 — 배포 관점 학습 자료

## 1. vLLM이란

UC Berkeley에서 2023년 발표한 **LLM 추론·서빙 엔진**. 핵심 가치:

- **고성능**: HF Transformers 대비 10~30배 처리량
- **OpenAI 호환 API**: ChatGPT API 그대로 호환
- **메모리 효율**: PagedAttention으로 KV 캐시 단편화 최소화
- **광범위한 모델 지원**: 200+ 아키텍처

대안 비교:

| 솔루션              | 특징                   | 적합                |
| ------------------- | ---------------------- | ------------------- |
| **vLLM**            | 최고 처리량, 프로덕션  | 다중 사용자, 고부하 |
| **TGI** (HF)        | vLLM과 유사, HF 생태계 | HF 통합             |
| **Ollama**          | 모델 핫스왑, CLI 친화  | 개인 실험           |
| **llama.cpp**       | CPU·엣지 최적화        | 저자원              |
| **HF Transformers** | 가장 광범위, 느림      | 연구                |

## 2. 핵심 개념

### PagedAttention

KV 캐시(어텐션 키·값 저장소)를 **OS 가상 메모리처럼 페이지 단위로 관리**. 기존엔 요청마다 max_len만큼 예약 → 짧은 요청도 큰 슬롯 차지. PagedAttention은 블록(보통 16토큰) 단위 동적 할당으로 메모리 낭비 ~90% 감소.

### Continuous Batching (in-flight batching)

요청들이 서로 다른 길이로 진행 중이어도 **매 step마다 배치 재구성**. 빈 슬롯 즉시 채움. 처리량 대폭 증가.

### V0 vs V1 Engine

v0.6+ 부터 **V1 엔진** 기본. 단순화된 코드, CUDA graphs 기본 사용, 비동기 스케줄링, 빠른 prefill. V0는 deprecated.

### Prefill vs Decode

- **Prefill**: 입력 프롬프트 전체를 한 번에 forward (병렬, 빠름)
- **Decode**: 한 토큰씩 생성 (직렬, 느림, KV 캐시 사용)

이 둘이 한 GPU에서 경쟁하는데 vLLM은 **chunked prefill**로 둘을 섞어서 처리량 극대화.

## 3. 설치

**pip/uv (네이티브)**

```bash
uv venv && source .venv/bin/activate
uv pip install vllm
# nightly:
uv pip install -U vllm --pre --extra-index-url https://wheels.vllm.ai/nightly/cu129
```

**Docker (프로덕션 권장)**

```bash
docker pull vllm/vllm-openai:latest        # CUDA 12.9
docker pull vllm/vllm-openai:latest-cu130  # CUDA 13.0
docker pull vllm/vllm-openai:gemma4        # 특정 모델 전용 빌드 (있는 경우)
docker pull vllm/vllm-openai:v0.20.2       # 버전 핀
```

## 4. 기본 실행

```bash
vllm serve <model>
```

`<model>` 종류:

- HF 경로: `meta-llama/Llama-3-8B-Instruct`
- 로컬 경로: `/data/models/my-model`
- ModelScope: `VLLM_USE_MODELSCOPE=1 vllm serve <model>`

Docker:

```bash
docker run --gpus all -p 8000:8000 \
  -v ~/.cache/huggingface:/root/.cache/huggingface \
  --ipc=host \
  vllm/vllm-openai:latest \
  <model> [옵션들...]
```

## 5. CLI 옵션 카테고리별

### 5.1 모델·로딩

| 옵션                                 | 설명                       |
| ------------------------------------ | -------------------------- |
| `<positional>`                       | 모델 경로 (첫 인자)        |
| `--served-model-name X`              | API에서 노출될 이름 (별칭) |
| `--task generate\|embed\|score`      | 모델 용도 (보통 자동)      |
| `--runner pooling\|generate`         | 러너 (임베딩은 `pooling`)  |
| `--trust-remote-code`                | HF 커스텀 코드 실행 허용   |
| `--dtype auto\|bfloat16\|float16`    | 활성화 dtype               |
| `--quantization awq\|gptq\|fp8\|...` | 양자화 (보통 자동감지)     |
| `--revision X`                       | 특정 git revision          |

### 5.2 메모리

| 옵션                               | 설명                                |
| ---------------------------------- | ----------------------------------- |
| `--gpu-memory-utilization 0.0~1.0` | GPU 메모리 점유 비율 (기본 0.9)     |
| `--max-model-len N`                | 최대 컨텍스트 길이                  |
| `--kv-cache-dtype fp16\|bf16\|fp8` | KV 캐시 정밀도 (fp8로 KV 절반 절약) |
| `--swap-space N`                   | CPU 스왑 (GB)                       |
| `--cpu-offload-gb N`               | CPU 오프로드 가중치 (GB)            |
| `--block-size N`                   | KV 블록 크기 (기본 16)              |

`--gpu-memory-utilization`은 **총 메모리 기준 비율**. 모델 가중치 + 활성화 + CUDA graph + KV 캐시 모두 이 한도 안에 들어가야 함.

### 5.3 배치·스케줄링

| 옵션                         | 설명                      |
| ---------------------------- | ------------------------- |
| `--max-num-seqs N`           | 동시 처리 요청 수         |
| `--max-num-batched-tokens N` | 한 forward pass 토큰 한도 |
| `--enable-chunked-prefill`   | 긴 prefill 분할 (V1 기본) |
| `--enable-prefix-caching`    | 공통 prefix KV 재사용     |
| `--async-scheduling`         | 비동기 스케줄링 (처리량↑) |

### 5.4 병렬화

| 옵션                              | 설명                                      |
| --------------------------------- | ----------------------------------------- |
| `--tensor-parallel-size N` (TP)   | 모델을 N개 GPU에 가로로 분할 (한 노드 내) |
| `--pipeline-parallel-size N` (PP) | 레이어 단위 분할 (다중 노드)              |
| `--data-parallel-size N` (DP)     | 동일 모델 복제 (처리량↑)                  |

규칙: **한 노드 내 → TP**, 노드 간 → PP+TP. TP는 GPU 수가 어텐션 헤드 수의 약수여야 함.

### 5.5 서빙·네트워크

| 옵션                | 설명             |
| ------------------- | ---------------- |
| `--host 0.0.0.0`    | 바인드 IP        |
| `--port 8000`       | 포트             |
| `--api-key X`       | API 키 인증 강제 |
| `--allowed-origins` | CORS 출처        |

### 5.6 멀티모달

| 옵션                                            | 설명                             |
| ----------------------------------------------- | -------------------------------- |
| `--limit-mm-per-prompt '{"image":4,"video":1}'` | 요청당 미디어 한도               |
| `--mm-processor-kwargs '{...}'`                 | 프로세서 옵션 (이미지 해상도 등) |
| `--disable-mm-preprocessor-cache`               | 캐시 끄기                        |

### 5.7 Reasoning · Tool Calling

| 옵션                         | 설명                                                               |
| ---------------------------- | ------------------------------------------------------------------ |
| `--enable-auto-tool-choice`  | 자동 tool 선택 활성화                                              |
| `--tool-call-parser X`       | 모델별 tool 출력 파서 (gemma4, hermes, llama3_json, openai 등)     |
| `--reasoning-parser X`       | reasoning 출력 파서 (gemma4, qwen3, deepseek_r1, openai_gptoss 등) |
| `--chat-template path.jinja` | 커스텀 chat 템플릿                                                 |

### 5.8 성능·디버깅

| 옵션                             | 설명                                   |
| -------------------------------- | -------------------------------------- |
| `--enforce-eager`                | CUDA graph 비활성 (디버그·메모리 절약) |
| `--compilation-config '{...}'`   | torch.compile 설정                     |
| `--max-cudagraph-capture-size N` | CUDA graph 최대 배치                   |

## 6. OpenAI 호환 API

기본 포트 8000, 표준 OpenAI 형식 그대로:

| 경로                   | 메서드 | 용도                        |
| ---------------------- | ------ | --------------------------- |
| `/v1/chat/completions` | POST   | 대화                        |
| `/v1/completions`      | POST   | 완성 (legacy)               |
| `/v1/embeddings`       | POST   | 임베딩                      |
| `/v1/models`           | GET    | 모델 목록                   |
| `/v1/responses`        | POST   | Responses API (OpenAI 신규) |
| `/v1/messages`         | POST   | Anthropic API 호환          |
| `/health`              | GET    | 헬스체크                    |
| `/metrics`             | GET    | Prometheus                  |

**클라이언트 코드 변경 거의 없음**:

```python
from openai import OpenAI
client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
resp = client.chat.completions.create(model="my-model", messages=[...])
```

## 7. 양자화

`config.json`에서 자동 감지가 기본.

| 형식      | 비트 | 활성화 | 지원 GPU       | 비고                                     |
| --------- | ---- | ------ | -------------- | ---------------------------------------- |
| BF16/FP16 | 16   | 16-bit | 전체           | 양자화 X (기준)                          |
| **AWQ**   | 4    | 16-bit | Pascal+        | 인기, 품질 좋음 (`awq_marlin` 커널 권장) |
| **GPTQ**  | 4    | 16-bit | Pascal+        | AWQ와 유사                               |
| **FP8**   | 8    | 8-bit  | Hopper, Ada+   | 네이티브 텐서코어, 빠름                  |
| **MXFP4** | 4    | 4-bit  | Ada+           | gpt-oss 등                               |
| **NVFP4** | 4    | 4-bit  | Blackwell only | RTX 5090, B200                           |
| **INT8**  | 8    | 8-bit  | 전체           | 구식                                     |

GPU별 호환:

- RTX 3090/A100 (sm_8x): BF16, AWQ, GPTQ
- RTX 4090, RTX 6000 Ada (sm_89): + **FP8, MXFP4**
- H100 (sm_90): + FP8 네이티브 가속
- RTX 5090, B200 (sm_100+): + NVFP4

## 8. 흔한 문제와 해결

### OOM (가장 흔함)

증상: `No available memory for the cache blocks` 또는 `CUDA out of memory`

대처 (위에서 아래 순서로 시도):

1. `--gpu-memory-utilization` 증가 (다른 프로세스 없으면 0.9까지)
2. `--max-model-len` 감소
3. `--max-num-seqs` 감소
4. `--max-num-batched-tokens` 감소
5. `--kv-cache-dtype=fp8` (KV 캐시 절반)
6. 양자화 적용
7. `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True` 환경변수

### 모델 로딩이 멈춘 듯

대부분 **다운로드 중**:

```bash
du -sh ~/.cache/huggingface/hub/models--*/
```

크기 늘면 정상. HF_TOKEN 설정하면 rate limit 완화.

### 토크나이저 에러

`--tokenizer X` 명시. 또는 `--tokenizer-mode slow` (huggingface tokenizers 우회).

### 멀티모달 출력 깨짐

- chat template 누락
- dtype 문제 (Gemma 4: bf16 강제 필요)
- 특정 모델은 vLLM 전용 빌드 필요 (예: `vllm/vllm-openai:gemma4`)

### CUDA driver 불일치

컨테이너 CUDA 런타임 < 호스트 드라이버 지원 한도여야 함. nvidia-smi에 표시되는 CUDA 버전이 상한.

## 9. 배포 패턴

### 패턴 A — 단일 모델 단일 컨테이너 (기본)

가장 단순. 한 모델·한 GPU.

### 패턴 B — 다중 모델 다중 컨테이너 (한 GPU 공유)

같은 GPU에 여러 vLLM 컨테이너. 각자 `--gpu-memory-utilization` 비율로 메모리 분할.

- 장점: 모델별 독립, 다른 vLLM 버전 가능
- 단점: 자원 분할 손해

### 패턴 C — LoRA 다중 어댑터

베이스 모델 1개 + LoRA 어댑터 여러 개 동적 스왑:

```bash
vllm serve <base> --enable-lora --max-loras=4 \
  --lora-modules adapter1=/path1 adapter2=/path2
```

### 패턴 D — Tensor Parallel

한 큰 모델을 여러 GPU에 분할:

```bash
vllm serve llama-70b --tensor-parallel-size=4
```

### 패턴 E — Router 앞단

여러 vLLM 인스턴스 위에 LiteLLM / vLLM Router로 통합 엔드포인트. 모델 라우팅·로드밸런싱.

## 10. 모니터링·운영

**Prometheus 메트릭** (`/metrics`):

- `vllm:num_requests_running` — 처리 중
- `vllm:num_requests_waiting` — 대기 큐 길이
- `vllm:gpu_cache_usage_perc` — KV 캐시 사용률
- `vllm:time_to_first_token_seconds` — TTFT
- `vllm:time_per_output_token_seconds` — TPOT
- `vllm:request_success_total` — 성공 카운트

**로그 레벨**:

```bash
export VLLM_LOGGING_LEVEL=DEBUG
```

**헬스체크**: `curl http://localhost:8000/health` → 200이면 정상.

## 11. 모델·기능 호환성 확인 순서

새 모델 띄울 때 권장 절차:

1. **공식 레시피 확인**: `docs.vllm.ai/projects/recipes` 에서 모델별 가이드
2. **지원 모델 목록**: `docs.vllm.ai/en/latest/models/supported_models/`
3. **vLLM 버전 요구사항** 확인 (예: Gemma 4는 v0.19+)
4. **HF 모델 카드의 `config.json`** 확인: `architectures` 필드가 vLLM 지원 목록에 있어야 함
5. **양자화 호환 GPU** 확인 (FP8은 Ada+, NVFP4는 Blackwell만)
6. **모델 전용 도커 태그** 우선 확인 (있으면 그것 사용)

## 12. 버전 선택 가이드

| 시나리오               | 권장                                 |
| ---------------------- | ------------------------------------ |
| 안정 프로덕션          | `:vX.Y.Z` 핀, 최신에서 1~2 마이너 뒤 |
| 최신 기능              | `:latest`                            |
| 특정 모델 (Gemma 4 등) | 모델 전용 태그 우선 (`:gemma4` 등)   |
| 연구·실험              | nightly wheel                        |
| Mac, CPU, 엣지         | vLLM 부적합 → llama.cpp              |

## 13. 학습 리소스

- **공식 문서**: https://docs.vllm.ai
- **모델별 레시피**: https://docs.vllm.ai/projects/recipes
- **GitHub**: https://github.com/vllm-project/vllm
- **논문**: _"Efficient Memory Management for Large Language Model Serving with PagedAttention"_ (SOSP 2023)
- **공식 Discord**: vLLM Slack/Discord 채널

---
