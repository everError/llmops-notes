# Ollama + Open WebUI Docker Compose 설정 가이드

## 📋 개요
이 Docker Compose 파일은 Ollama (로컬 LLM 서버)와 Open WebUI (웹 인터페이스)를 함께 실행하는 구성입니다.

## 🎮 시스템 요구사항

### 필수 사항

#### 1. 하드웨어
- **NVIDIA GPU** (GeForce, Tesla, Quadro 등)

#### 2. 소프트웨어
Docker Compose 파일에서 GPU를 사용하려면 다음이 **모두** 필요합니다:

1. **NVIDIA 드라이버** (450.x 이상 권장)
2. **Docker** (20.10 이상 권장)
3. **Docker Compose** (v2.x 권장)
4. **nvidia-container-toolkit** (nvidia-docker2)

### 설치 확인 방법

```bash
# 1. GPU 확인
lspci | grep -i nvidia

# 2. NVIDIA 드라이버 확인
nvidia-smi

# 출력 예시:
# +-----------------------------------------------------------------------------+
# | NVIDIA-SMI 535.54.03    Driver Version: 535.54.03    CUDA Version: 12.2   |
# +-----------------------------------------------------------------------------+

# 3. Docker 확인
docker --version
# 출력 예시: Docker version 24.0.5, build ced0996

# 4. Docker Compose 확인
docker compose version
# 출력 예시: Docker Compose version v2.20.2

# 5. nvidia-container-toolkit 확인 (가장 중요!)
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
# nvidia-smi 출력이 나오면 정상
```

**모든 명령어가 정상 작동하면 사용 준비 완료입니다!**

### 필수 구성 요소 설치 (없는 경우)

#### NVIDIA 드라이버 설치 (Ubuntu 예시)
```bash
# 추천 드라이버 확인
ubuntu-drivers devices

# 드라이버 자동 설치
sudo ubuntu-drivers autoinstall

# 또는 특정 버전 설치
sudo apt install nvidia-driver-535

# 재부팅
sudo reboot

# 설치 확인
nvidia-smi
```

#### Docker 설치
```bash
# Docker 공식 설치 스크립트
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 재로그인 또는
newgrp docker

# 확인
docker --version
```

#### nvidia-container-toolkit 설치
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

# Docker 재시작
sudo systemctl restart docker

# 확인
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

## 🏗️ 서비스 구성

### 1. Ollama 서비스
로컬에서 LLM 모델을 실행하는 서버입니다.

**주요 설정:**
```yaml
services:
  ollama:
    image: ollama/ollama
    deploy:
      resources:
        reservations:
          devices:
          - driver: nvidia
            capabilities: ["gpu"]
            count: all
    ports:
      - "11434:11434"
    volumes:
      - /data/docker-data/ollama/data/ollama:/root/.ollama
    restart: always
```

**설정 설명:**
- **이미지**: `ollama/ollama` (CUDA 런타임 내장)
- **GPU 할당**: NVIDIA GPU 전체 사용 (`count: all`)
  - `count: all` - 사용 가능한 모든 GPU 할당
  - `count: 1` - GPU 1개만 할당
  - `device_ids: ['0']` - 특정 GPU 선택 (0번)
- **포트**: `11434` (Ollama API 기본 포트)
- **볼륨**: `/data/docker-data/ollama/data/ollama:/root/.ollama`
  - 모델 파일과 설정이 저장되는 영구 스토리지
  - **주의**: 호스트에 해당 디렉토리가 미리 존재해야 함
- **재시작 정책**: `always` (컨테이너 종료 시 항상 재시작)

### 2. Open WebUI 서비스
Ollama를 위한 웹 기반 채팅 인터페이스입니다.

**주요 설정:**
```yaml
openwebui:
  image: ghcr.io/open-webui/open-webui:latest
  restart: unless-stopped
  ports:
    - "3000:8080"
  environment:
    - OLLAMA_BASE_URL=http://ollama:11434
  volumes:
    - ./data/openwebui:/app/backend/data
  depends_on:
    - ollama
```

**설정 설명:**
- **이미지**: `ghcr.io/open-webui/open-webui:latest`
- **포트**: `3000:8080` (호스트 3000 → 컨테이너 8080)
- **접속 URL**: `http://localhost:3000`
- **Ollama 연결**: `OLLAMA_BASE_URL=http://ollama:11434`
  - Docker 네트워크 내부에서 서비스명(`ollama`)으로 연결
- **볼륨**: `./data/openwebui:/app/backend/data`
  - 채팅 기록, 사용자 설정 등 저장
  - 현재 디렉토리 기준 상대 경로
- **의존성**: `depends_on: ollama` (Ollama가 먼저 시작)
- **재시작 정책**: `unless-stopped` (수동으로 중지하지 않는 한 재시작)

## 🚀 사용 방법

### 1. 사전 준비
```bash
# Ollama 데이터 디렉토리 생성
sudo mkdir -p /data/docker-data/ollama/data/ollama
sudo chown -R $USER:$USER /data/docker-data/ollama

# Open WebUI 데이터 디렉토리 생성 (자동 생성되지만 미리 만들어도 됨)
mkdir -p ./data/openwebui
```

### 2. GPU 상태 확인
```bash
# 호스트에서 GPU 확인
nvidia-smi

# Docker에서 GPU 인식 확인
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi
```

### 3. 시작
```bash
# Docker Compose v2 (권장)
docker compose up -d

# Docker Compose v1 (구버전)
docker-compose up -d
```

### 4. 컨테이너 상태 확인
```bash
# 실행 중인 컨테이너 확인
docker compose ps

# 출력 예시:
# NAME                STATUS              PORTS
# ollama              Up 2 minutes        0.0.0.0:11434->11434/tcp
# openwebui           Up 2 minutes        0.0.0.0:3000->8080/tcp
```

### 5. GPU 사용 확인
```bash
# Ollama 컨테이너에서 GPU 확인
docker compose exec ollama nvidia-smi

# 호스트에서 실시간 GPU 모니터링
watch -n 1 nvidia-smi

# GPU 사용률 상세 모니터링
nvidia-smi dmon -s pucvmet
```

### 6. 로그 확인
```bash
# 전체 로그
docker compose logs -f

# Ollama만
docker compose logs -f ollama

# Open WebUI만
docker compose logs -f openwebui

# 최근 100줄만 보기
docker compose logs --tail=100 ollama
```

### 7. 접속
- **Open WebUI**: http://localhost:3000
  - 첫 접속 시 계정 생성 필요
- **Ollama API**: http://localhost:11434
  - API 확인: `curl http://localhost:11434/api/tags`

### 8. 모델 다운로드 및 사용
```bash
# 모델 다운로드 (예: llama2 7B)
docker compose exec ollama ollama pull llama2

# 다른 모델 예시
docker compose exec ollama ollama pull mistral
docker compose exec ollama ollama pull codellama

# 다운로드된 모델 목록 확인
docker compose exec ollama ollama list

# 출력 예시:
# NAME            ID              SIZE      MODIFIED
# llama2:latest   78e26419b446    3.8 GB    2 hours ago

# CLI에서 모델 실행 테스트
docker compose exec ollama ollama run llama2 "안녕하세요"

# 모델 삭제
docker compose exec ollama ollama rm llama2
```

### 9. 중지 및 재시작
```bash
# 중지 (컨테이너 삭제, 데이터는 유지)
docker compose down

# 재시작
docker compose restart

# 특정 서비스만 재시작
docker compose restart ollama
```

### 10. 완전 삭제
```bash
# 컨테이너, 네트워크, 이미지 모두 삭제
docker compose down --rmi all --volumes

# 데이터 디렉토리도 삭제하려면
sudo rm -rf /data/docker-data/ollama
rm -rf ./data/openwebui
```

## 📂 디렉토리 구조
```
프로젝트/
├── docker-compose.yml
└── data/
    └── openwebui/              # Open WebUI 데이터 (자동 생성)
        ├── webui.db            # 사용자 계정, 설정
        └── uploads/            # 업로드된 파일

외부 볼륨:
/data/docker-data/ollama/
└── data/
    └── ollama/
        ├── models/             # 다운로드된 LLM 모델
        └── .ollama/            # Ollama 설정
```

## 🔧 주요 환경 변수 및 옵션

### Ollama GPU 관련 설정

#### 특정 GPU 선택
```yaml
environment:
  # 0번 GPU만 사용
  - NVIDIA_VISIBLE_DEVICES=0
  
  # 여러 GPU 사용
  - NVIDIA_VISIBLE_DEVICES=0,1
  
  # 모든 GPU 사용 (기본값)
  - NVIDIA_VISIBLE_DEVICES=all
```

#### 디버그 모드
```yaml
environment:
  # 상세한 로그 출력 (CUDA 정보, 모델 로딩 과정 등)
  - OLLAMA_DEBUG=2
```

#### 다중 GPU 환경 설정 예시
```yaml
services:
  ollama:
    image: ollama/ollama
    deploy:
      resources:
        reservations:
          devices:
          - driver: nvidia
            device_ids: ['0', '1']  # 0번, 1번 GPU만 사용
            capabilities: ["gpu"]
    environment:
      - NVIDIA_VISIBLE_DEVICES=0,1
```

### Open WebUI 관련 설정

#### Ollama 연결 설정
```yaml
environment:
  # 외부 Ollama 서버 연결
  - OLLAMA_BASE_URL=http://다른서버:11434
  
  # 인증이 필요한 경우
  - OLLAMA_API_KEY=your-api-key
```

#### 포트 변경
```yaml
ports:
  - "8080:8080"  # 포트 8080으로 변경
```

## 💡 성능 최적화 및 팁

### 1. 모델별 GPU 메모리 요구사항
- **7B 모델** (llama2, mistral): 최소 8GB VRAM
- **13B 모델**: 최소 16GB VRAM
- **34B 모델**: 최소 24GB VRAM
- **70B+ 모델**: 최소 48GB VRAM (또는 다중 GPU)

### 2. Quantized 모델 사용
메모리가 부족한 경우 양자화된 모델 사용:
```bash
# 4-bit 양자화 (메모리 1/4)
docker compose exec ollama ollama pull llama2:7b-q4_0

# 8-bit 양자화 (메모리 1/2)
docker compose exec ollama ollama pull llama2:7b-q8_0
```

### 3. GPU 메모리 사용량 모니터링
```bash
# 실시간 GPU 메모리 사용량
watch -n 1 'nvidia-smi --query-gpu=timestamp,name,memory.used,memory.total,utilization.gpu --format=csv'

# 또는
nvidia-smi dmon -s mu
```

### 4. 동시 요청 제한
Open WebUI에서 많은 사용자가 동시 사용 시:
```yaml
environment:
  - OLLAMA_NUM_PARALLEL=2  # 동시 처리 요청 수 제한
```

### 5. 모델 캐싱
```bash
# 모델을 메모리에 유지 (빠른 응답)
# Ollama는 기본적으로 5분간 메모리에 유지
# 설정 변경하려면:
environment:
  - OLLAMA_KEEP_ALIVE=10m  # 10분간 유지
```

## ⚠️ 주의사항

1. **GPU 필수**: Ollama는 GPU 없이도 실행되지만, CPU 추론은 **매우 느립니다** (10-100배 차이)
2. **디스크 공간**: 모델 크기가 크므로 충분한 공간 필요
   - 7B 모델: 약 4-8GB
   - 13B 모델: 약 8-16GB
   - 70B 모델: 약 40-80GB
3. **메모리**: 시스템 RAM도 충분해야 함 (최소 16GB 권장)
4. **NVIDIA 드라이버**: 드라이버 버전이 너무 낮으면 최신 CUDA를 지원하지 못함
5. **방화벽**: 포트 3000과 11434가 열려있어야 함
6. **Docker Compose 버전**: v2 사용 권장 (`docker compose` 명령어)
7. **볼륨 권한**: `/data/docker-data/ollama` 디렉토리에 쓰기 권한 필요

## 🔍 트러블슈팅

### 1. GPU 인식 안 됨
**증상**: `nvidia-smi: command not found` 또는 GPU 사용 안 됨

**해결방법**:
```bash
# nvidia-container-toolkit 확인
docker run --rm --gpus all nvidia/cuda:12.2.0-base-ubuntu22.04 nvidia-smi

# 실패 시 재설치
sudo apt-get install --reinstall nvidia-container-toolkit
sudo systemctl restart docker

# Docker 데몬 설정 확인
sudo cat /etc/docker/daemon.json

# 다음 내용이 있어야 함:
{
  "runtimes": {
    "nvidia": {
      "path": "nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}

# 없다면 추가 후 Docker 재시작
sudo systemctl restart docker
```

### 2. "could not select device driver" 오류
**증상**: 컨테이너 시작 시 GPU 드라이버 오류

**해결방법**:
```bash
# nvidia-container-runtime 설정
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 다시 시도
docker compose up -d
```

### 3. Ollama 컨테이너가 계속 재시작됨
**증상**: `docker compose ps`에서 Restarting 상태

**해결방법**:
```bash
# 로그 확인
docker compose logs ollama

# 볼륨 권한 문제일 가능성
sudo chown -R $USER:$USER /data/docker-data/ollama

# 볼륨 경로가 존재하는지 확인
ls -la /data/docker-data/ollama/data/ollama
```

### 4. Open WebUI에서 Ollama 연결 안 됨
**증상**: "Ollama connection failed" 오류

**해결방법**:
```bash
# Ollama 컨테이너 상태 확인
docker compose ps ollama

# Ollama API 응답 확인
curl http://localhost:11434/api/tags

# Docker 네트워크 내부에서 확인
docker compose exec openwebui curl http://ollama:11434/api/tags

# 실패 시 depends_on 확인 및 재시작
docker compose restart
```

### 5. GPU 메모리 부족 (OOM)
**증상**: "CUDA out of memory" 오류

**해결방법**:
```bash
# 더 작은 모델 사용
docker compose exec ollama ollama pull llama2:7b

# Quantized 모델 사용 (메모리 절약)
docker compose exec ollama ollama pull llama2:7b-q4_0

# 실행 중인 다른 모델 종료
docker compose exec ollama ollama ps
# 사용 안 하는 모델이 메모리에 로드되어 있을 수 있음
```

### 6. 포트 충돌
**증상**: "port is already allocated" 오류

**해결방법**:
```bash
# 사용 중인 프로세스 확인
sudo lsof -i :3000
sudo lsof -i :11434

# 포트 변경 (docker-compose.yml 수정)
ports:
  - "3001:8080"  # 3000 → 3001로 변경
  - "11435:11434"  # 11434 → 11435로 변경
```

### 7. 모델 다운로드 실패
**증상**: "error pulling model" 또는 다운로드 중단

**해결방법**:
```bash
# 네트워크 확인
docker compose exec ollama ping -c 3 ollama.ai

# 디스크 공간 확인
df -h /data/docker-data/ollama

# 다시 시도
docker compose exec ollama ollama pull llama2

# 부분 다운로드된 파일 삭제 후 재시도
docker compose exec ollama rm -rf /root/.ollama/models/blobs/sha256-*
```

### 8. CUDA 버전 불일치
**증상**: CUDA 관련 오류 메시지

**해결방법**:
```bash
# 호스트 CUDA 버전 확인
nvidia-smi | grep "CUDA Version"

# 드라이버가 너무 오래된 경우 업데이트
sudo ubuntu-drivers autoinstall
sudo reboot
```

### 9. 권한 문제
**증상**: Permission denied 오류

**해결방법**:
```bash
# Docker 그룹에 사용자 추가
sudo usermod -aG docker $USER
newgrp docker

# 볼륨 디렉토리 권한 수정
sudo chown -R $USER:$USER /data/docker-data/ollama
sudo chown -R $USER:$USER ./data/openwebui

# Docker 소켓 권한 확인
sudo chmod 666 /var/run/docker.sock
```

### 10. 느린 추론 속도
**증상**: 응답 생성이 너무 느림

**확인사항**:
```bash
# GPU가 실제로 사용되고 있는지 확인
nvidia-smi

# CPU로 실행되고 있을 가능성
docker compose logs ollama | grep -i "cuda\|gpu"

# GPU 사용률이 낮다면
# - 모델이 너무 작을 수 있음
# - 배치 크기 조정 필요
# - 다른 프로세스가 GPU 사용 중일 수 있음
```

## 📚 추가 리소스

- **Ollama 공식 문서**: https://github.com/ollama/ollama
- **Open WebUI 문서**: https://docs.openwebui.com
- **지원 모델 목록**: https://ollama.com/library
- **NVIDIA Container Toolkit**: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/

## 🎓 유용한 명령어 모음

```bash
# === 모델 관리 ===
# 모델 검색
docker compose exec ollama ollama list

# 모델 정보 확인
docker compose exec ollama ollama show llama2

# 실행 중인 모델 확인
docker compose exec ollama ollama ps

# === 시스템 모니터링 ===
# 실시간 GPU 모니터링
watch -n 1 nvidia-smi

# 컨테이너 리소스 사용량
docker stats

# 디스크 사용량
docker system df

# === 로그 및 디버깅 ===
# 에러 로그만 보기
docker compose logs ollama | grep -i error

# 특정 시간 이후 로그
docker compose logs --since 30m ollama

# 실시간 로그 (마지막 50줄)
docker compose logs -f --tail=50

# === 백업 및 복구 ===
# 모델 백업
sudo tar -czf ollama-models-backup.tar.gz /data/docker-data/ollama

# Open WebUI 데이터 백업
tar -czf openwebui-backup.tar.gz ./data/openwebui

# 복구
sudo tar -xzf ollama-models-backup.tar.gz -C /
tar -xzf openwebui-backup.tar.gz
```