# 설치 매뉴얼 — VM 서버 / vLLM 서버

- 작성일: 2026-07-21
- 대상 독자: 서버 구축·재설치를 담당하는 시스템 엔지니어
- 근거 자료: 두 서버(192.168.47.115, 192.168.47.185) SSH 원격 접속 read-only 조회로 확보한 실제 배포 구성 (`docker-compose.yml`, `.env` 등)
- ⚠️ 본 문서의 명령어는 **현재 운영 중인 서버의 실측 구성을 재구성 가능하도록 정리한 것**입니다. `.env` 등 자격증명 파일의 **실제 값은 보안상 본 문서에 기재하지 않으며**, 별도 비밀 관리 절차(사내 볼트/암호화 저장)를 따라야 합니다.

---

## 0. 공통 사전 준비 (두 서버 동일)

| 항목 | 값 |
|---|---|
| OS | Ubuntu 24.04.4 LTS (Server) |
| 설치 방식 | Curtin 기반 표준 Ubuntu Server 설치 (파티션: EFI 1G + /boot 2G ext4 + LVM 나머지) |
| 필수 패키지 | Docker Engine, Docker Compose Plugin |

### 0.1 Docker 설치 (공식 저장소 방식)
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
설치 확인 (2026-07-21 실측 버전): `docker --version` → Docker 29.6.2, `docker compose version` → v5.3.1

### 0.2 운영 계정/디렉터리 구조
```bash
# 서비스 계정 (실측: master, uid=1000, sudo 그룹 소속)
sudo usermod -aG docker master

# 공통 디렉터리 구조 (실측 경로)
sudo mkdir -p /docker/compose /docker/data
```

---

## 1. VM 서버 (192.168.47.115) 설치 절차

### 1.1 최상위 compose 구조
`/docker/compose/docker-compose.yml` — 하위 스택을 include하는 진입점:
```yaml
include:
  - openwebui/docker-compose.yml
  - ragflow/docker-compose.yml
```

### 1.2 OpenWebUI 배포
`/docker/compose/openwebui/docker-compose.yml`:
```yaml
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    environment:
      OPENAI_API_BASE_URL: http://192.168.47.185:8100/v1
      OPENAI_API_KEY: "none"
      DEFAULT_MODELS: Qwen3.5-122B-A10B-FP8
      ENABLE_OLLAMA_API: "false"
    volumes:
      - /docker/data/openwebui:/app/backend/data
```
> vLLM 서버가 먼저 기동되어 8100 포트로 서비스 중이어야 정상 연동됩니다(§2 참고).

```bash
cd /docker/compose
sudo docker compose -f openwebui/docker-compose.yml up -d
```

### 1.3 RAGFlow 배포
디렉터리: `/docker/compose/ragflow/` — 공식 [infiniflow/ragflow](https://github.com/infiniflow/ragflow) 저장소의 `docker/` 배포 자산을 기준으로 구성되어 있으며, `docker-compose.yml`(진입점) + `docker-compose-base.yml`(인프라 서비스) + `.env`(환경설정) + `service_conf.yaml.template` + `entrypoint.sh` + `init.sql` 파일이 필요합니다.

**핵심 환경변수** (`.env`, 실측 기준 — 값은 보안상 생략, 항목만 기재):
| 변수 | 실측 설정값 | 설명 |
|---|---|---|
| `DOC_ENGINE` | elasticsearch | 벡터/문서 검색 엔진 선택 (elasticsearch/infinity/opensearch/oceanbase/seekdb 중 택1) |
| `DEVICE` | cpu | RAGFlow 실행 프로파일 (cpu/gpu) → 컨테이너명이 `ragflow-cpu`로 결정됨 |
| `COMPOSE_PROFILES` | `${DOC_ENGINE},${DEVICE}` = `elasticsearch,cpu` | 활성화할 compose profile 조합 |
| `STACK_VERSION` | 8.11.3 | Elasticsearch 버전 |
| `RAGFLOW_IMAGE` | infiniflow/ragflow:v0.26.4 | RAGFlow 컨테이너 이미지 |
| `MEM_LIMIT` | 8073741824 (약 7.5GiB) | 각 인프라 컨테이너 메모리 상한 |
| `SVR_WEB_HTTP_PORT` / `SVR_WEB_HTTPS_PORT` | 8030 / 443 | 웹 UI 포트 |
| `SVR_HTTP_PORT` / `ADMIN_SVR_HTTP_PORT` | 9380 / 9381 | API / Admin API 포트 |
| `EXPOSE_MYSQL_PORT` | 3306 | MySQL 외부 노출 포트 |
| `ES_PORT` | 1200 | Elasticsearch 외부 노출 포트 |
| `MINIO_PORT` / `MINIO_CONSOLE_PORT` | 9000 / 9001 | MinIO API / 콘솔 포트 |
| `REDIS_PORT` | 6379 | Redis(Valkey) 포트 |
| `TZ` | Asia/Seoul | 타임존 |
| `REGISTER_ENABLED` | 1 | 사용자 자가 가입 허용 여부 |
| `ELASTIC_PASSWORD` / `MYSQL_PASSWORD` / `MINIO_PASSWORD` / `REDIS_PASSWORD` | (기재 생략) | 사내 비밀 관리 절차에 따라 별도 보관 |

**배포 명령**:
```bash
cd /docker/compose/ragflow
# .env 파일에 위 변수 설정 (비밀번호는 openssl rand -hex 32 로 생성)
sudo docker compose up -d
```
> 실행 중인 서비스: `ragflow-cpu`(웹/API), `mysql`(8.0.39), `es01`(Elasticsearch 8.11.3), `minio`(pgsty/minio), `redis`(valkey/valkey:8). `.env`의 `TEI_MODEL`(기본값 `Qwen/Qwen3-Embedding-0.6B`) 임베딩 서비스는 **현재 프로파일에 포함되어 있지 않아 미기동 상태** — 임베딩을 RAGFlow 내장 서비스로 쓸지, 외부 모델 프로바이더(vLLM 등)로 연결할지는 설치 후 RAGFlow Admin 설정에서 결정 필요.

### 1.4 GBrain 배포
자체 Dockerfile로 이미지를 빌드하여 구성합니다.

```dockerfile
FROM oven/bun:1
WORKDIR /app
RUN bun install -g gbrain
# Postgres+pgvector 연결
EXPOSE 8787
CMD ["gbrain", "serve", "--http"]
```
- 지식 데이터 저장을 위해 별도 Postgres(+pgvector 확장) 컨테이너를 함께 구성 (예: `pgvector/pgvector:pg16`)

```bash
cd /docker/compose/gbrain
sudo docker build -t gbrain:0.42.63.0 .
sudo docker compose up -d
```

---

## 2. vLLM 서버 (192.168.47.185) 설치 절차

### 2.1 NVIDIA 드라이버 및 CUDA 저장소 등록 (실측 구성)
```bash
# NVIDIA CUDA 저장소 (developer.download.nvidia.com)
# /etc/apt/sources.list.d/cuda-ubuntu2404-x86_64.list 로 등록됨

# NVIDIA Container Toolkit 저장소 (nvidia.github.io)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y nvidia-driver-open nvidia-dkms-open nvidia-container-toolkit
sudo reboot
```
실측 설치 버전: 드라이버/컨테이너 도구 `610.43.02-1ubuntu1`, `nvidia-container-toolkit 1.19.1-1`

### 2.2 Docker에 NVIDIA 런타임 등록
```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```
실측 `/etc/docker/daemon.json`:
```json
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

### 2.3 GPU 인식 확인
```bash
nvidia-smi
# NVIDIA RTX PRO 6000 Blackwell Max-Q Workstation Edition x2, 각 97,887 MiB VRAM 인식되어야 함
```

### 2.4 모델 파일 배치
```bash
sudo mkdir -p /data/models /data/vllm-cache
# Qwen3.5-122B-A10B-FP8 모델 가중치를 /data/models/Qwen3.5-122B-A10B-FP8 에 배치 (실측 용량 119GB)
# Hugging Face에서 다운로드하는 경우:
#   huggingface-cli download Qwen/Qwen3.5-122B-A10B-FP8 --local-dir /data/models/Qwen3.5-122B-A10B-FP8
```

### 2.5 vLLM 컨테이너 배포
`/docker/compose/qwen35-122b/docker-compose.yml` (실측 원문):
```yaml
services:
  vllm:
    image: vllm/vllm-openai:latest
    container_name: vllm-qwen35-122b
    restart: unless-stopped
    runtime: nvidia
    ipc: host
    network_mode: host
    volumes:
      - /data/models:/data/models:ro
      - /data/vllm-cache:/root/.cache/vllm
    environment:
      - HUGGING_FACE_HUB_HOME=/tmp/hf-cache
      - OMP_NUM_THREADS=8
    command: >
      --model /data/models/Qwen3.5-122B-A10B-FP8
      --served-model-name Qwen3.5-122B-A10B-FP8
      --host 0.0.0.0
      --port 8100
      --tensor-parallel-size 2
      --gpu-memory-utilization 0.95
      --max-model-len 131072
      --max-num-seqs 512
      --trust-remote-code
      --enable-prefix-caching
      --kv-cache-dtype fp8
      --performance-mode throughput
      --enable-flashinfer-autotune
      --async-scheduling
      --enable-chunked-prefill
      --enable-auto-tool-choice
      --tool-call-parser qwen3_coder
      --reasoning-parser qwen3
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```
```bash
cd /docker/compose/qwen35-122b
sudo docker compose up -d
```

### 2.6 배포 확인
```bash
curl http://localhost:8100/v1/models
# {"data":[{"id":"Qwen3.5-122B-A10B-FP8", ...}]} 형태 응답 확인

sudo docker logs -f vllm-qwen35-122b
# "Uvicorn running on http://0.0.0.0:8100" 로그 확인 후 정상
```

---

## 3. 설치 후 연동 확인
1. vLLM 서버(§2) 정상 기동 확인 (8100 포트 응답)
2. VM 서버 OpenWebUI(§1.2) 접속 → `http://192.168.47.115:3000` → 모델 목록에 `Qwen3.5-122B-A10B-FP8` 표시 확인
3. RAGFlow(§1.3) 접속 → `http://192.168.47.115:8030` → 초기 관리자 계정 생성 → 모델 프로바이더 설정에서 vLLM 서버 API(`http://192.168.47.185:8100/v1`) 등록

