---
title: PaddleOCR 기반 도메인 특화 OCR fine-tuning
tags: [vision-ai, model, data]
toc: true
toc_sticky: true
post_no: 7
---

OCR(Optical Character Recognition, 광학 문자 인식)은 이미지나 스캔 문서 속의 문자를 식별해 디지털 텍스트로 변환하는 기술입니다.
단순히 글자를 읽는 기능처럼 보이지만, 실제 구현 과정에서는 다양한 글자 모양·배경·촬영 환경에 대응해야 하므로 
Detection(문자 영역 탐지)과 Recognition(문자열 인식)이라는 두 단계로 나누어 처리합니다.

초기에는 OCR이 이미지를 글자로 변환하는 기술이라고 생각했으나,  
실제 fine-tuning을 진행해 보니 Detection과 Recognition 구조 차이, 데이터 라벨 구조,  
그리고 환경설정의 까다로움까지 예상보다 깊이 있게 다뤄야 할 영역이 많았습니다.  

공개된 OCR 데이터셋은 대부분 일상적인 상황과 표준어 중심으로 구성되어 있어,  
특정 환경에서 자주 등장하는 표현이나 어휘를 충분히 반영하지 못하는 경우가 많습니다.  
이에 맞춰 한국어 확장 커스텀 사전부터 이미지 생성, 라벨링까지 직접 데이터셋을 제작하고  
PaddleOCR 기반으로 fine-tuning을 진행했습니다.

---

## Text Detection vs Text Recognition

| 구분      | Text Detection (문자 영역 탐지)                             | Text Recognition (문자열 인식)               |
| ------- | ----------------------------------------------- | ----------------------------------- |
| 역할  | 이미지/영상에서 문자가 있는 영역을 찾아 Bounding Box로 표시 | Detection이 찾아준 영역 내부의 문자 내용을 해석 |
| 출력  | 위치 정보(x, y, w, h 또는 polygon 좌표)                 | 문자열(예: "PADDLEOCR")                 |
| 난이도 | 다양한 각도·배경·조명에서 텍스트 영역 구분                        | 다양한 폰트·언어·왜곡·노이즈 대응                 |
| 예시  | 간판 위치 박스                                        | 박스 내부의 "COFFEE"                     |

> 대부분의 OCR 엔진은 Detection → Recognition 순서로 동작합니다.

---

## PaddleOCR vs EasyOCR

OCR 프레임워크 중 유명한 것은 크게 두 가지가 있습니다.  
EasyOCR은 설치와 사용이 쉽습니다. `pip install easyocr` 한 줄로 바로 시작할 수 있습니다.  
다만, 구조가 단일 파이프라인 형태라 Detection·Recognition을 개별적으로 튜닝하거나,  
End-to-End 구조를 쓰기에는 제약이 많습니다.

PaddleOCR은 Baidu에서 만든 PaddlePaddle 프레임워크 기반으로,  
Detection·Recognition·End-to-End 모델을 모두 제공합니다.  
모델 구조와 학습 설정을 세밀하게 조정할 수 있고, TensorRT/ONNX 변환도 지원합니다.  
단점은 설치가 복잡하고, 문서가 중국어 중심이라 처음 진입 장벽이 높습니다.

이번 프로젝트의 목적이 특정 도메인에서의 고정밀 fine-tuning이었기 때문에,  
구조적 유연성과 End-to-End 지원이 강점인 PaddleOCR을 선택했습니다.


| 항목         | PaddleOCR                                          | EasyOCR                       |
| ---------- | ------------------------------------------------------ | --------------------------------- |
| 개발사/배경 | Baidu 주도, PaddlePaddle 프레임워크 기반                        | Jaided AI 주도, PyTorch 기반          |
| 구조     | Detection / Recognition / (옵션) Classification 단계별 모듈화                           | 단일 파이프라인                          |
| 언어 지원  | 80+ 언어                                                 | 약 80여 개 (일부 언어는 인식률 낮음)           |
| fine-tuning  | 공식 문서와 예제 풍부, Detection·Recognition 각각 독립 fine-tuning 가능                    | 제한적, Recognition 쪽 fine-tuning 위주                |
| 속도/최적화 | 경량 모델(PP-OCR Mobile)부터 대형 모델까지 다양, ONNX/TensorRT 변환 용이 | 간단 설치 후 바로 사용 가능하지만, 경량화·배포 옵션 적음 |
| 커뮤니티   | 중국·아시아권에서 매우 활발, GitHub 스타 수 높음                           | 초기 진입은 쉬우나 유지보수 속도 느림             |
| 장점     | 산업 적용 사례 풍부, 대규모 커스터마이징 가능                             | 설치 간단, 빠른 프로토타이핑                  |
| 단점     | 초기 설정과 학습 곡선 있음                                        | 대규모·고성능 튜닝에는 제약                   |

---

## Paddle 계열 패키지

Paddle 계열은 이름이 비슷해서 처음 접하면 헷갈리기 쉽습니다.  

| 이름               | 역할                                   | 설치 방식                                                        | 비고                                                  |
| ---------------- | ------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| PaddlePaddle | 프레임워크 코어 (PyTorch, TensorFlow 같은 엔진) | `pip install paddlepaddle` 또는 `pip install paddlepaddle-gpu` | GPU 쓸 경우 CUDA 버전 맞춰서 GPU 빌드 설치 필수                   |
| PaddleX      | Paddle 기반 AutoML/GUI 툴킷              | `pip install paddlex`                                        | OCR 포함 여러 작업 지원하지만, 의존성 충돌 잦음. OCR fine-tuning에 필수 아님      |
| PaddleOCR    | OCR 전용 라이브러리 (GitHub 레포)             | `git clone https://github.com/PaddlePaddle/PaddleOCR.git`    | 학습·추론 전체 파이프라인 포함       |
| paddleocr    | 추론용 CLI/경량 패키지                       | `pip install paddleocr`                                      | `paddleocr --image_dir img.jpg` 같은 추론만 가능. 학습 코드 없음 |
| ppocr        | PaddleOCR 레포 안의 파이썬 패키지 네임스페이스       | 레포 클론 후 `pip install -e .`                                   | fine-tuning시 사용하는 학습/추론 핵심 코드                           |

---

## PaddleOCR fine-tuning 환경 구성 (Conda 기반)

PaddleOCR은 설치 순서를 잘못 잡으면 의존성 충돌이나 버전 불일치가 발생할 수 있습니다.  
아래 순서를 반드시 지키는 것이 안전합니다.

### 1. Conda 가상환경 생성
```bash
conda update -n base -c defaults conda
conda create -n paddle python=3.10
conda activate paddle
python -m pip install --upgrade pip setuptools wheel
```

### 2. PaddleOCR 설치 (순서 중요)

```bash
# GPU 환경 기준 (CUDA 11.8 예시)
pip install paddlepaddle-gpu==2.6.2 -f https://www.paddlepaddle.org.cn/whl/mkl/avx/stable.html

# CPU만 사용할 경우
# pip install paddlepaddle

# 추론만 쓸 경우
pip install paddleocr

# 학습 시에는 반드시 GitHub clone + 개발 모드 설치
git clone https://github.com/PaddlePaddle/PaddleOCR.git
cd PaddleOCR
pip install -r requirements.txt
pip install -e .

# (선택) ONNX 변환
pip install paddle2onnx
```

### 3. 학습 실행 예시

```bash
# REC 예시
python tools/train.py -c /workspace/idxkim/cfg/rec_svtrnet.yml

# E2E 예시
python tools/train.py -c /workspace/idxkim/cfg/e2e_r50_vd_pg.yml
```

- `-c` 옵션에는 PaddleOCR의 config 파일 경로를 넣어야 합니다.
- 커스텀 데이터셋을 쓰려면 원본 config를 복사 후 데이터 경로만 수정하여 사용합니다.

---

## 설치 시 자주 발생하는 오류

| 오류 상황                                          | 원인                                        | 해결 방법                                |
| ---------------------------------------------- | ----------------------------------------- | ------------------------------------ |
| `ModuleNotFoundError: No module named "ppocr"` | pip `paddleocr`만 설치 후 학습 시도               | GitHub clone + `pip install -e .` 실행 |
| CUDA 버전 불일치                                    | PaddlePaddle GPU 버전과 CUDA 환경 불일치          | PaddlePaddle 호환표 확인 후 맞는 버전 설치       |
| CPU/GPU 빌드 동시 설치                               | `paddlepaddle`와 `paddlepaddle-gpu` 둘 다 설치 | 하나만 남기고 제거                           |
| 의존성 충돌                                         | `paddlex` 등 불필요 패키지 설치                    | pip uninstall paddlex                |

### 설치 시 핵심 체크리스트

- 설치 순서: PaddlePaddle → PaddleOCR → 기타 유틸 순으로 설치
- 학습 목적: GitHub clone + pip install -e . 필수
- 추론 목적: pip paddleocr만 설치
- 문제가 꼬이면 가상환경을 새로 만드는 게 가장 빠름
- GPU 사용 시 반드시 CUDA 버전 호환 여부 확인 (공식 문서 참고)

---

## Docker 컨테이너 기반 학습 환경 구성

실무에서 모델 학습 시 이미 빌드된 분석엔진용 Docker 이미지(A 이미지)를 사용합니다.  
배포 환경과 동일하게 구성되어 있어, 추가 빌드 없이 바로 컨테이너를 생성할 수 있습니다.

이 방식은 학습 환경과 배포 환경이 완전히 일치하므로,  
모델을 분석엔진에 올릴 때 환경 호환성 문제를 최소화할 수 있습니다.  
필요에 따라 학습 전용 이미지(B 이미지)를 따로 사용해 실험한 뒤,  
완성된 모델만 분석엔진 환경에 반영하기도 합니다.

---

### 1. 배포 환경 기반에서 학습

- A 이미지: 실제 제품 배포 시 사용하는 분석엔진 컨테이너의 베이스 이미지
- 이 컨테이너에서 바로 학습·검증까지 진행
- 장점:
  - 학습 환경과 배포 환경이 동일
  - Ultralytics, SAM, PaddleOCR 등 다른 Vision AI 프레임워크와 공존 가능
  - 모델 검증, 최적화, 배포까지 한 컨테이너에서 처리 가능

---

### 2. 학습 전용 이미지에서 학습

- A 이미지와는 별도의 학습 전용 컨테이너
- 학습 완료 후 모델 가중치와 설정 파일만 A 이미지 환경에 반영
- 장점:
  - 환경 충돌 최소화
  - 실험적인 버전 테스트에 유리

---

### 컨테이너별 비교

| 항목 | A 이미지 (배포 환경 기반) | B 이미지 (학습 전용) |
| -- | ------------------ | ---------------- |
| 용도 | 학습 + 검증 + 최종 배포 | 학습 전용, 실험/대체 |
| 구성 | 분석엔진 + PaddleOCR + 기타 AI 프레임워크 | CUDA + PaddleOCR 최소 구성 |

> 학습 완료 모델을 A 이미지에 반영할 때는  
> 가중치 파일 + character_dict + config.yml 세트를 그대로 옮기는 것이 안전합니다.

---

## OCR 데이터셋 생성 과정

이번 프로젝트에서는 사전 제작한 한국어 확장 커스텀 사전을 활용하여 도메인 맞춤형 한글 OCR 데이터셋을 직접 생성했습니다.
실제 서비스 환경에서 발생할 수 있는 색상 대비, 폰트 다양성, 배치, 초성 분포, 회전 요소를 반영하여,
모델이 다양한 상황에서도 안정적으로 동작할 수 있도록 설계했습니다.

---

### 1. HSV 기반 배경/전경 색상 선택

OCR 모델은 배경과 글자색의 대비가 낮으면 인식률이 떨어집니다.  
RGB 색상값을 무작위로 지정하는 방식 대신 HSV 색공간에서 색상을 생성하고,  
상대 휘도(luminance) 기반으로 최소 대비 기준을 충족하는 전경색을 선택했습니다.

- 목적: 글자와 배경이 충분히 구분되도록 하여 가독성을 확보  
- 구현 방식:  
  1. HSV 범위에서 무작위 색상 생성
  2. 배경 휘도 값에 따라 전경색 후보 범위를 다르게 설정
  3. 대비 비율이 `threshold` 이상인 색상만 채택
- 효과: 색상 다양성은 유지하면서도, 글자가 묻히는 경우를 방지

```python
import colorsys
import random

def hsv_rand(h, s, v):
    hh, ss, vv = random.uniform(*h), random.uniform(*s), random.uniform(*v)
    r, g, b = colorsys.hsv_to_rgb(hh, ss, vv)
    return tuple(int(255*x) for x in (r, g, b))

def calculate_luminance(rgb):
    r, g, b = [c/255 for c in rgb]
    return 0.2126*r + 0.7152*g + 0.0722*b

def contrast_ratio(bg, fg, thresh=2.0):
    L1, L2 = calculate_luminance(bg), calculate_luminance(fg)
    return (max(L1, L2)+0.05) / (min(L1, L2)+0.05) >= thresh
```

---

### 2. 폰트 크기, 외곽선, 굵기 랜덤화

한글 OCR에서는 글자 크기와 스타일 변화에 대응할 수 있는 학습이 중요합니다.
고정된 스타일만 학습하면 실제 환경에서 작은 글자나 얇은 글씨를 잘 인식하지 못합니다.

- 목적: 다양한 글자 크기·굵기·외곽선 스타일에 대응  
- 구현 방식:  
  - 폰트 크기: 40~100px 범위에서 랜덤
  - 외곽선(stroke): 두께(0~3px)와 색상 랜덤
- 효과: 폰트와 크기가 바뀌어도 인식률 유지

```python
from PIL import ImageFont
font_size = random.randint(40, 100)
font = ImageFont.truetype(FONT_PATH, font_size)
stroke_w = random.randint(0, 3)
```

---

### 3. 초성 기반 Stratified Split

데이터셋을 학습/검증 세트로 나눌 때, 문자 분포가 불균형하면 성능 검증이 왜곡될 수 있습니다.  
특히, 자주 등장하지 않는 초성이 검증 세트에서 전부 빠지면 모델이 그 문자를 학습하지 못합니다.

- 목적: 모든 초성이 학습·검증에 고르게 분포하도록 유지
- 구현 방식:
  - 각 단어의 초성을 추출하여 그룹화
  - 그룹별로 무작위 분할
- 효과: 검증 정확도의 신뢰도 향상

```python
CHO = list("ㄱㄲㄴㄷㄸㄹㅁㅂㅃㅅㅆㅇㅈㅉㅊㅋㅌㅍㅎ")
def get_choseong(ch):
    return CHO[(ord(ch)-ord("가"))//(21*28)] if "가" <= ch <= "힣" else "기타"
```

---

### 4. Detection/Recognition 데이터 동시 생성

OCR 파이프라인은 문자 영역 탐지(Detection)와 문자열 인식(Recognition) 두 단계로 구성됩니다.
합성 단계에서 두 작업에 필요한 데이터를 동시에 생성하면 효율성이 높아집니다.

- 목적: 한 번의 합성으로 두 작업의 학습 데이터를 모두 확보
- 구현 방식:
  - Detection 라벨: polygon 좌표 + 텍스트
  - Recognition 라벨: crop 이미지 + 텍스트
- 효과: 데이터 생성 효율 향상, 두 태스크 간 일관성 유지

---

### 5. 회전 + 직사각형 crop 방식

데이터 생성 시 이미지와 라벨 좌표를 동시에 회전하여 augmentation 효과를 주었습니다.
Recognition crop은 회전된 polygon 그대로 잘라내는 사다리꼴이 아니라,
회전된 polygon을 감싸는 axis-aligned 직사각형을 잘라 저장합니다.

- 목적: 회전된 데이터에서도 라벨 불일치 없이 Recognition 입력을 일정하게 유지
- 구현 방식:
  1. 이미지와 polygon 좌표를 동일 각도로 회전
  2. polygon의 min/max 좌표를 이용해 직사각형 bbox 계산
  3. bbox 영역을 crop → 필요 시 배경 여백 포함

- 효과:
  - 기울어진 글자라도 모델 입력 형태가 일정
  - perspective 변환 없이 구현 단순화

```python
def crop_rotated_bbox(rotated_img, rotated_points):
    xs = [p[0] for p in rotated_points]
    ys = [p[1] for p in rotated_points]
    xmin, xmax = int(min(xs)), int(max(xs))
    ymin, ymax = int(min(ys)), int(max(ys))
    return rotated_img.crop((xmin, ymin, xmax, ymax))
```

---

### 6. 전체 설계 방향

1. 색상 대비 최적화 → HSV 기반 색상 생성 + 대비 계산으로 가독성 확보
2. 다양한 시각 스타일 대응 → 폰트 크기·굵기·외곽선 랜덤화
3. 문자 분포 균형 유지 → 초성 기반 Stratified Split
4. 멀티태스크 데이터 동기화 → Detection/Recognition 동시 생성
5. 좌표·입력 형태 안정성 → 회전은 데이터 생성 단계에서 처리, Recognition은 직사각형 crop

---

## PaddleOCR 모델 유형 비교

| 유형 | 핵심 구조 | 장점 | 단점 | 주요 사용 케이스 |
|------|----------|------|------|------------------|
| Recognition (SVTR) | 잘라진 텍스트만 인식 | 데이터 준비 용이, 빠른 실험 | Detection 품질 의존 | 빠른 프로토타이핑 |
| Detection (DB) | 텍스트 위치 감지 | Recognition 품질 개선, 복잡 배경 강점 | 라벨 품질 중요, 학습 시간 증가 | 서비스 배포 전 탐지 성능 강화 |
| End-to-End (PGNet) | 감지+인식 동시 학습 | 상호 최적화, 속도·정확도 모두 확보 | 학습 난이도·데이터 품질 요구 높음 | 고정밀 OCR 서비스 |

---

### 1. Recognition(SVTR)

텍스트 크롭 이미지만을 이용해 SVTR 기반 Recognition 모델을 fine-tuning했습니다.
이 방식은 데이터 준비가 비교적 간단하고, 빠르게 실험을 시작할 수 있다는 장점이 있습니다.
다만 Detection 단계에서 텍스트 위치를 잘못 잡으면, 이후 Recognition 성능이 제한되는 한계가 있습니다.
복잡한 배경이나 다양한 배치 구조에서는 Detection 오류가 누적되어 전체 인식률이 떨어질 수 있습니다.

- 장점  
  - 데이터 준비와 파이프라인 구성이 단순합니다.  
  - 빠른 성능 검증과 파라미터 튜닝에 유리합니다.

- 단점  
  - Detection 품질에 의존도가 높습니다.  
  - 실제 서비스 환경의 복잡한 레이아웃에는 취약합니다.

---

### 2. Detection(DB)

텍스트 위치를 더 정확하게 찾기 위해 DB(Differentiable Binarization) 기반 Detection 모델을 fine-tuning했습니다.
DB는 속도와 정확도의 균형이 좋아 PaddleOCR에서 가장 널리 사용되는 Detection 구조이며,
이번 fine-tuning에서는 텍스트 탐지 정확도를 높여 Recognition 입력 품질을 개선하는 데 집중했습니다.
Detection 품질이 향상되면서 Recognition 단계의 오류도 크게 줄어들었습니다.

- 장점  
  - Recognition 입력 품질이 눈에 띄게 개선됩니다.  
  - 복잡한 레이아웃이나 다양한 배경 조건에서 강해집니다.  
  - 경량·고정밀 버전 모두 지원해 환경에 맞게 선택할 수 있습니다.

- 단점  
  - 박스 라벨링 품질이 성능에 직접적인 영향을 미칩니다.  
  - 학습 시간이 늘어날 수 있습니다.

---

### 3. End-to-End(PGNet)

Detection과 Recognition을 동시에 학습하는 PGNet 기반 End-to-End 구조를 적용했습니다.
Backbone과 Neck은 공개 checkpoint를 재사용하고, Head는 분리하여 커스텀 사전에 맞춰 새로 학습했습니다.
이 방식은 두 모듈이 상호 보완적으로 최적화되므로 전체 인식 정확도를 극대화할 수 있습니다.
다만 학습 난이도가 높고, 데이터셋 품질 관리와 하이퍼파라미터 설정에 더 많은 노력이 필요합니다.

- 장점  
  - Detection과 Recognition이 상호 최적화됩니다.  
  - 파이프라인 전체의 추론 속도와 일관성이 높아집니다.  
  - 라벨링 일관성이 유지되면 재학습 효율이 좋습니다.

- 단점  
  - 학습 난이도가 높고, 데이터셋 품질 요구 수준이 높습니다.  
  - 모델 구조 변경이나 Head 재학습 과정에서 오류 가능성이 증가합니다.

---

## Head를 분리해야 하는 이유

PaddleOCR에서 fine-tuning을 진행할 때 안정적인 학습을 위해 Head를 분리합니다.  
아래에서는 Head를 분리하는 이유와, 적용 시 함께 고려해야 할 사항을 정리했습니다.

---

### 1. Ultralytics와 PaddleOCR의 checkpoint 로딩 구조 비교

Ultralytics YOLO 계열은 `pretrained=True` 상태에서 클래스 수를 변경하면, Head 부분이 자동으로 재초기화되거나 `strict=False` 옵션을 통해 shape가 맞지 않는 키를 건너뛸 수 있습니다.
이 방식은 Backbone과 Neck만 그대로 두고 Head는 새로 구성되므로, 모델 구조 변경 시 비교적 유연하게 대처할 수 있습니다.

반면 PaddleOCR의 PGNet은 기본 loader가 checkpoint 전체를 strict하게 읽습니다.
Backbone과 Neck만 로드하는 별도의 스위치가 제공되지 않기 때문에, Head를 분리하지 않으면 shape mismatch 오류가 발생할 가능성이 있습니다.

---

### 2. OCR에서 Head의 특성

OCR의 Head는 문자 집합(character_dict)과 시퀀스 디코딩 방식에 종속됩니다.  
라벨 구성 차이가 있으면 출력 차원 불일치, 분포 불일치로 인해 학습이 불안정해질 수 있습니다.

---

### 3. Head를 분리해야 하는 이유

1. 라벨 공간 불일치  
   - 공개 checkpoint는 영문/범용 데이터 기준으로 학습된 경우가 많습니다. 
   - 이번처럼 한국어 확장 사전을 쓸 경우 기존 Head는 맞지 않을 수 있습니다.

2. 전이학습 단위 차이  
   - Backbone/Neck은 시각 특징을 추출하므로 재사용 가치가 높지만,   
   - Head는 문자 클래스 분포와 언어적 패턴을 반영하므로 데이터셋에 맞춘 재학습이 필요합니다.

3. 수렴 안정성  
   - 불일치 Head를 사용하면 초기 학습에서 불필요한 gradient를 소모하고 불안정해질 수 있습니다. 
   - 출력 차원을 맞춘 Head로 시작하면 적은 데이터로도 빠르게 수렴할 수 있습니다.

---

### 4. checkpoint 변환 필요성

- PaddleOCR의 사전학습 checkpoint는 `backbone.`, `neck.`, `head.` 네임스페이스로 파라미터가 구분됩니다.  
- Head를 재학습하고 Backbone/Neck만 전이학습하려면 checkpoint 변환이 필요합니다.  
- 변환 시 Head 파라미터를 제거하고 Backbone/Neck만 남긴 새로운 `.pdparams` 파일을 생성해야 합니다.

---

### Head 처리 방식 변경 과정에서 겪은 이슈

처음에는 다음과 같은 방법을 시도했습니다.
- PaddleOCR 내부 loader를 수정하여 `strict=False`를 적용
- 학습 직전에 state dict에서 Head 부분만 제거하는 커스텀 로직 삽입

그러나 다음과 같은 문제가 반복적으로 발생했습니다.
- Head 관련 shape mismatch / missing key / unexpected key 오류 발생  
- 학습 로그에 Head가 scratch(랜덤 초기화)로 붙었다는 메시지가 출력되며 성능 하락  
- 분산 학습 환경, 재시작, 버전 차이에 따라 결과가 재현되지 않음

결국, pretrained checkpoint를 Backbone/Neck 전용으로 변환해 사용하는 것이 가장 안정적이고 재현성이 높았습니다.

---

## 해결 방법

아래 스크립트를 사용하면 Head 네임스페이스를 제거한 새로운 checkpoint를 생성할 수 있습니다.  
변환된 파일을 YAML 설정의 `Global.pretrained_model` 경로에 지정하여 사용합니다.

```python
# filter_ckpt.py 
import sys 
import paddle

def filter_head(in_path, out_path, drop_prefixes=("head.",)):
    # PaddleOCR의 .pdparams 로드
    state_dict = paddle.load(in_path)

    # head.* 키 제거
    kept = {k: v for k, v in state_dict.items()
            if not any(k.startswith(p) for p in drop_prefixes)}

    # 저장
    paddle.save(kept, out_path)
    print(f"[filter] saved -> {out_path} "
          f"(kept={len(kept)}, dropped={len(state_dict)-len(kept)})")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print(f"Usage: python {sys.argv[0]} <input.pdparams> <output.pdparams>")
        sys.exit(1)

    src, dst = sys.argv[1], sys.argv[2]
    filter_head(src, dst)
```

실행 예시:

```bash
python filter_ckpt.py \
  /workspace/idxkim/pretrained/best_accuracy.pdparams \
  /workspace/idxkim/pretrained/backbone_neck_only.pdparams
```

YAML 설정:

```yaml
Global:
  pretrained_model: /workspace/idxkim/pretrained/backbone_neck_only.pdparams
```

Head 제거 확인:

```python
import paddle
st = paddle.load("/workspace/idxkim/pretrained/backbone_neck_only.pdparams")
assert all(not k.startswith("head.") for k in st.keys())
print("✅ head.* 없음")
```

---

## 마무리

이번 프로젝트에서 PaddleOCR 기반 fine-tuning을 통해 도메인 특화 OCR 성능을 개선했습니다.
데이터셋 설계와 checkpoint 관리가 학습 안정성과 성능에 미치는 영향을 검증했습니다.

실험 결과, 다음과 같은 접근이 효과적이었습니다.

- 환경 설정 및 종속성 설치를 표준화하여 재현성 확보  
- Head를 분리하고 Backbone/Neck 전용 checkpoint를 구성해 안정적으로 전이학습 진행  
- Recognition → Detection → End-to-End 순으로 단계적 fine-tuning 적용  
- 실험 로그, 하이퍼파라미터, 환경 버전을 체계적으로 기록해 재실행 가능성 보장  

위 절차를 통해 복잡한 OCR 프레임워크에서도 안정적인 학습 파이프라인을 구성할 수 있었습니다. 
또한, 데이터셋 품질과 checkpoint 전략이 OCR 성능 향상에 직결됨을 확인했습니다.





