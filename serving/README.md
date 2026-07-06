# Serving

LLM을 추론 서버로 띄우고 처리량·지연·메모리를 튜닝하는 영역입니다.

## 노트

- [vLLM 정리](./notes/vllm.md) — PagedAttention, Continuous Batching, CLI 옵션, 양자화, 배포 패턴, 트러블슈팅
- [Ollama + Open WebUI Docker Compose](./notes/Ollama%20+%20Open%20WebUI%20Docker%20Compose.md) — 로컬 LLM 서버 + 웹 UI 구성 가이드

## 엔진 선택 기준

| 엔진            | 적합한 상황                          |
| --------------- | ------------------------------------ |
| **vLLM**        | 다중 사용자, 고부하, 프로덕션 서빙   |
| **Ollama**      | 개인 실험, 모델 핫스왑, 간단한 배포  |
| **llama.cpp**   | CPU·엣지, 저자원 환경                |
| **TGI**         | HuggingFace 생태계 통합              |

## 앞으로 다룰 주제

- [ ] 양자화 형식별(AWQ, GPTQ, FP8) 실측 성능·품질 비교
- [ ] vLLM 배칭 파라미터 튜닝 실습 (`max-num-seqs`, `max-num-batched-tokens`)
- [ ] 멀티 GPU — Tensor Parallel 실습
- [ ] Kubernetes 위 서빙 — KServe, 오토스케일링
