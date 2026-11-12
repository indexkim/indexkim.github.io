---
title: Data Analyst에서 Applied AI Engineer까지
tags: [career, vision-ai]
toc: true
toc_sticky: true
post_no: 4
last_modified_at: 2026-11-12
---

AI Engineer라는 직무명은 회사와 조직에 따라 의미가 조금씩 다릅니다.  
어떤 조직에서는 모델 학습이 업무의 중심이고,  
어떤 조직에서는 최적화와 Serving, 시스템 연동까지 담당합니다.

저 역시 커리어가 바뀔 때마다 업무 범위가 달라졌습니다.  
Data Analyst로 시작해 Vision AI 모델을 직접 학습했고,  
제품화 조직에서는 모델 검증과 최적화를 맡았습니다.  
이후에는 분석엔진과 시스템 연동까지 업무 범위가 넓어졌습니다.

돌아보면 직무명보다 중요한 변화는 따로 있었습니다.  
직접 다루고 책임지는 범위가 Data에서 Model,  
그리고 Product와 System까지 넓어졌다는 점입니다.

```text
Data
  ↓
Model Training
  ↓
Validation / Optimization
  ↓
Inference
  ↓
Product Integration
```

이 글에서는 AI 직무를 일반적인 기준으로 분류하기보다,  
제가 현재의 Applied AI Engineer 역할에 도달한 과정을 정리합니다.

---

## 1. Data Analyst — 데이터 품질과 자동화에서 시작

AI 관련 첫 직무는 Data Analyst였습니다.  
상경계열 전공 후 사업관리 조직에서 데이터 구축 과정을 관리했습니다.  
조직에 개발자가 없어 반복 작업을 줄이는 Python 도구도 직접 만들었습니다.

주로 Pandas와 Jupyter로 다음 작업을 자동화했습니다.

- 이미지 Resize / Rename / 비식별화
- 데이터 상태 점검과 결과 시각화
- Slack 알림과 일일 리포트 생성
- 공정별 진행 현황과 잔여 물량 관리

수집부터 정제, 가공, 라벨링, 학습 테스트, 검수까지의 작업은,  
하나의 FTP 서버에서 처리되도록 흐름을 구성하고 운영했습니다.  
공정별 잔여 물량과 품질을 기준으로 KPI 달성을 위한 보완 계획도 세웠습니다.

모델 개발은 공식 R&R이 아니었지만,  
데이터 품질 검증을 위해 직접 라벨링한 데이터로 학습을 진행했습니다.  
목적은 새 모델이 아니라 데이터셋이 학습에 적합한지 확인하는 것이었습니다.

학습 결과가 기대에 못 미칠 때 Dataset과 Annotation부터 확인하는 습관은,  
이 시기부터 이어진 것입니다.

---

## 2. AI Engineer — 모델 학습과 성능 검증

이후 AI Engineer로 이동해 Vision AI 모델 학습을 중심으로 일했습니다.  
실무와 함께 AI 관련 대학원 과정을 병행했습니다.  
주로 Detection, Segmentation, Tracking Task를 다뤘습니다.

프로젝트 목적과 데이터 특성에 맞는 모델을 선정하고,  
Architecture와 Config, 학습 조건을 바꿔가며 결과를 비교했습니다.

```text
Dataset
  ↓
Model Selection
  ↓
Training
  ↓
Evaluation
  ↓
Data / Config 수정
  ↓
Re-training
```

필요하면 데이터 추가 수집과 정제·라벨링까지 직접 진행했습니다.  
학습 후에는 정량 지표와 추론 결과 시각화로 성능을 확인했습니다.  
데이터 구축 경험 덕분에 문제가 생기면 데이터 단계까지 직접 확인했습니다.

당시의 관심사는 다음 한 가지였습니다.

> 어떤 모델과 학습 조건이 주어진 데이터에서 가장 좋은 결과를 만드는가?

제품화 조직으로 이동하면서 모델을 평가하는 기준이 달라지기 시작했습니다.

---

## 3. Applied AI Engineer — 모델의 실제 적용 가능성을 검증하다

제품화 조직에서 Applied AI Engineer라는 직함이 새로 만들어졌고,  
그 첫 담당자로 합류했습니다.  
연구소에서 전달받은 모델을 검증하고,  
실제 시스템에 적용할 수 있는지 판단하는 것이 핵심 업무였습니다.

정량 성능이 높아도 실제 환경에서 그대로 쓸 수 있는 것은 아니었습니다.  
실제 입력에서의 오탐·미탐과 추론 속도,  
GPU Memory와 Resource 조건을 함께 확인해야 했습니다.

주요 업무는 다음과 같았습니다.

- 모델의 정량·정성 성능 검증
- 실제 환경을 고려한 테스트셋 구성
- 오류 사례 분석과 추가 학습
- Model Optimization 및 경량화
- FPS, Latency, GPU Memory 확인
- 분석엔진 적용 가능 여부 검토
- PoC 및 기능 시연을 위한 모델 개발·검증

전달받은 모델이 요구사항을 만족하지 못하면 원인을 찾아 추가 학습했습니다.  
필요한 모델이 준비되지 않은 경우에는,  
데이터 구성부터 학습, 검증, 최적화까지 직접 수행했습니다.

모델 개발과 검증의 R&R이 명확하지 않던 시기였기 때문에,  
연구소와 제품화 팀 사이에서 요구사항과 일정을 조율하는 역할도 맡았습니다.

이 역할에서 받는 질문은 결국 하나로 모였습니다.

> "돼요?" / "안 돼요?"

이 질문에 답하려면 정확도뿐 아니라 실행 조건까지 함께 봐야 했습니다.

```text
Model Accuracy
       +
Latency / Throughput
       +
GPU Resource
       +
실제 입력에서의 오류
       ↓
적용 가능 여부 판단
```

처음에는 "안 돼요"였던 모델을,  
추가 학습과 최적화로 "되게" 만드는 것이 이 역할의 핵심이었습니다.

### 사례: 운영 환경에 맞춘 모델 크기 조정

화학공장 시운전에서는 오탐 데이터를 반영해 YOLOv12L로 모델을 갱신했습니다.  
하지만 현장 시스템에서 OOM(Out-Of-Memory)이 발생해 운용이 어려웠습니다.  
모델 크기와 추론 효율을 고려해 YOLOv12M으로 바꿔 재학습·최적화했고,  
이 모델을 분석엔진에 탑재해 운영 모델로 확정했습니다.

더 큰 모델이 항상 더 나은 선택은 아니라는 점을 확인한 사례였습니다.  
자세한 과정은 [Vision AI 제품 개발 과정 (3)](/vision-ai-product-dev-3/)에 정리했습니다.

---

## 4. End-to-End 범위까지 확장된 업무

이후에는 분석엔진과 시스템 연동까지 직접 확인하는 업무가 늘었습니다.  
Task 정의부터 실제 시스템에서의 결과 확인까지 하나의 흐름으로 다루게 됐습니다.

```text
Task / Requirement
        ↓
Data
        ↓
Model Training
        ↓
Validation / Optimization
        ↓
Inference
        ↓
Analysis Engine
        ↓
System Integration
```

분석엔진에서 문제가 생기면 원인이 모델 내부에만 있지는 않았습니다.  
입력 영상의 조건이나 Preprocessing이 달라졌을 수도 있고,  
모델 출력은 정상인데 Post-processing이나 Interface에서 잘못될 수도 있었습니다.

여러 채널을 동시에 분석하는 환경에서는,  
단일 영상에서 문제없던 모델도 처리 속도나 GPU 측면의 제약이 드러났습니다.  
그래서 모델 검증과 별도로 추론 로그와 시스템 동작 상태를 함께 확인했습니다.

이 과정에서 다음 영역까지 업무가 확대됐습니다.

- Docker 기반 실행 환경과 GPU Resource 관리
- Video Decode / Preprocessing
- Model Inference와 Post-processing
- 분석엔진의 실행 상태 및 성능 확인
- API와 기존 시스템 Interface
- 추론 로그 및 오류 분석
- 배포 환경에서의 실행과 연동 검증

주로 다룬 기술은 ONNX, TensorRT, Docker, FastAPI, Kafka, Elasticsearch입니다.

### 사례: 폐쇄망 AI 솔루션 배포

인터넷이 차단된 폐쇄망 환경에 AI 솔루션 전체를 배포하고 연동했습니다.  
VLM 기반 OCR 모델을 LoRA로 파인튜닝하고 FastAPI로 서빙했으며,  
분석엔진과 Kafka·Elasticsearch 검색 백엔드, 웹 서비스까지 연결했습니다.  
권한 문제, 브로커 연결 실패 같은 현장 이슈도 직접 추적해 해결했습니다.  
과정은 [폐쇄망 AI Solution 서비스 구성 및 배포](/deploying-air-gapped-ai-solution-services/)에 있습니다.

모든 시스템 구조를 처음부터 단독으로 설계했다는 의미는 아닙니다.  
기존 시스템과 다른 개발자의 담당 영역이 있었고, 필요할 때 협업했습니다.  
다만 모델이 실제로 어떻게 실행되고 결과가 전달되는지는 직접 이해해야 했습니다.

이 경험을 통해 모델의 학습 결과를 평가하는 것과,  
모델이 포함된 분석 시스템 전체를 검증하는 것은 다른 일임을 체감했습니다.

---

## 5. End-to-End 경험 이후에도 Applied AI Engineer인 이유

업무 범위는 Data에서 Model, Inference, Engine, System까지 넓어졌습니다.  
하지만 제가 전문성을 쌓고 싶은 영역은 따로 있습니다.  
바로 AI 모델 개발과 실제 서비스 적용을 연결하는 과정입니다.

End-to-End 범위를 경험한 뒤에는 모델이 적용되는 시스템의 조건도 함께 고려하게 됐습니다.  
예를 들어 모델을 다음 단계에 넘길 때는 Weight와 Config 외에도 확인할 것이 많습니다.

- Input / Output 형식
- Preprocessing / Post-processing 조건
- GPU Resource와 Memory 요구량
- Latency 및 Batch 처리 조건
- 주요 Failure Case
- Model Load / Unload와 실행 환경
- 기존 Interface와의 호환성

이런 정보는 모델을 분석엔진에 연결하고 운영 환경에서 검증할 때 직접 영향을 줍니다.

Backend나 Infra의 모든 영역을 같은 깊이로 다룰 필요는 없다고 생각합니다.  
대신 제가 맡은 모델이 앞뒤 단계와 어떻게 연결되는지 알고,  
필요한 부분까지 직접 확인할 수 있어야 합니다.

| 단계 | 주요 업무 | 확장된 범위 |
|---|---|---|
| Data Analyst | 데이터 수집·정제·품질 관리·자동화 | Dataset |
| AI Engineer | Model Training / Evaluation | Model |
| Applied AI Engineer | 성능 검증·최적화·제품 적용 | Model → Product |
| End-to-End 업무 | 분석엔진·Interface·시스템 연동 | Model → System |

제 강점은 AI 모델을 직접 다루면서도 학습에서 작업을 끝내지 않고,  
실제 사용 환경의 조건까지 확인해 적용 가능한 형태로 만드는 것입니다.  
그래서 현재 제 역할을 Applied AI Engineer라고 정의합니다.

---

## 6. 블로그에 기록하고 있는 작업들

이 블로그의 글들은 대부분 이러한 업무 과정에서 다룬 내용을 정리한 것입니다.

| 영역 | 관련 포스팅 |
|---|---|
| Model / Data | [PaddleOCR 기반 도메인 특화 OCR fine-tuning](/ocr-fine-tuning/) |
| PoC / Product | [PoC, 데모, 2025 AI EXPO 후기 - 기술 설계와 전시의 경계](/proof-of-concepts-demo-expo-review/) |
| Input / Vision System | [Vision AI 시스템을 위한 CCTV 설치 및 구성 실전 가이드](/cctv-for-vision-ai/) |
| Optimization / Engine | [Vision AI 제품 개발 과정 - (3) 모델 최적화 및 경량화, 분석엔진 개발, 시스템 배포 및 운영](/vision-ai-product-dev-3/) |
| Deploy / System | [폐쇄망 AI Solution 서비스 구성 및 배포](/deploying-air-gapped-ai-solution-services/) |
| VLM / Interface | [대규모 VLM 영상 추출 시스템의 프롬프트 최적화·구조적 한계](/vlm-custom-prompt-optimization/) |

Model Training, CCTV, Docker, Engine, VLM처럼 주제는 다르지만,  
모두 AI 모델을 실제 환경에 적용하면서 다룬 기술과 문제를 정리한 글입니다.

---

## 마무리

각 단계의 경험은 따로 떨어져 있지 않았습니다.  
데이터 품질 관리 경험은 모델 학습으로 이어졌고,  
학습 경험은 제품화 단계의 추가 학습과 성능 개선으로 이어졌습니다.  
시스템 연동을 경험한 뒤에는 적용 가능성을 판단하는 기준도 넓어졌습니다.

앞으로도 모델 학습과 검증 역량을 기반으로,  
추론 최적화와 운영 환경의 문제 해결까지 연결하는 엔지니어로 성장하고자 합니다.
