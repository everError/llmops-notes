# Observability

LLM 서비스의 관측성을 다룹니다. 일반 APM과 달리 **LLM 호출 체인(trace) 단위**로 프롬프트·응답·토큰·비용·지연을 추적하는 것이 핵심입니다.

## 다룰 도구

- **Langfuse** — 오픈소스 LLM 트레이싱 플랫폼. docker compose로 셀프호스팅 가능, LangChain 콜백 연동
- **vLLM `/metrics`** — Prometheus 형식 서빙 메트릭 (TTFT, TPOT, KV 캐시 사용률, 대기 큐)

## 핵심 지표

| 지표                | 의미                                    |
| ------------------- | --------------------------------------- |
| TTFT                | Time To First Token — 체감 응답 속도    |
| TPOT                | Time Per Output Token — 생성 속도       |
| 토큰 사용량         | 요청별 입력/출력 토큰 — 비용의 원천     |
| trace               | 한 요청이 거친 체인·도구 호출 전체 기록 |

## 앞으로 다룰 주제

- [ ] Langfuse 셀프호스팅 구축
- [ ] [llm-chat](../../llm-chat)에 LangChain 콜백으로 트레이싱 연동
- [ ] 세션/유저 단위 추적, 프롬프트별 비용 분석
- [ ] vLLM 메트릭 대시보드 구성
