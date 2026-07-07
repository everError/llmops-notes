# Langfuse 실습

Langfuse 셀프호스팅 + Python SDK 트레이싱 실습 환경입니다.

## 구성

```
langfuse/
├── docker-compose.yml   # 공식 compose (langfuse-web/worker, Postgres, ClickHouse, Redis, MinIO)
├── .env.example         # 시크릿·초기 시딩 템플릿 → .env 로 복사해서 사용
├── environment.yml      # mamba 환경 정의 (langfuse-lab)
└── notebooks/
    └── 01_tracing_basics.ipynb
```

## 셋업 순서

### 1. 시크릿 준비

```bash
cp .env.example .env
# .env 의 CHANGEME 값 채우기
# Git Bash (openssl 번들 포함):
openssl rand -base64 32   # SALT, NEXTAUTH_SECRET
openssl rand -hex 32      # ENCRYPTION_KEY (hex 64자 필수)
# PowerShell 대안:
#   $b = New-Object byte[] 32; (New-Object Security.Cryptography.RNGCryptoServiceProvider).GetBytes($b)
#   [Convert]::ToBase64String($b)                            # base64
#   -join ($b | ForEach-Object { $_.ToString('x2') })        # hex
# Python 대안: python -c "import secrets; print(secrets.token_hex(32))"
```

`.env` 의 `LANGFUSE_INIT_*` 값은 첫 기동 시 조직/프로젝트/로그인 계정/API 키를 자동 생성한다.
덕분에 UI에서 수동으로 프로젝트를 만들 필요 없이 노트북에서 바로 같은 키로 연결된다.

### 2. Langfuse 스택 기동

```bash
docker compose up -d
docker compose ps            # 전부 healthy 확인 (첫 기동은 마이그레이션 때문에 1~2분 소요)
```

http://localhost:3000 접속 → `.env` 의 `LANGFUSE_INIT_USER_EMAIL` / `PASSWORD` 로 로그인.

### 3. mamba 환경 생성

```bash
mamba env create -f environment.yml
mamba activate langfuse-lab
jupyter lab
```

`notebooks/01_tracing_basics.ipynb` 부터 진행.

## Windows(Docker Desktop) 주의사항

- **볼륨은 named volume 그대로 둘 것** — Windows 경로 바인드 마운트로 바꾸면 ClickHouse/Postgres가 NTFS에서 권한·성능 문제를 일으킨다.
- **메모리** — 스택 전체가 4GB 정도 사용. WSL2 상한은 `~/.wslconfig` 의 `memory=` 로 조절.
- **포트 3000 충돌 시** — `docker-compose.yml` 의 langfuse-web `ports` 를 `3001:3000` 등으로 변경하고, `.env` 의 `NEXTAUTH_URL` 과 `LANGFUSE_HOST` 도 함께 수정.

## 정리(완전 삭제)

```bash
docker compose down -v   # 볼륨까지 삭제
```
