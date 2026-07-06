# 서버 GPU를 Docker로 활용하기 위한 가이드 문서

## 목적
이 문서는 서버에 장착된 NVIDIA GPU를 Docker 컨테이너에서 효율적으로 활용하기 위한 설정 및 준비 과정을 단계별로 안내한다. GPU는 AI, 머신러닝, 고성능 컴퓨팅(HPC) 등에서 필수적인 자원으로, Docker를 통해 이를 안전하고 재현 가능한 환경에서 사용 가능하게 한다.

## 1. 하드웨어 점검 및 준비
### 1.1. GPU 확인
- **과정**: `nvidia-smi` 명령어를 실행하여 GPU 모델, 수량, 메모리 상태 확인.
- **목적**: 서버에 설치된 GPU가 정상 작동하는지, Docker에서 사용할 수 있는지 초기 점검.
- **이유**: GPU가 인식되지 않으면 컨테이너 내에서 GPU 활용이 불가능하며, 이후 작업이 무의미해짐.

### 1.2. 부하 테스트
- **과정**: `gpu-burn` 또는 유사 툴로 GPU에 최대 부하를 1~2시간 적용, `nvtop` 또는 `watch -n 1 -d nvidia-smi`로 온도 및 팬 속도 모니터링(목표: 온도 85도 미만, 팬 50% 내외 유지).
- **목적**: 장시간 GPU 사용 시 안정성 및 과열 여부 확인.
- **이유**: GPU는 연속 작업에서 성능 저하나 손상을 입을 수 있으며, 사전에 이를 방지하기 위함.

## 2. 소프트웨어 설치 및 설정
### 2.1. NVIDIA 드라이버 설치
- **과정**: `sudo ubuntu-drivers devices`로 호환 드라이버 확인 후 `sudo apt install nvidia-driver-<버전>`(예: nvidia-driver-570) 실행.
- **목적**: GPU를 운영체제에서 인식하고 제어할 수 있는 환경 구축.
- **이유**: 드라이버가 없으면 GPU 자원에 접근할 수 없어 Docker 내 GPU 활용이 불가능.

### 2.2. Docker 설치
- **과정**: `sudo apt update && sudo apt install docker.io`로 Docker 설치, `sudo systemctl start docker`로 서비스 활성화.
- **목적**: 컨테이너화된 환경을 제공하여 GPU 작업을 격리된 상태로 실행.
- **이유**: Docker는 소프트웨어 의존성 문제를 해결하고, 재현 가능한 환경을 보장함.

### 2.3. NVIDIA Container Toolkit 설치
- **과정**: 
  - `curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit.gpg`.
  - `curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list`.
  - `sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit`.
  - `sudo nvidia-ctk runtime configure --runtime=docker`와 `sudo systemctl restart docker`로 적용.
- **목적**: Docker 컨테이너가 GPU를 인식하고 활용할 수 있게 설정.
- **이유**: 기본 Docker는 GPU 지원을 하지 않으므로, NVIDIA Container Toolkit이 필수.

## 3. Docker 컨테이너 환경 구성
### 3.1. GPU 테스트 컨테이너 실행
- **과정**: `docker run --rm --gpus all nvidia/cuda:<버전>-base-ubuntu<버전> nvidia-smi`(예: nvidia/cuda:12.2.0-base-ubuntu22.04) 실행.
- **목적**: 컨테이너 내에서 GPU가 정상 작동하는지 확인.
- **이유**: GPU 연결이 제대로 되었는지 검증하여 이후 작업의 기초를 닦음.

### 3.2. docker-compose를 통한 다중 컨테이너 관리
- **과정**: `docker-compose.yml` 작성 (예: `image: nvidia/cuda:<버전>`, `command: nvidia-smi`, `deploy.resources.reservations.devices` 또는 `runtime: nvidia` 설정) 후 `docker compose up -d`로 실행.
- **목적**: 여러 GPU 모니터링 컨테이너를 쉽게 관리하고 배포.
- **이유**: 복잡한 워크로드에서 다중 GPU를 활용하거나 상태를 실시간으로 추적할 때 유용.

### 3.3. 커스터마이징된 이미지 생성
- **과정**: Dockerfile에 베이스 이미지(`nvidia/cuda:<버전>`)를 기반으로 Python, PyTorch 등 추가 라이브러리 설치 후 `docker build`로 이미지 생성.
- **목적**: 특정 워크로드(예: AI 모델 훈련/추론)에 맞춘 환경 구축.
- **이유**: 기본 이미지로는 작업에 필요한 소프트웨어가 부족할 수 있어 커스터마이징 필요.

## 4. 검증 및 유지보수
### 4.1. GPU 활용 검증
- **과정**: 커스터마이징된 컨테이너로 간단한 GPU 연산(예: 행렬 곱셈) 테스트.
- **목적**: 실제 작업 환경에서 GPU가 올바르게 작동하는지 최종 확인.
- **이유**: 모니터링 외에 실질적인 연산이 가능한지 검증해야 함.

### 4.2. 정기 점검
- **과정**: 드라이버, CUDA, Docker 버전 업데이트, 부하 테스트 재실시.
- **목적**: 최신 성능 및 보안 패치 적용.
- **이유**: 오래된 소프트웨어는 호환성 문제나 성능 저하를 초래할 수 있음.

## 결론
서버 GPU를 Docker로 활용하려면 하드웨어 점검(1), 소프트웨어 설치(2), 컨테이너 환경 구성(3), 검증/유지보수(4) 단계를 거쳐야 한다. 이 과정을 통해 GPU를 안전하고 효율적으로 활용할 수 있는 환경이 구축되며, AI/ML이나 HPC와 같은 고성능 작업에 적합한 기반이 마련된다. 각 단계는 GPU 활용의 안정성과 재현성을 보장하기 위해 필수적이다.