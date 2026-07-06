# Infra

LLM 워크로드의 기반이 되는 GPU·컨테이너 인프라를 다룹니다.

## 노트

- [서버 GPU Docker로 활용 가이드](./notes/서버%20GPU%20Docker로%20활용%20가이드.md) — NVIDIA 드라이버, nvidia-container-toolkit 설치부터 GPU 컨테이너 검증까지

## 앞으로 다룰 주제

- [ ] GPU 모니터링 — DCGM Exporter + Prometheus
- [ ] 한 GPU를 여러 컨테이너가 공유하는 전략 (메모리 분할, MPS)
- [ ] Kubernetes GPU 스케줄링 — device plugin, time-slicing
