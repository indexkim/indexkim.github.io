---
title: 폐쇄망 AI Solution 서비스 구성 및 배포
tags: [deploy, engine, model, system]
toc: true
toc_sticky: true
post_no: 10
---

이번 글에서는 폐쇄망 AI Solution 개발환경 구축에 이어,  
실제 서비스 구성과 배포 과정에 대해 정리합니다.  
RHEL 9을 기준으로 하지만, 다른 Linux 배포판에서도 동일한 방식으로 적용할 수 있습니다.

> 특정 조직의 내부 시스템이나 정책, 네트워크 설정은 포함하지 않으며  
> 모든 내용은 공개된 기술과 일반적인 Linux 환경을 기반으로 구성되었습니다.

---


## 15. 파인튜닝

본 프로젝트에서는 VLM(Vision-Language Model) 기반으로 OCR 파인튜닝을 수행했습니다.  
기존 PaddleOCR 기반 파인튜닝은 [PaddleOCR 기반 도메인 특화 OCR fine-tuning](/ocr-fine-tuning/)에서 다뤘으며,  
이번에는 VLM 기반 OCR 방식을 적용했습니다.

---

### VLM(Vision-Language Model)이란?

VLM은 이미지와 텍스트를 함께 처리할 수 있는 멀티모달 모델입니다.  
GPT 계열, Gemini 계열, Qwen-VL과 같은 모델들이 대표적인 VLM에 해당합니다.  
다만 GPT 및 Gemini와 같은 상용 멀티모달 모델은  
내부 Vision–Language 결합 구조가 공개되어 있지 않습니다.  
아래 구조는 주로 오픈소스 VLM에서 일반적으로 사용되는 아키텍처입니다.

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Vision Encoder  │────▶│  Projector   │────▶│ Language Model  │
│ (이미지 인코더)  │     │ (모달리티 정렬) │     │  (텍스트 생성)   │
└─────────────────┘     └──────────────┘     └─────────────────┘
```

| 컴포넌트 | 역할 | 대표 예시 |
|----------|------|----------|
| Vision Encoder | 이미지를 feature vector로 인코딩 | CLIP ViT, SigLIP, EVA-CLIP |
| Projector | 이미지 feature를 LLM embedding space로 변환 | Linear/MLP projection |
| Language Model | 이미지 정보와 텍스트를 기반으로 응답 생성 | Qwen2, LLaMA, Mistral |

VLM에 "이 이미지의 텍스트를 읽어줘"라고 프롬프트를 주면  
이미지 속 텍스트를 인식하여 자연어로 출력합니다.  
이처럼 텍스트 인식을 LLM 생성 과정으로 수행하는 방식을 VLM OCR이라 합니다.

---

### OCR 전용 모델 vs VLM OCR

| 항목 | OCR 전용 모델 (PaddleOCR, EasyOCR) | VLM OCR |
|------|-------------------------------------|---------|
| 파이프라인 | Detection → Recognition (2단계) | 이미지 → 텍스트 생성 (End-to-End) |
| 출력 형태 | 좌표(bbox) + 텍스트 | 텍스트 (좌표는 별도 설계 필요) |
| 모델 크기 | 수십 MB ~ 수백 MB | 수 GB 이상 |
| 학습 데이터 | Detection/Recognition 각각 별도 라벨 필요 | 이미지-텍스트 쌍 기반 학습 |
| 파인튜닝 | 모델별 전용 학습 코드 필요 | LoRA 등 LLM 기반 범용 기법 활용 |
| 복잡한 레이아웃 | 좌표 기반 정밀 제어에 강점 | 문맥 기반 보완 가능 |
| 추론 속도 | 빠름 | 상대적으로 느림 (생성 기반) |

> OCR 전용 모델은 좌표 기반 정밀 탐지와 빠른 추론에 강점이 있고  
> VLM OCR은 문맥 이해 기반의 유연한 텍스트 인식에 강점이 있음  
> 두 방식은 대체 관계가 아니라 용도에 따라 선택하는 보완 관계

---

### 1. 데이터셋 구축: 모델 보조 라벨링

학습 데이터는 영상에서 자막이 표시되는 구간을 캡처하여 구축했습니다.  
사전학습된 베이스 VLM의 추론 결과를 초안 라벨로 활용하고,  
사람이 오류를 수정한 뒤 해당 데이터로 파인튜닝을 진행했습니다.

```
영상에서 자막 구간 캡처 (이미지 추출)
        ↓
베이스 VLM으로 인퍼런스 (초안 라벨 생성)
        ↓
사람이 오류 수정 (Human-in-the-Loop 검수)
        ↓
수정된 데이터로 파인튜닝 학습
```

> 해당 방식은 Model-Assisted Labeling에 해당하며  
> 모델 예측을 사람이 검수·보정하여 최종 정답을 확정하는 Human-in-the-Loop 기반 전략임  
> 모델 예측을 그대로 학습에 사용하는 Pseudo Labeling과는 구분됨  
> 데이터셋 규모가 작을수록 라벨 품질이 성능에 미치는 영향이 큼

---

### 2. LoRA 파인튜닝

LoRA(Low-Rank Adaptation)는 대규모 모델의 원본 가중치를 고정한 채  
저랭크 어댑터 행렬만 추가 학습하는 기법입니다.  
전체 파라미터 대비 소수(보통 1% 내외)만 학습하므로 메모리 효율이 높지만,  
데이터가 매우 적은 경우에는 여전히 과적합이 발생할 수 있습니다.

VLM 파인튜닝 시에는 Vision Encoder와 Projector를 동결(Freeze)하고  
Language Model에만 LoRA를 적용하는 것이 일반적입니다.  
다만 도메인 특성이 강한 경우 일부 레이어를 추가 학습하기도 합니다.

| 컴포넌트 | 학습 여부 | 이유 |
|----------|----------|------|
| Vision Encoder | Freeze | 사전학습된 이미지 특징 보존 |
| Projector | Freeze | 일반적으로 동결, 도메인 갭이 큰 경우만 학습 |
| Language Model | LoRA 적용 | 태스크 특화 텍스트 생성 학습 |

> LoRA는 어댑터 가중치만 저장하므로 용량이 수십 MB 수준으로 매우 작음  
> 데이터가 매우 적은 경우 상위 레이어에만 LoRA를 적용하는 방식도 고려할 수 있음

---

### 3. 평가

OCR 성능 평가에는 CER(Character Error Rate, 문자 오류율)을 사용합니다.  
CER은 모델이 출력한 텍스트와 정답 텍스트 사이의 편집 거리를 문자 단위로 측정하며,  
단어 단위 지표(WER)보다 OCR 태스크에 적합합니다.  
평가 시에는 공백 및 특수문자 처리 기준을 사전에 정의하여 일관된 방식으로 CER을 계산했습니다.

파인튜닝 후 베이스 모델 대비 CER이 개선되었으며,  
소규모 데이터셋임에도 Model-Assisted Labeling 방식의 효과를 확인할 수 있었습니다.

---

### 4. 가중치 관리

학습 완료 시 어댑터 파일(`adapter_config.json`, `adapter_model.safetensors`) 2개만 저장됩니다.  
전체 모델을 다시 가져올 필요 없이 어댑터 파일만 교체하면 됩니다.

> 여러 버전의 어댑터를 관리하면 태스크별 모델 전환이 가능  
> 다만 런타임 전환 가능 여부는 사용하는 서빙 프레임워크에 따라 달라짐

---

## 16. 모델 서빙

모델 서빙은 학습이 완료된 AI 모델을 실시간으로 호출할 수 있는 상태로 배포하는 과정입니다.  
학습된 가중치 파일(.pt, .safetensors, .trt 등)을 메모리에 로드하고,  
외부 요청에 대해 추론 결과를 반환하는 API 형태로 운영합니다.

모델 서빙은 크게 두 가지 요소로 구성됩니다.

| 구분 | 설명 |
|------|------|
| 추론 엔진 | 모델을 실행하는 런타임 (TensorRT, vLLM, SGLang, Transformers 등) |
| 서빙 레이어 | 추론 결과를 API로 제공하는 웹 서버 (FastAPI, Flask, Triton 등) |

---

### 1. Hugging Face Transformers

Transformers는 Hugging Face에서 개발한 오픈소스 라이브러리로,  
다양한 pretrained 모델(LLM, VLM, Vision 등)을  
로드하고 추론·학습할 수 있는 통합 프레임워크입니다.

```python
from transformers import AutoModelForCausalLM, AutoProcessor

model = AutoModelForCausalLM.from_pretrained("model_path", trust_remote_code=True)
processor = AutoProcessor.from_pretrained("model_path", trust_remote_code=True)
```

| 항목 | 설명 |
|------|------|
| 역할 | 모델 로드, 토크나이징, 추론, 학습을 위한 기본 라이브러리 |
| 지원 모델 | Hugging Face Hub에 등록된 수만 개의 모델 |
| `trust_remote_code` | 모델 폴더 내 커스텀 Python 코드(*.py)를 직접 실행하여 비표준 아키텍처도 로드 가능 |
| 위치 | 추론 엔진(vLLM 등)이나 서빙 프레임워크(FastAPI 등)의 기반 레이어 |

vLLM, SGLang 등은 Hugging Face 모델 포맷과 토크나이저를 호환하여 로드합니다.  
다만 Transformers의 기본 추론 API는 대규모 서빙 최적화를 제공하지 않으므로,  
고부하 프로덕션 환경에서는 전용 추론 엔진을 사용하는 것이 일반적입니다.  

---

### 2. 추론 엔진 비교

#### (1) TensorRT

NVIDIA의 GPU 최적화 추론 엔진입니다.  
모델을 `.trt` 엔진 파일로 변환하여 Layer Fusion, Kernel Auto-Tune, Quantization 등  
하드웨어 수준의 최적화를 적용합니다.

```bash
# ONNX → TensorRT 변환 예시
trtexec --onnx=model.onnx --saveEngine=model.trt --fp16
```

| 항목 | 설명 |
|------|------|
| 대상 모델 | YOLO, ResNet 등 고정 구조 모델 |
| 장점 | 최고 수준의 추론 속도, FP16/INT8 경량화 내장 |
| 단점 | GPU 아키텍처에 종속 (Ampere에서 생성한 엔진은 Blackwell에서 사용 불가) |
| VLM 지원 | 제한적 (커스텀 아키텍처는 변환 불가능한 경우 많음) |

> TensorRT 엔진은 생성 시점의 GPU 아키텍처에 종속되므로  
> 폐쇄망 반입 시 현장 GPU와 동일한 환경에서 변환해야 함

---

#### (2) vLLM

LLM/VLM 전용 고속 추론 엔진입니다.  
PagedAttention, Continuous Batching 등 LLM에 특화된 최적화를 제공하며  
OpenAI 호환 API를 기본으로 지원합니다.

| 항목 | 설명 |
|------|------|
| 대상 모델 | LLM, VLM (Hugging Face 호환 모델) |
| 장점 | 높은 처리량, Continuous Batching, OpenAI 호환 API 기본 제공 |
| 제약 | 공식 지원 모델 목록 외 아키텍처는 추가 구현 필요 |
| 폐쇄망 | CUDA 버전 의존성 있음, GPU 아키텍처별 이미지 구분 필요 |

---

#### (3) SGLang

SGLang은 LLM/VLM 추론 및 프롬프트 프로그래밍을 위한 프레임워크입니다.  
RadixAttention 기반의 KV Cache 재사용 구조를 사용하여  
동일 prefix를 공유하는 반복 프롬프트 처리에 강점이 있습니다.  

| 항목 | 설명 |
|------|------|
| 대상 모델 | LLM, VLM (Hugging Face 호환 모델) |
| 장점 | KV Cache 재사용, 구조화된 출력 지원 |
| 제약 | 공식 지원 외 아키텍처는 추가 구현 필요 |
| 폐쇄망 | Blackwell에서는 dev 이미지 필요, Ampere/Hopper는 안정 릴리즈 사용 가능 |

---

#### vLLM / SGLang Docker 배포

vLLM과 SGLang은 공식 Docker 이미지를 사용하여 컨테이너로 실행합니다.  


##### vLLM

```bash
docker run -itd \
    --name vllm_server \
    --gpus '"device=0"' \
    --ipc=host \
    -p <PORT>:<PORT> \
    -v /path/to/models:/models \
    vllm/vllm-openai:v0.11.0 \
    --model /models/VLM-7B \
    --host 0.0.0.0 \
    --port <PORT> \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.7 \
    --max-model-len 8192
```

##### SGLang

```bash
docker run -itd \
    --name sglang_server \
    --gpus '"device=0"' \
    --shm-size 32g \
    -p <PORT>:<PORT> \
    -v /path/to/models:/models \
    --ipc=host \
    lmsysorg/sglang:v0.5.4 \
    python3 -m sglang.launch_server \
    --model-path /models/model_name \
    --host 0.0.0.0 --port <PORT> \
    --mem-fraction-static 0.8 \
    --context-length 70000
```

##### 정상 실행 확인

```bash
# 컨테이너 로그 확인
docker logs -f vllm_server
# → "Application startup complete" 메시지 확인

# API 응답 확인
curl http://localhost:<PORT>/v1/models
# → 200 OK
```

| 주요 옵션 | 설명 |
|-----------|------|
| `--tensor-parallel-size` | 모델을 분할할 GPU 수 (30B 이상 모델은 2+ 권장) |
| `--gpu-memory-utilization` | GPU 메모리 사용 비율 (0.6~0.9, 멀티 모델 시 낮게 설정) |
| `--max-model-len` | 최대 컨텍스트 길이 (메모리에 직접 영향) |
| `--mem-fraction-static` | SGLang 전용, KV Cache에 할당할 메모리 비율 |

> 폐쇄망에서는 Docker 이미지를 tar로 반입해야 하므로  
> `docker save` / `docker load`로 이미지를 준비함 ([폐쇄망 AI Solution 개발환경 구축 (2)](/building-air-gapped-ai-solution-dev-env-2/#12-docker-설치-및-gpu-연동) 참고)

---

#### 참고: Blackwell GPU 환경에서의 차이

Blackwell(B200, B300 등) 환경에서는 CUDA 13.0 이상이 필요하며,  
작성 시점 기준으로 안정 릴리즈 대신 nightly/dev 이미지를 사용해야 할 수 있습니다.

| 항목 | Ampere / Hopper | Blackwell |
|------|----------------|-----------|
| vLLM 이미지 | `v0.11.0` (안정) | `cu130-nightly-x86_64` (nightly) |
| SGLang 이미지 | `v0.5.4` (안정) | `dev-cu13` (dev) |
| vLLM 추가 설정 | 없음 | CUDA 13.0 ptxas 바이너리 주입 + `TRITON_PTXAS_PATH` 필요 |
| SGLang 추가 설정 | 없음 | 이미지 변경만으로 동작 |

vLLM은 Blackwell에서 Triton 커널 컴파일 시  
`ptxas fatal: Value 'sm_103a' is not defined` 오류가 발생합니다.  
CUDA 13.0 툴킷의 ptxas 바이너리를 추출하여  
`-v ptxas:/opt/ptxas:ro -e TRITON_PTXAS_PATH=/opt/ptxas`로 주입하면 해결됩니다.

> SGLang은 이미지 태그만 변경하면 Blackwell에서 동작  
> vLLM은 이미지 교체 + ptxas 주입이 모두 필요

---

#### (4) Transformers + FastAPI (직접 구현)

Hugging Face Transformers로 모델을 직접 로드하고  
FastAPI로 API 서버를 구성하는 방식입니다.  
대부분의 Hugging Face 호환 모델에서 사용할 수 있습니다.

```bash
# FastAPI 서버 실행
uvicorn app.main:app --host 0.0.0.0 --port <PORT>
```

| 항목 | 설명 |
|------|------|
| 대상 모델 | 모든 Hugging Face 모델 (커스텀 아키텍처 포함) |
| 장점 | 어떤 모델이든 서빙 가능, API 구조를 자유롭게 설계 가능 |
| 단점 | Continuous Batching 미지원, 최적화를 직접 구현해야 함 |
| 폐쇄망 | 의존성이 적어 반입이 비교적 간편 |

---

### 3. 서빙 방식 비교

| 항목 | TensorRT | vLLM | SGLang | Transformers + FastAPI |
|------|----------|------|--------|----------------------|
| 대상 | Vision 모델 (YOLO 등) | LLM / VLM | LLM / VLM | 모든 모델 |
| 속도 | 최고 | 높음 | 높음 | 보통 |
| Continuous Batching | X | O | O | X |
| OpenAI 호환 API | X | O (기본) | O (기본) | 직접 구현 |
| 커스텀 아키텍처 | 변환 가능 시 | 지원 목록 한정 | 지원 목록 한정 | 모든 모델 가능 |
| LoRA 어댑터 전환 | X | O | O | 직접 구현 가능 |
| 폐쇄망 설치 난이도 | 중간 | 높음 (의존성 많음) | 높음 (의존성 많음) | 낮음 |

> 일부 커스텀 아키텍처 기반 VLM은 vLLM, SGLang 등 범용 추론 엔진을 지원하지 않음  
> 이 경우 Transformers + FastAPI 조합이 유일한 서빙 방법이 됨

---

### 4. 커스텀 아키텍처 VLM의 서빙 제약

일부 VLM은 vLLM/SGLang의 공식 지원 목록에 포함되지 않은 커스텀 아키텍처를 사용합니다.  
이번 프로젝트에서 사용한 VLM도 이에 해당했습니다.

| 제약 | 설명 |
|------|------|
| 커스텀 Projector | vLLM/SGLang의 모델 레지스트리가 인식하지 못함 |
| 커스텀 이미지 처리 | vLLM의 멀티모달 파이프라인과 호환되지 않음 |
| 동적 타일링 | 입력마다 텐서 shape이 달라져 Continuous Batching과 충돌 |

vLLM/SGLang이 해당 아키텍처를 공식 지원하기 전까지는  
Transformers의 `trust_remote_code=True`로 직접 로드하는 것이 유일한 방법이었습니다.

---

### 5. FastAPI 기반 VLM 서빙 구성

위의 제약으로 인해 Transformers + PEFT + FastAPI 조합으로 직접 서빙 서버를 구축했습니다.  
API 형식은 OpenAI 호환으로 구성하여  
추후 vLLM/SGLang 전환 시 클라이언트 수정이 불필요하도록 설계했습니다.

```
[FastAPI 서버]
├── /health                   # 헬스 체크
├── /v1/chat/completions      # OpenAI 호환 API (단건 / 스트리밍)
├── /v1/batch/ocr             # 배치 OCR API (최대 8건)
└── /ocr                      # 단순 OCR API
```

서빙 시에는 베이스 모델과 LoRA 어댑터를 분리하여 로드하며,  
`use_lora` 파라미터로 런타임에 베이스/파인튜닝 모델을 전환할 수 있습니다.

> 어댑터 폴더에는 `adapter_config.json`과 `adapter_model.safetensors` 2개만 있어야 함  
> 베이스 모델의 설정 파일이 함께 있으면 로드 시 충돌 발생

#### Docker 컨테이너 실행

```bash
docker run -d --gpus '"device=0"' \
    --shm-size 32g \
    -p <PORT>:<PORT> \
    -v /path/to/model:/workspace/model/VLM \
    -v /path/to/adapter:/workspace/adapter/best_model_clean \
    -v /path/to/server:/workspace/server \
    --name vlm_ocr_server \
    base_image:latest \
    uvicorn app.main:app --host 0.0.0.0 --port <PORT>
```

#### 정상 실행 확인

```bash
curl http://localhost:<PORT>/health
# → {"status": "ok", "model_type": "both (base + finetuned)"}
```

> 모델과 어댑터를 별도 볼륨으로 마운트하면 어댑터만 교체하여 모델 버전 전환 가능  
> 폐쇄망에서는 서버 실행 후 반드시 헬스 체크와 샘플 추론으로 정상 동작을 확인해야 함

---

## 17. 분석엔진 연동

분석엔진은 AI 모델의 실행과 외부 시스템 연동을 담당하는 백엔드 서비스입니다.  
컨테이너 매니저와 분석 스케줄러로 구성되며,  
컨테이너 매니저가 각 AI 모델별 분석 컨테이너를 생성·관리합니다.

```
         ┌── 분석엔진 (컨테이너 매니저 + 스케줄러 + 분석 컨테이너들)
         │
웹 서비스 ┤
         │
         └── 검색백엔드 ←── Kafka ←── 분석엔진
```

---

### 1. 시작 및 종료

컨테이너 매니저를 먼저 시작한 후 스케줄러를 시작해야 합니다.

```bash
# 컨테이너 매니저 시작
cd ~/engine/container_manager
./run.sh

# 스케줄러 시작
cd ~/engine/scheduler
./run.sh
```

```bash
# 상태 확인
docker ps --filter "name=container_manager" --filter "name=scheduler"

# 로그 확인
docker logs -f container_manager
docker logs -f scheduler
```

---

### 2. IP 및 연동 설정

스케줄러의 `run.sh`에서 각 컴포넌트의 IP와 포트를 환경 변수로 설정합니다.

| 환경 변수 | 설명 |
|-----------|------|
| `AGENT_API_HOST` / `PORT` | 컨테이너 매니저 서버 주소 |
| `SCHEDULER_HOST` / `PORT` | 스케줄러 자신의 주소 |
| `VIDEO_SEARCH_API_HOST` / `PORT` | 검색 API 서버 주소 |
| `KAFKA_BROKER_URI` | Kafka 브로커 주소 |

컨테이너 매니저의 `params.cfg`에서는  
분석 컨테이너들이 사용하는 환경 변수와 볼륨 마운트 경로를 설정합니다.

> params.cfg 변경 후에는 기존 분석 컨테이너들도 리셋해야 새 설정이 적용됨  
> `./reset_containers.sh <scheduler_host:port>` 명령으로 일괄 리셋 가능

---

### 3. 분석 컨테이너 관리

컨테이너 매니저는 분석 요청에 따라 GPU가 연동된 분석 컨테이너를 자동으로 생성합니다.  
각 컨테이너는 독립된 모델을 탑재하고 있으며,  
분석이 완료되면 결과를 Kafka를 통해 검색백엔드로 전달합니다.

```bash
# 분석 컨테이너 일괄 리셋
./reset_containers.sh localhost:<SCHEDULER_PORT>

# 실행 중인 분석 컨테이너 확인
docker ps | grep analysis
```

> 분석 컨테이너 내부에서 GPU를 사용하므로 NVIDIA 드라이버 및 Container Toolkit이   
> 정상 설치되어 있어야 함 ([폐쇄망 AI Solution 개발환경 구축 (2)](/building-air-gapped-ai-solution-dev-env-2/#10-nvidia-드라이버-설치) 참고)

---

## 18. 검색백엔드 배포

검색백엔드는 분석엔진의 결과를 수집·인덱싱하고, 조건 기반 검색 API를 제공하는 서비스입니다.  
코드 개발은 별도 담당자가 수행하며,  
폐쇄망 환경에서는 솔루션 담당자가 배포, 로그 확인, 장애 대응을 직접 수행합니다.

### 1. 구성 요소

| 구성 요소 | 역할 | 비고 |
|-----------|------|------|
| Kafka | 분석엔진과의 데이터 송수신 | 분석 전 반드시 실행 |
| Elasticsearch | 분석 데이터 인덱싱 및 검색 | 분석 전 반드시 실행 |
| 검색 API 서버 | 웹과 통신하는 검색 API | docker compose로 실행 |
| 검색 오케스트레이터 | LLM 기반 쿼리 이해 및 검색 수행 | docker compose로 실행 |
| LLM 서빙 (SGLang) | 검색·요약을 위한 LLM | [16장](#16-모델-서빙) 참고 |

---

### 2. 시작 및 종료

```bash
# Kafka
cd ~/infrastructure/kafka
docker compose up -d

# Elasticsearch
cd ~/infrastructure/elasticsearch
docker compose up -d

# 검색 API 서버
cd ~/search/api/docker
docker compose -f docker-compose.release.yml up -d

# 검색 오케스트레이터
cd ~/search/orchestrator/docker
docker compose -f docker-compose.release.yml up -d
```

---

### 3. IP 설정

각 컴포넌트의 `.env` 파일에서 IP와 포트를 설정합니다.

```bash
# 검색 API 서버 (.env)
SERVER_EXTERNAL_HOST=<검색_API_SERVER_IP>
ELASTICSEARCH_HOST=https://<ES_SERVER_IP>:<ES_PORT>
KAFKA_HOST=<KAFKA_SERVER_IP>
```

> 설정 변경 후 반드시 해당 컴포넌트를 재시작해야 적용됨

---

### 4. 로그 확인 및 에러 추적

```bash
# 컨테이너 로그
docker logs -f search_api
docker logs -f search_orchestrator

# 파일 로그
tail -f ~/search/api/logs/orchestration.log
tail -f ~/search/orchestrator/logs/orchestrator.log
```

---

### 5. 폐쇄망에서 직접 해결한 이슈

#### (1) Elasticsearch 디렉터리 권한 문제

Elasticsearch 컨테이너는 root가 아닌 전용 사용자(기본 UID 1000)로 실행됩니다.  
호스트에 bind mount한 데이터/로그 디렉터리의 UID/GID가 일치하지 않으면  
로그 파일 생성 및 rename 단계에서 JVM 초기화가 실패할 수 있습니다.  

```
gc.log permission error
cannot change name gc.log to gc1.log
```

해당 오류는 GC 로그 파일에 대한 쓰기/rename 권한이 없어 발생한 문제였습니다.

일반적인 환경에서는 `chown`/`chgrp`를 통해 UID/GID를 일치시키는 것이 권장됩니다.  
그러나 본 사례에서는 다음과 같은 제약이 존재했습니다.

- sudo 권한 미제공
- 호스트 파일 소유권 변경 불가
- 컨테이너 생성 권한만 허용

이 제약 하에서 컨테이너 내부 root 권한을 활용하여  
bind mount된 디렉터리의 접근 권한을 간접적으로 조정하는 방식으로 대응했습니다.

```bash
docker run --rm -v /path/to/es_data:/data <image> chmod -R 777 /data
docker run --rm -v /path/to/es_logs:/data <image> chmod -R 777 /data
```

> `chmod 777`은 보안상 권장되지 않지만,   
> 권한이 제한된 환경에서 서비스 가동을 위한 현실적인 대응 전략임  
> 동일 증상은 SELinux 컨텍스트 미적용에서도 발생할 수 있으며  
> 이 경우 `restorecon`으로 해결 가능 ([폐쇄망 AI Solution 개발환경 구축 (2)](/building-air-gapped-ai-solution-dev-env-2/#12-1-docker-데이터-저장-경로-변경) 참고)

---

#### (2) Elasticsearch 클러스터 재시작 시 노드 종료

Elasticsearch 클러스터를 최초로 부트스트랩한 뒤에는 `docker-compose.yml`의 `cluster.initial_master_nodes` 설정을 제거하는 것이 원칙입니다.  
클러스터가 이미 형성된 상태에서 이 설정을 남겨두면,  
재시작 시 불필요한 부트스트랩 시도를 유발하거나  
cluster UUID 불일치로 실행 실패가 발생할 수 있습니다.

---

#### (3) Elasticsearch 인덱싱 확인

Elasticsearch와 Kibana를 실행한 뒤, Kibana UI를 통해 인덱싱 상태를 확인합니다.  

Kibana 인덱스 관리 화면(`/app/elasticsearch/content/search_indices`)에서  
인덱스 목록과 도큐먼트 수가 정상적으로 표시되고,  
분석 이후 도큐먼트 수가 증가하는지 확인합니다.  

도큐먼트 수가 증가하지 않으면 다음 순서로 점검합니다.

- Kafka 연결 상태 확인
- 검색백엔드 컨슈머/인덱서 로그 확인
- Elasticsearch 로그에서 인덱싱 에러(4xx/5xx) 확인

---

#### (4) Kafka 브로커 연결 실패

Kafka가 실행되었으나 분석엔진이나 검색 서버에서 연결하지 못하는 경우,  
Kafka의 `KAFKA_ADVERTISED_HOST_NAME`이 실제 서버 IP와 일치하는지 확인합니다.

```bash
# Kafka env 파일 확인
KAFKA_ADVERTISED_HOST_NAME=<실제_서버_IP>
```

---

## 19. 웹 서비스 배포

웹 서비스는 사용자 대시보드, 분석 요청 및 결과 조회를 담당하는 서비스입니다.  
코드 개발은 별도 담당자가 수행하며,  
폐쇄망 환경에서는 솔루션 담당자가 배포, 로그 확인, 장애 대응을 직접 수행합니다.

### 1. 이미지 반입 및 실행

웹 서비스는 프론트엔드, 백엔드, DB(MySQL)로 구성되며,  
각각 별도의 Docker 이미지를 tar로 반입하여 배포합니다.

```bash
# 이미지 로드
docker load -i web_frontend.tar.gz
docker load -i web_backend.tar.gz
docker load -i mysql.tar.gz

# 컨테이너 실행 (예: 프론트엔드)
docker run -d \
    --name web_frontend \
    --network="host" \
    -e PORT=<PORT> \
    -e HOSTNAME=0.0.0.0 \
    --restart unless-stopped \
    web_frontend:v1.0
```

> 백엔드와 DB도 동일한 방식으로 이미지 로드 후 컨테이너를 실행  
> 버전 업데이트 시에는 기존 컨테이너를 중지·삭제한 뒤 새 이미지를 로드하여 재실행

---

### 2. DB 설정 (MySQL)

웹 백엔드의 설정은 MySQL DB를 통해 관리됩니다.

```bash
# DB 컨테이너 접속
docker exec -it db_container /bin/bash
mysql -u root -p
use app_db;
```

#### AI 모델 등록

```sql
INSERT INTO <모델_설정_테이블>
(name, description, ip, port, model_data)
VALUES
('VLM-7B', '멀티모달 모델', '<VLM_SERVER_IP>', '<VLM_PORT>', '/models/VLM-7B');
```

#### IP 설정 변경

```sql
UPDATE <시스템_설정_테이블> SET value = 'http://<HOST>:<PORT>' WHERE type = 'search.base-url';
UPDATE <시스템_설정_테이블> SET value = 'http://<HOST>:<PORT>' WHERE type = 'engine.base-url';
```

---

### 3. 운영 중 발생한 이슈

#### 분석 에러 시 웹 무한 대기

분석 도중 에러가 발생하면 웹에서 분석 상태가 "진행 중"으로 멈추는 경우가 있습니다.  
이때 DB에서 직접 status 값을 변경하여 강제로 실패 또는 완료 처리합니다.

```sql
-- 분석 상태 강제 변경 (실패 처리)
UPDATE <분석_작업_테이블> SET status = 'FAILED' WHERE task_id = <id>;

-- 또는 완료 처리
UPDATE <분석_작업_테이블> SET status = 'COMPLETED' WHERE task_id = <id>;
```

> 각 컴포넌트 간 연동은 일직선이 아니라 분석-웹, 분석-검색, 검색-웹 등 다대다 구조  
> 에러 원인이 어느 연결에서 발생했는지 파악하는 것이 핵심

#### 에러 추적 흐름

```
                ┌── 웹 (분석 상태 조회)
분석엔진 ──────┤
                └── Kafka → 검색백엔드 (결과 인덱싱)
                                 │
                            웹 (검색 결과 조회)
```

```
웹에서 무한 대기 발견
    ↓
1. 분석엔진 로그 확인 → 분석 자체가 실패했는지?
2. Kafka 연결 상태 확인 → 결과 전달이 끊겼는지?
3. 웹 백엔드 로그 확인 → 콜백을 수신하지 못했는지?
    ↓
원인에 따라 조치 (재시작 / DB 강제 변경 / 설정 수정)
```

> 폐쇄망에서는 솔루션 담당자가 모든 컴포넌트의 로그를 직접 확인해야 함  
> 코드 수정이 필요한 경우 개발자에게 로그를 전달하고 수정된 이미지를 재반입

---

## 20. 통합 운영 및 테스트

### 1. 서비스 실행 순서

전체 서비스는 의존성에 따라 순서대로 실행해야 합니다.

```
1. 인프라 (Kafka, Elasticsearch)
    ↓
2. LLM 서빙 (SGLang / vLLM)
    ↓
3. 분석엔진 (컨테이너 매니저 → 스케줄러)
    ↓
4. 검색백엔드 (검색 API → 검색 오케스트레이터)
    ↓
5. 웹 서비스 (프론트엔드, 백엔드)
```

> 역순으로 종료하는 것을 권장  
> Kafka/Elasticsearch가 내려간 상태에서 분석이 실행되면 데이터 유실 가능

---

### 2. E2E 통합 테스트

모든 서비스가 실행된 후 다음 흐름을 순차적으로 검증합니다.

| 단계 | 확인 항목 |
|------|----------|
| 1 | 웹에서 분석 요청 → 스케줄러가 요청 수신 |
| 2 | 컨테이너 매니저가 분석 컨테이너 생성 → GPU 연동 확인 |
| 3 | 분석 완료 → Kafka로 결과 전송 |
| 4 | 검색백엔드가 결과 수신 → Elasticsearch 인덱싱 |
| 5 | 웹에서 검색 → 결과 조회 확인 |

---

### 3. 운영 모니터링

| 항목 | 확인 방법 |
|------|----------|
| GPU 사용률 | `nvidia-smi`, `nvtop`, Grafana 등 |
| 컨테이너 상태 | `docker ps -a` |
| 디스크 사용량 | `df -h` (로그·분석 결과 누적 확인) |
| Kafka 상태 | 컨슈머 랙(lag) 확인 |
| Elasticsearch 상태 | `curl -k https://localhost:<ES_PORT>/_cluster/health` |

---

### 4. 장애 대응

#### 서비스 재시작

```bash
# 특정 컨테이너 재시작
docker restart <container_name>

# 분석 컨테이너 일괄 리셋
./reset_containers.sh <scheduler_host:port>

# docker compose 서비스 재시작
cd ~/search/api/docker
docker compose -f docker-compose.release.yml down
docker compose -f docker-compose.release.yml up -d
```

#### 폐쇄망 특화 트러블슈팅

| 증상 | 원인 | 조치 |
|------|------|------|
| 컨테이너 실행 실패 | 이미지 미반입 또는 손상 | `docker images`로 확인 후 재반입 |
| GPU 미인식 | NVIDIA 드라이버 또는 Container Toolkit 이상 | `nvidia-smi` 확인, [폐쇄망 AI Solution 개발환경 구축 (2)](/building-air-gapped-ai-solution-dev-env-2/#10-nvidia-드라이버-설치) 참고 |
| 컴포넌트 간 연결 실패 | IP/포트 설정 불일치 | `.env` 파일 및 `params.cfg` IP 확인 |
| 분석 결과 미수신 | Kafka 연결 끊김 | Kafka 실행 상태 및 `ADVERTISED_HOST_NAME` 확인 |
| 검색 결과 없음 | Elasticsearch 인덱싱 실패 | Elasticsearch 클러스터 상태 및 디렉터리 권한 확인 |
| 웹 무한 대기 | 콜백 누락 | DB status 강제 변경 ([19장](#19-웹-서비스-배포) 참고) |

> 폐쇄망에서는 에러 발생 시 외부 지원이 제한되므로  
> 로그 확인 → 원인 추적 → 조치의 흐름을 솔루션 담당자가 직접 수행해야 함

---


## 마무리

이번 글에서는 폐쇄망 환경에서 AI 서비스를 실제 운영 가능한 형태로 구성하고 배포했습니다.  
모델 고도화부터 통합 운영까지 다음 단계를 직접 수행했습니다.

1. VLM 기반 OCR 파인튜닝 전략 수립 및 LoRA 적용

2. 추론 엔진 비교와 서빙 아키텍처 설계

3. 커스텀 아키텍처 VLM 제약 분석 및 대응

4. 분석엔진·검색 백엔드·웹 서비스 통합 구조 정리

5. 컨테이너 기반 배포 및 운영 안정화

6. 통합 테스트 및 장애 대응 체계 수립


일반적인 환경에서는 인프라, AI, 백엔드, 운영이 분리되지만,  
폐쇄망 환경에서는 모델 학습부터 배포, 서비스 연동, 장애 대응까지  
현장에서 통합적으로 판단하고 정리해야 합니다.

이번 프로젝트에서는  
OS 설치와 네트워크 구성부터  
GPU 드라이버·컨테이너 런타임·원격 개발 환경 구축,  
모델 파인튜닝, 추론 엔진 선택 및 서빙 구조 설계,  
분석엔진·검색 백엔드·웹 서비스 배포,  
서비스 실행 순서 체계화와 현장 이슈 대응까지 맡으며  
전체 시스템이 안정적으로 동작하도록 구조를 정리하고 조율하는 역할을 수행했습니다. 

특히 범용 추론 엔진이 지원하지 않는 커스텀 아키텍처 VLM을  
현장 환경에서 동작하도록 전환하고 검증한 과정은  
기술 제약 속에서 현실적인 대안을 도출해야 했던 작업이었습니다.

모델 개발을 넘어 인프라·서빙·서비스 연동·운영 안정화까지 직접 다루며  
AI 시스템을 구성 요소 단위가 아닌, 실제 운영되는 전체 시스템 기준으로 판단하게 되었습니다.

