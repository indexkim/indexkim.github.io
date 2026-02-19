---
title: 폐쇄망 AI Solution 개발환경 구축 (2)
tags: [docker, infra, os, system]
toc: true
toc_sticky: true
post_no: 9
---


이 글은 폐쇄망 AI Solution 개발환경 구축의 두 번째 편입니다.   
RHEL 9을 기준으로 하지만, 다른 Linux 배포판에서도 동일한 방식으로 적용할 수 있습니다.

> 특정 조직의 내부 시스템이나 정책, 네트워크 설정은 포함하지 않으며  
> 모든 내용은 공개된 기술과 일반적인 Linux 환경을 기반으로 구성되었습니다.

---


## 10. NVIDIA 드라이버 설치

NVIDIA 드라이버는 GPU 하드웨어를 제어하는 커널 모듈입니다.  
NVIDIA 드라이버가 없으면 `nvidia-smi` 명령이 작동하지 않고, CUDA를 사용할 수 없습니다.  
NVIDIA Container Toolkit을 사용하는 경우 호스트에는 NVIDIA 드라이버만 설치합니다.

### 1. GPU 장치 인식 확인

시스템에서 NVIDIA GPU가 정상적으로 인식되는지 확인합니다.  
결과 미출력 시 BIOS에서 PCI 장치를 활성화하거나 슬롯 연결을 점검합니다.

```bash
lspci | grep -i nvidia
```

---

### 2. 기존 드라이버 제거

기존 NVIDIA 관련 드라이버나 `nouveau` 모듈이 있다면 제거합니다.  
`nouveau`는 기본 커널 모듈로, 공식 NVIDIA 드라이버(`nvidia.ko`)와 충돌을 일으킬 수 있습니다.

```bash
sudo dnf remove -y '*nvidia*'
sudo dnf remove -y xorg-x11-drv-nouveau
```

---

### 3. nouveau 블랙리스트 설정

부팅 시 `nouveau` 모듈이 자동 로드되지 않도록 차단합니다.  
`modeset=0` 설정은 X11 초기화 시 프레임버퍼 점유를 방지합니다.

```bash
sudo bash -c 'cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF'
```

생성되는 파일 경로:

```
/etc/modprobe.d/blacklist-nouveau.conf
```

---

### 4. 커널 빌드 도구 설치

NVIDIA 드라이버는 커널 모듈을 직접 빌드하기 때문에 버전에 맞는 개발 도구가 필요합니다.  
`uname -r` 명령으로 커널 버전을 확인하고, `kernel-devel` 패키지의 버전이 일치해야 합니다.

`gcc`는 GNU Compiler Collection의 약자로, C/C++ 코드를 컴파일하여 실행 가능한 바이너리로 만드는 컴파일러입니다. 
NVIDIA 드라이버는 내부적으로 C 코드로 작성된 커널 모듈을 빌드하므로, `gcc`가 설치되어 있어야 설치 과정이 정상적으로 완료됩니다.

```bash
sudo dnf install -y gcc kernel-devel kernel-headers
uname -r
rpm -q kernel-devel
```

---

### 5. initramfs 재생성

`initramfs`는 부팅 시 커널이 로드하는 임시 루트 파일시스템입니다.  
이전에 포함된 `nouveau.ko`를 제거하기 위해 새로 생성합니다.  

```bash
sudo dracut --force
```

---

### 6. X11 종료 및 CLI 모드 전환

GUI(X11, GDM)가 실행 중이면 드라이버 설치가 실패할 수 있습니다.  
CLI 모드로 전환하여 그래픽 세션을 종료합니다.

```bash
sudo systemctl set-default multi-user.target
sudo systemctl isolate multi-user.target
```

---

### 7. 시스템 재부팅

재부팅 후 다음 명령으로 `nouveau`가 비활성화되었는지 확인합니다.

```bash
sudo reboot
```

출력 결과가 없으면 성공적으로 비활성화된 것입니다.

```bash
lsmod | grep nouveau
```

---

### 8. NVIDIA 드라이버 설치

#### (1) RHEL 9 환경용 `.rpm` 드라이버 설치

NVIDIA 공식 사이트에서 `nvidia-driver-local-repo-rhel9-580.*.rpm` 패키지를 사용해  
로컬 리포지토리를 구성한 뒤 설치할 수 있습니다.

이 파일은 폐쇄망 환경에서도 동작하는 오프라인 설치용 패키지입니다.  
설치 후 자동으로 `/var/nvidia-driver-local-repo-rhel9-580.*` 경로에 드라이버 파일이 풀리고,  
`/etc/yum.repos.d/`에 로컬 리포지토리가 등록됩니다.

설치 절차는 다음과 같습니다.

```bash
# 1. local repo 패키지 설치
sudo rpm -ivh nvidia-driver-local-repo-rhel9-580.95.05-1.0-1.x86_64.rpm

# 2. 리포지토리 등록 갱신
sudo dnf clean all
sudo dnf repolist

# 3. 드라이버 설치
sudo dnf install -y nvidia-driver
```

---

#### (2) `.run` 파일 방식 설치 

`.run` 파일 방식으로도 설치할 수 있습니다.  
이 방식은 커널 모듈을 직접 빌드하므로 인터넷 연결이 필요하지 않습니다.  

설치 절차는 다음과 같습니다.

```bash
# 1. 실행 권한 부여
chmod +x NVIDIA-Linux-x86_64-580.95.05.run

# 2. 드라이버 설치
sudo bash NVIDIA-Linux-x86_64-580.95.05.run --no-cc-version-check
```

`--no-cc-version-check` 옵션은 GCC 버전 불일치 시에도 설치가 중단되지 않도록 하는 옵션입니다.  
이 방식은 로컬 리포지토리 구성 없이 바로 설치할 수 있어 간편합니다.  

---

#### (3) 설치 확인

드라이버 설치가 완료되면 시스템을 재부팅한 뒤,
GPU 인식 여부를 확인합니다.

```bash
sudo reboot
```

재부팅 후 다음 명령을 실행합니다.

```bash
nvidia-smi
```

정상적으로 설치되었다면 다음과 같은 출력이 표시됩니다.

```text
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.95.05              Driver Version: 580.95.05      CUDA Version: 13.0     |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA B300 SXM6 AC            On  |   00000000:1A:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   1  NVIDIA B300 SXM6 AC            On  |   00000000:3C:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   2  NVIDIA B300 SXM6 AC            On  |   00000000:62:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   3  NVIDIA B300 SXM6 AC            On  |   00000000:73:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   4  NVIDIA B300 SXM6 AC            On  |   00000000:9A:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   5  NVIDIA B300 SXM6 AC            On  |   00000000:BC:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   6  NVIDIA B300 SXM6 AC            On  |   00000000:DF:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
|   7  NVIDIA B300 SXM6 AC            On  |   00000000:F0:00.0 Off |                    0 |
| N/A   27C    P0            139W / 1100W |       0MiB / 275040MiB |      0%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
```

`Driver Version`과 `CUDA Version`이 정상적으로 표시되면
드라이버 설치가 완료된 것입니다.

---

### 9. 그래픽 모드 복원 및 확인


CLI 모드에서 설치를 완료한 경우,  
GUI 환경이 필요하다면 다음 명령으로 그래픽 모드를 복원합니다.


```bash
sudo systemctl set-default graphical.target
sudo reboot
```

---

### .run 방식과 .rpm 방식 비교

| 구분    | `.run` 방식        | `.rpm` 방식          |
| ----- | ---------------- | ------------------ |
| 설치 방식 | 커널 모듈 직접 빌드      | 패키지 관리 기반          |
| 장점    | 단일 파일로 간편        | Red Hat 지원, 시스템 통합 |
| 단점    | 커널 업데이트 시 재설치 필요 | 의존성 파일을 모두 준비해야 함  |
| 권장 환경 | 오프라인 설치 환경    | 표준화된 기업 환경         |

---

### 전체 절차 요약

```bash
# 1. 기존 드라이버 및 nouveau 제거
sudo dnf remove -y '*nvidia*'
sudo dnf remove -y xorg-x11-drv-nouveau

# 2. 블랙리스트 등록
sudo bash -c 'cat <<EOF > /etc/modprobe.d/blacklist-nouveau.conf
blacklist nouveau
options nouveau modeset=0
EOF'

# 3. 커널 빌드 도구 설치
sudo dnf install -y gcc kernel-devel kernel-headers

# 4. initramfs 재생성
sudo dracut --force

# 5. CLI 모드 전환 후 재부팅
sudo systemctl set-default multi-user.target
sudo reboot

# 6-1. NVIDIA 드라이버 설치 (rpm 방식)
sudo rpm -ivh nvidia-driver-local-repo-rhel9-580.95.05-1.0-1.x86_64.rpm
sudo dnf clean all
sudo dnf install -y nvidia-driver

# 6-2. NVIDIA 드라이버 설치 (run 방식)
chmod +x NVIDIA-Linux-x86_64-580.95.05.run
sudo bash NVIDIA-Linux-x86_64-580.95.05.run --no-cc-version-check

# 7. 그래픽 모드 복귀 및 확인
sudo systemctl set-default graphical.target
sudo reboot
```

> `nouveau` 비활성화 → `initramfs` 재생성 → CLI 전환 → 드라이버 설치 순서를 지켜야 함  
> 6단계에서 `.rpm` 방식(6-1) 또는 `.run` 방식(6-2) 중 선택

---

## 11. NVIDIA Container Toolkit 설치

`NVIDIA Container Toolkit`은 컨테이너의 GPU 인식을 지원하는 핵심 구성 요소입니다.  
미설치시 컨테이너 내부에서 `nvidia-smi` 명령이 동작하지 않으며,  
AI 모델 학습이나 추론 시 GPU 가속을 사용할 수 없습니다.

---

### 1. 구성 요소

NVIDIA Container Toolkit은 다음 패키지들로 구성됩니다.

| 구성 요소                      | 역할                                                 | 비고                       |
| -------------------------- | -------------------------------------------------- | ------------------------ |
| `libnvidia-container`      | GPU 장치 파일(`/dev/nvidia*`)과 드라이버 라이브러리를 컨테이너 내부로 전달 | GPU 접근 핵심                |
| `nvidia-container-runtime` | Docker 기본 런타임(`runc`)을 확장하여 GPU 지원 추가              | `--runtime=nvidia` 옵션 제공 |
| `nvidia-docker2`           | 과거 CLI 호환용 패키지                                     | 현재는 통합됨          |
| `nvidia-container-toolkit` | 위 구성 전체를 포함하는 최신 통합 패키지                            | RHEL 9 기준 사용             |

---

### 2. 설치 전제 조건

NVIDIA 커널 드라이버(`nvidia.ko`)가 이미 설치되어 있어야 합니다.  
설치하려는 NVIDIA Container Toolkit 버전은 드라이버 버전과 호환되어야 합니다.

드라이버 설치 여부 확인:

```bash
nvidia-smi
```

버전 호환성 예시:
- `driver 580.95.05` → `toolkit 1.18.0` 사용 가능
- `driver 550.78` → `toolkit 1.17.9` 사용 가능

---

### 3. 설치 절차

GitHub Release에서 직접 받은 `.rpm` 파일은 별도의 리포지토리 등록 없이  
`dnf install -y *.rpm` 명령어로 설치할 수 있습니다.  
의존성 순서가 있기 때문에 위 순서를 지키는 것이 안정적입니다.  

NVIDIA Container Toolkit 설치 후에는 재부팅이 필요하지 않으며,  
Docker 또는 Podman 런타임에서 GPU를 인식하도록 설정만 추가하면 됩니다.

```bash
# 1. GitHub Release에서 패키지 다운로드
# https://github.com/NVIDIA/nvidia-container-toolkit/releases
# (예: 1.18.0 버전 기준)
libnvidia-container1-1.18.0-1.x86_64.rpm
libnvidia-container-tools-1.18.0-1.x86_64.rpm
nvidia-container-toolkit-base-1.18.0-1.x86_64.rpm
nvidia-container-toolkit-1.18.0-1.x86_64.rpm

# 2. 설치 실행
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.18.0-1
sudo dnf install -y \
  libnvidia-container1-${NVIDIA_CONTAINER_TOOLKIT_VERSION}.x86_64.rpm \
  libnvidia-container-tools-${NVIDIA_CONTAINER_TOOLKIT_VERSION}.x86_64.rpm \
  nvidia-container-toolkit-base-${NVIDIA_CONTAINER_TOOLKIT_VERSION}.x86_64.rpm \
  nvidia-container-toolkit-${NVIDIA_CONTAINER_TOOLKIT_VERSION}.x86_64.rpm
```

---

### 4. 설치 확인

설치된 패키지 목록을 확인할 수 있습니다.  

```bash
rpm -qa | grep nvidia-container
```

출력 예시:

```text
libnvidia-container1-1.18.0-1.x86_64
libnvidia-container-tools-1.18.0-1.x86_64
nvidia-container-toolkit-base-1.18.0-1.x86_64
nvidia-container-toolkit-1.18.0-1.x86_64
```

NVIDIA Container Toolkit CLI 버전을 확인합니다.  

```bash
nvidia-ctk --version
```

출력 예시:

```text
NVIDIA Container Toolkit CLI version 1.18.0
```

정상적으로 설치되었다면 `/etc/nvidia-container-runtime/config.toml` 파일이 존재하며,  
NVIDIA Container Toolkit 설정이 적용된 상태입니다.

```bash
ls /etc/nvidia-container-runtime/
cat /etc/nvidia-container-runtime/config.toml | head -n 10
```

---

### 호스트 CUDA 설치에서 컨테이너 격리로: CUDA 설치가 필요 없어진 이유

과거에는 환경 설정이 딥러닝 학습의 최대 진입장벽 중 하나였습니다.  

Windows 환경에서는 CUDA Toolkit 설치 단계부터 Visual Studio 버전 제약으로 막히는 경우가 많았고,
OpenCV 같은 복잡한 라이브러리는 빌드 실패와 재시도를 며칠에 걸쳐 반복해야 했습니다.  
Ubuntu는 CUDA/cuDNN 설치가 비교적 수월했으나 OpenCV 빌드 시 WITH_QT, WITH_FFMPEG 같은 옵션 조합에서 충돌이 반복되었습니다.  

그러나 Docker 기반 워크플로우로 전환된 이후부터는 상황이 크게 달라졌습니다.  
호스트에는 드라이버만 설치하면 되고, CUDA/cuDNN은 모두 컨테이너에 포함되기 때문에  
환경 구성으로 인한 문제는 사실상 사라졌습니다.

#### 1. 호스트 직접 설치 방식 (conda)
- 호스트에 NVIDIA 드라이버 + CUDA Toolkit + cuDNN 설치 필요
- Windows의 경우 Visual Studio, CMake, Ninja 등 추가 개발 도구 필수
- OpenCV 소스 빌드 시 Qt/GTK GUI 옵션과 FFmpeg 의존성 충돌이 빈번
- CUDA/cuDNN/PyTorch 버전 조합을 맞추는 데 많은 시간 소모
- 프로젝트마다 다른 CUDA 버전을 쓰면 호스트 환경 충돌 발생
- 현재도 Jupyter 기반 로컬 개발, 빠른 프로토타이핑에는 유용함

#### 2. 컨테이너 방식 (Docker + NVIDIA Container Toolkit)
- 호스트에는 NVIDIA 드라이버만 설치하면 충분
- CUDA Toolkit, cuDNN, AI 프레임워크는 컨테이너 이미지에 포함
- 호스트는 GPU 하드웨어 제어만 담당
- 프로젝트마다 CUDA 버전이 달라도 컨테이너로 격리되어 충돌 없음
- 환경 재현성이 높고, 학습·배포·운영까지 동일한 구성 유지 가능
- 프로덕션·학습 환경에서는 사실상 표준

> 단, 호스트에서 직접 CUDA C++ 컴파일을 수행해야 하는 경우 CUDA Toolkit 설치 필요  
> → PyTorch C++ Extension, TensorRT 플러그인, FlashAttention·xFormers 등 고성능 CUDA 커널, NVDEC/NPP 기반 미디어 처리 등

---

## 12. Docker 설치 및 GPU 연동

Docker를 설치해 컨테이너 기반 개발 환경을 구성합니다.  
NVIDIA Container Toolkit과 연동하여 GPU를 사용할 수 있도록 설정합니다.

---

### 1. Docker 설치

사전에 다운로드한 `.rpm` 파일을 사용해 Docker를 설치합니다.

```bash
# Docker 설치
sudo dnf install -y ./containerd.io-*.rpm \
  ./docker-ce-*.rpm \
  ./docker-ce-cli-*.rpm \
  ./docker-buildx-plugin-*.rpm \
  ./docker-compose-plugin-*.rpm

# 서비스 등록 및 실행
sudo systemctl enable --now docker
sudo systemctl status docker
```

---

### 2. NVIDIA Container Toolkit 연동

NVIDIA Container Toolkit을 Docker 런타임에 연결하면  
컨테이너 내부에서도 GPU가 인식되어 CUDA 연산을 사용할 수 있습니다.

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

### 3. 사용자 권한 설정

현재 사용자를 `docker` 그룹에 추가해야 루트 권한 없이 명령을 실행할 수 있습니다.  
변경 사항은 새 터미널 세션에서 적용됩니다.  

```bash
sudo usermod -aG docker $USER
```

---

### 4. 컨테이너 생성 및 이미지 관리

폐쇄망 환경에서는 Docker Hub에 직접 접근할 수 없으므로,  
외부 인터넷 환경에서 이미지를 준비해 tar 파일 형태로 반입해야 합니다.  

이미지를 만드는 방법은 두 가지입니다.

1. Dockerfile 방식
2. `docker commit` 방식

두 방식 모두 최종적으로 `docker save`로 tar 파일을 생성해 반입합니다.

---

#### Dockerfile

가장 일반적이며 재현성과 협업·버전 관리에 유리합니다.  
동일 Dockerfile로 언제든지 같은 환경을 다시 만들 수 있고 Git으로 관리가 가능합니다.

```dockerfile
# Dockerfile
FROM pytorch/pytorch:2.4.0-cuda12.4

# 시스템 패키지 설치
RUN apt-get update && \
    apt-get install -y vim git wget && \
    rm -rf /var/lib/apt/lists/*

# Python 패키지 설치
RUN pip install --no-cache-dir numpy pandas scikit-learn opencv-python

# 작업 디렉터리 설정
WORKDIR /workspace
```

```bash
# 이미지 빌드
docker build -t idxkim_image:v0.1 .

# 이미지를 tar로 저장
docker save -o idxkim_image_v0.1.tar idxkim_image:v0.1
```

분석 엔진 개발, 프로덕션 배포, 팀 간 협업 환경에 적합하며  
CI/CD 파이프라인 구축에 활용됩니다.

---

#### docker commit

컨테이너 내부에서 직접 패키지를 설치하고,
그 실행 상태 그대로 스냅샷을 저장하는 방식입니다.
재현성은 Dockerfile보다 떨어지지만,
파인튜닝·실험·일회성 분석에 빠르게 활용할 수 있습니다.

```bash
# 1. 베이스 이미지 다운로드
docker pull pytorch/pytorch:2.4.0-cuda12.4

# 2. 컨테이너 실행 및 필요한 패키지 설치
docker run -it --name temp_container pytorch/pytorch:2.4.0-cuda12.4 /bin/bash
# (컨테이너 내부에서)
# apt-get update && apt-get install -y vim git wget
# pip install numpy pandas scikit-learn opencv-python
# exit

# 3. 커스텀 이미지로 저장
docker commit temp_container idxkim_image:v0.1
docker rm temp_container

# 4. 이미지를 tar로 저장
docker save -o idxkim_image_v0.1.tar idxkim_image:v0.1
```

모델 파인튜닝, 데이터 전처리 파이프라인 테스트, 라이브러리 조합 탐색 등  
일회성 분석 및 실험 환경에 활용됩니다.

---

#### 두 방식 비교

| 항목         | Dockerfile   | docker commit  |
| ---------- | ------------ | -------------- |
| 재현성        | 높음           | 낮음 (작업 기록 필요)  |
| 버전 관리      | Git 관리 가능    | 어려움            |
| 협업         | 용이           | 환경 차이 발생       |
| 자동화(CI/CD) | 적합           | 부적합            |
| 용도         | 엔진·API 서버·배포 | 실험·파인튜닝·일회성 작업 |

---

#### 폐쇄망 반입 방법

두 방식 중 무엇을 사용하든 tar 파일 형태로 반입합니다.
tar 파일 방식을 사용하는 이유는 다음과 같습니다.

- 외부에서 검증된 환경을 그대로 가져올 수 있음
- 폐쇄망 내부에서 재빌드할 필요 없음
- 의존성 충돌 없이 동일한 환경 보장
- 물리적 이동만으로 환경 복제 가능

```bash
# 외부 환경에서 이미지 검증 완료 후 저장
docker save -o idxkim_image_v0.1.tar idxkim_image:v0.1

# 폐쇄망 내부에서 즉시 사용
docker load -i idxkim_image_v0.1.tar
docker run --gpus all idxkim_image:v0.1  # 바로 실행 가능
```

#### commit 이미지에서 Dockerfile 복원하기

`docker commit`으로 만든 이미지를 Dockerfile로 되돌릴 수 있는지는  
원본 Dockerfile의 존재 여부에 따라 결정됩니다.

---

##### 원본 Dockerfile이 없는 경우

설치 패키지 정보를 기반으로 유사한 Dockerfile을 작성할 수 있지만 완전한 복원은 불가능합니다.

```bash
# 이미지에서 컨테이너 실행
docker run -it idxkim_image:v0.1 /bin/bash

# 시스템 패키지 목록 추출
apt list --installed > apt_packages.txt
dpkg -l > dpkg_list.txt

# Python 패키지 정확한 버전 추출
pip freeze > requirements.txt

# 설치된 CUDA 버전 확인
ls /usr/local/cuda*/bin/
which nvidia-smi
```

추출한 정보로 Dockerfile을 재작성합니다.

```dockerfile
FROM pytorch/pytorch:2.4.0-cuda12.4

# apt_packages.txt 참고하여 필요한 패키지만 선택
RUN apt-get update && \
    apt-get install -y \
    vim \
    git \
    wget \
    && rm -rf /var/lib/apt/lists/*

# requirements.txt 그대로 사용
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

WORKDIR /workspace
```

그러나 다음과 같은 정보는 복원할 수 없습니다.

- 설치 순서 및 의존성 설치 흐름
- 빌드 스크립트의 컴파일 옵션 (FFmpeg/OpenCV 빌드 플래그 등)
- 환경 변수 및 커스텀 스크립트
- 중간 레이어가 어떤 명령으로 만들어졌는지에 대한 정보

---

##### 원본 Dockerfile이 있는 경우

원본 Dockerfile과 빌드 스크립트를 보유하고 있다면,  
`docker commit`으로 만든 이미지든 tar 파일이든 상관없이  
언제든지 동일한 환경을 재생성할 수 있습니다.  

Dockerfile은 환경 구성의 설계도이므로,  
이를 보관하면 필요할 때마다 동일한 환경을 재현할 수 있습니다.

---

### 5. GPU 테스트

커스텀 이미지에 `nvidia-smi`와 PyTorch가 모두 포함되어 있으므로,  
하나의 이미지로 GPU 드라이버 및 CUDA 연동을 모두 확인할 수 있습니다.

```bash
# nvidia-smi로 GPU 드라이버 확인
docker run --rm --gpus all idxkim_image:v0.1 nvidia-smi

# PyTorch로 CUDA 가용성 확인
docker run --rm --gpus all idxkim_image:v0.1 \
  python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU count: {torch.cuda.device_count()}')"
```

GPU 정보가 정상적으로 출력되면 Docker와 NVIDIA 런타임 연동이 완료된 것입니다.

---

## 12-1. Docker 데이터 저장 경로 변경

Docker 설치 후 저장 경로(`/var/lib/docker`)를 변경해야 하는 경우에만 수행합니다.   
새 설치 시에는 `/etc/docker/daemon.json` 파일로 경로를 미리 지정하세요.

### 주의사항

- 레이어 손상 위험: 복사 중 중단되면 컨테이너 실행이 실패할 수 있습니다.
- 다운타임 발생: 모든 컨테이너를 중지한 상태에서 작업해야 합니다.
- 백업 권장: 중요한 컨테이너나 이미지는 사전에 백업해 두는 것이 안전합니다.
- SELinux 컨텍스트: 경로 변경 후 새 디렉터리에 SELinux 컨텍스트를 등록해야 합니다.  
  등록하지 않으면 이후 볼륨 마운트 시 권한 문제가 발생할 수 있습니다.

```bash
# 새 경로에 SELinux 컨텍스트 등록
sudo semanage fcontext -a -t container_var_lib_t "/home/docker(/.*)?"
sudo restorecon -Rv /home/docker
```
---

### 1. 데이터 디렉터리 이동

Docker 서비스를 중지하고 기존 데이터를 새 경로로 복사합니다.

```bash
sudo systemctl stop docker
sudo mkdir -p /home/docker
sudo rsync -aP /var/lib/docker/ /home/docker/
sudo vi /etc/docker/daemon.json
```

`/etc/docker/daemon.json` 수정 내용:

```json
{
    "data-root": "/home/docker",
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

---

### 2. 서비스 재시작 및 확인

Docker 데몬을 다시 로드하고 경로 변경을 확인합니다.

```bash
sudo systemctl daemon-reload
sudo systemctl start docker
docker info | grep "Docker Root Dir"
```

출력 예시:

```
Docker Root Dir: /home/docker
```

정상적으로 변경되면 기존 디렉터리를 제거합니다.

```bash
sudo rm -rf /var/lib/docker
```

---

## 12-2. 설치 전 데이터 경로 지정 (사전 설정 방식)

Docker 설치 전에 데이터 경로를 지정하는 방법입니다. 
이후 일반적인 Docker 설치 절차를 진행하면
초기부터 `/home/docker` 경로를 사용하게 됩니다.

```bash
sudo mkdir -p /home/docker
sudo mkdir -p /etc/docker
sudo bash -c 'cat <<EOF > /etc/docker/daemon.json
{
  "data-root": "/home/docker"
}
EOF'
```

---

## 12-3. 명령어 자동 완성 설치 (선택)

`docker` 및 `podman` 명령의 자동 완성이 활성화되어
명령 입력 효율이 향상됩니다.

```bash
sudo dnf install -y bash-completion
```

---

## 13. Podman 설치 및 GPU 연동

Podman은 데몬(daemon) 없이 동작하는 컨테이너 엔진입니다.  
데몬은 백그라운드에서 상시 실행되는 서비스 프로세스로,  
Docker는 `dockerd`라는 데몬을 통해 컨테이너를 관리하지만   
Podman은 명령 실행 시 즉시 컨테이너를 생성·종료합니다.  
따라서 Rootless 실행이 가능하고 보안성이 높습니다.

RHEL 9에서는 Podman이 기본 컨테이너 엔진으로 권장됩니다.

---

### Docker vs Podman 주요 차이점

| 구분        | Docker                         | Podman                         |
| --------- | ------------------------------ | ------------------------------ |
| GPU 옵션    | `--gpus all`                   | `--device nvidia.com/gpu=all`  |
| NVIDIA 연동 | `nvidia-ctk runtime configure` | `nvidia-ctk cdi generate`      |
| 데이터 경로    | `/var/lib/docker`              | `/var/lib/containers`          |
| 설정 파일     | `/etc/docker/daemon.json`      | `/etc/containers/storage.conf` |
| 권한        | `docker` 그룹 필요                 | Rootless 지원                    |


> 중요: GPU 사용 시 옵션이 다름  
> Docker: `docker run --gpus all <image>`  
> Podman: `podman run --device nvidia.com/gpu=all <image>`

---

### 1. Podman 설치

Podman은 별도의 데몬이 필요하지 않으며, 설치 직후 바로 사용 가능합니다.

```bash
# Podman 설치
sudo dnf install -y podman
```

---

### 2. NVIDIA Container Toolkit 연동

`cdi generate`는 Podman용 GPU 장치 매핑 설정 파일(`/etc/cdi/nvidia.yaml`)을 생성합니다.
Docker의 `runtime configure` 명령과 동일한 역할을 수행합니다.

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

---

### 3. 컨테이너 이미지 로드 및 실행

커스텀 이미지를 로드하고 GPU 연동을 테스트합니다.

```bash
# 커스텀 이미지 로드
sudo podman load -i idxkim_image_v0.1.tar

# nvidia-smi로 GPU 드라이버 확인
sudo podman run --rm -it \
  --device nvidia.com/gpu=all \
  idxkim_image:v0.1 \
  nvidia-smi

# PyTorch로 CUDA 가용성 확인
sudo podman run --rm -it \
  --device nvidia.com/gpu=all \
  idxkim_image:v0.1 \
  python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}'); print(f'GPU count: {torch.cuda.device_count()}')"
```

정상적으로 GPU 정보가 출력되면 Podman과 NVIDIA Container Toolkit이 연동된 것입니다.

---

### 4. 작업용 컨테이너 실행 예시

실제 개발 작업을 위한 컨테이너를 실행합니다.

```bash
sudo podman run -it --rm \
  --device nvidia.com/gpu=all \
  --shm-size=100g \
  -v /workspace:/workspace \
  idxkim_image:v0.1
```

---

## 13-1. Podman 데이터 저장 경로 변경

Podman 설치 후 저장 경로(`/var/lib/containers`)를 변경해야 하는 경우에만 수행합니다.  
새 설치 시에는 `/etc/containers/storage.conf` 파일로 경로를 미리 지정하세요.

### 주의사항

- 레이어 손상 위험: 복사 중 중단되면 컨테이너 실행이 실패할 수 있습니다.
- 다운타임 발생: 모든 컨테이너를 중지한 상태에서 작업해야 합니다.
- 백업 권장: 중요한 컨테이너나 이미지는 사전에 백업해 두는 것이 안전합니다.
- Rootless 모드: Rootless 환경은 `~/.local/share/containers` 경로도 함께 확인해야 합니다.

---

### 1. 데이터 디렉터리 이동

Podman 컨테이너를 중지하고 기존 데이터를 새 경로로 복사합니다.

```bash
sudo podman stop --all
sudo du -sh /var/lib/containers
sudo mkdir -p /home/podman
sudo rsync -aP /var/lib/containers/ /home/podman/
sudo vi /etc/containers/storage.conf
```

`/etc/containers/storage.conf` 수정 내용:

```ini
[storage]
driver = "overlay"
runRoot = "/run/containers/storage"
graphRoot = "/home/podman"

[storage.options]
additionalimagestores = []
```

---

### 2. 적용 및 확인

설정을 다시 로드하고 경로 변경을 확인합니다.

```bash
sudo systemctl daemon-reexec
sudo systemctl start podman
podman info | grep graphRoot
```

출력 예시:

```
graphRoot: /home/podman
```

정상적으로 변경되면 기존 디렉터리를 제거합니다.

```bash
sudo rm -rf /var/lib/containers
```

---

## 13-2. 설치 전 데이터 경로 지정 (사전 설정 방식)

Podman 설치 전에 데이터 저장 경로를 변경하는 방법입니다.  
이후 일반적인 Podman 설치 절차를 진행하면
초기부터 `/home/podman` 경로를 사용합니다.

```bash
sudo mkdir -p /home/podman
sudo mkdir -p /etc/containers
sudo bash -c 'cat <<EOF > /etc/containers/storage.conf
[storage]
driver = "overlay"
runRoot = "/run/containers/storage"
graphRoot = "/home/podman"

[storage.options]
additionalimagestores = []
EOF'
```

---

## 13-3. 명령어 자동 완성 설치 (선택)

`podman` 및 `docker` 명령 자동 완성이 활성화되어
명령 입력 효율이 향상됩니다.

```bash
sudo dnf install -y bash-completion
```

---

## 14. VSCode Remote-SSH 설치

폐쇄망 환경에서 VSCode를 이용해 원격 개발 환경(SSH)을 구성합니다.  
호스트는 RHEL 9, 클라이언트는 Windows 10 기준이며,  
2025.11 기준 최신 VSCode 버전 `1.105.1` 
(커밋 해시 `7d842fb85a0275a4a8e4d7e040d2625abbf7f084`)을 사용합니다.

### 1. 커밋 해시 확인

```
버전: 1.105.1
커밋: 7d842fb85a0275a4a8e4d7e040d2625abbf7f084
```

VSCode를 실행한 후 도움말 > 정보(About) 에서 버전과 커밋 해시를 확인할 수 있습니다.  
클라이언트(Windows)와 호스트(RHEL)의 커밋 해시는 반드시 동일해야 합니다.

### 2. 설치 파일 준비

| 구분 | 파일명 | 비고 |
|------|---------|------|
| VSCode (RHEL 9) | code-1.105.1-1760482588.el8.x86_64.rpm | root 권한으로 설치 |
| VSCode (Windows 10) | VSCodeUserSetup-x64-1.105.1.exe | 클라이언트 설치용 |
| VSCode Server | vscode-server-linux-x64.tar.gz | 커밋 해시 일치 필수 |
| VSCode 확장 | *.vsix | Python, Jupyter, Remote-SSH 등 |
| Docker Desktop | Docker Desktop Installer.exe | Dev Container 실행용 |

### 3. 확장 프로그램 다운로드 (`.vsix` 파일)

VSCode에서 확장의 `.vsix` 파일을 다운로드합니다.

1. VSCode를 실행합니다.  
2. 확장 탭에서 필요한 확장을 검색합니다.  
3. 마우스 오른쪽 버튼을 클릭한 후 "`.vsix` 파일 다운로드"를 선택합니다.  

다운로드한 `.vsix` 파일은 오프라인 환경에서 확장 설치 시 사용합니다.

### 4. VSCode Server 다운로드

VSCode Server는 클라이언트와 동일한 커밋 해시를 기준으로 다운로드해야 합니다.

다운로드 경로:
```
https://update.code.visualstudio.com/commit/<commit-hash>/server-linux-x64/stable
```

예시:
```
https://update.code.visualstudio.com/commit/7d842fb85a0275a4a8e4d7e040d2625abbf7f084/server-linux-x64/stable
```

다운로드한 파일명은 `vscode-server-linux-x64.tar.gz` 입니다.

---

### 5. 호스트 설치 절차 (RHEL 9)

#### (1) VSCode 설치 (root 권한)

사전에 다운로드한 `.rpm` 파일을 사용해 VSCode를 설치합니다.

```bash
sudo dnf install -y code-1.105.1-1760482588.el8.x86_64.rpm
```

#### (2) 사용자별 확장 설치

사용자 계정에 필요한 확장을 설치합니다.

```bash
# .vsix 파일이 위치한 경로로 이동
cd /path/to/vsix/files

# 확장 설치
code --install-extension *.vsix

# 설치 확인
code --list-extensions --show-versions
```

#### (3) VSCode Server 구성 

폐쇄망에서는 VSCode가 서버 파일을 자동으로 다운로드할 수 없으므로  
`vscode-server-linux-x64.tar.gz` 파일을 사전에 준비하고 직접 설치해야 합니다.

```bash
COMMIT_HASH="7d842fb85a0275a4a8e4d7e040d2625abbf7f084"
mkdir -p ~/.vscode-server/cli/servers/Stable-${COMMIT_HASH}/server
cd ~/.vscode-server/cli/servers/Stable-${COMMIT_HASH}/server
tar -xzf ~/vscode-server-linux-x64.tar.gz --strip-components=1
chown -R $USER:$USER ~/.vscode-server
chmod -R 755 ~/.vscode-server
```

설치 후 디렉터리 구조 예시:

```
~/.vscode-server/
 └── cli/
     └── servers/
         └── Stable-7d842fb85a0275a4a8e4d7e040d2625abbf7f084/
             └── server/
                 ├── bin/
                 ├── node
                 ├── extensions/
                 ├── out/
                 ├── package.json
                 └── product.json
```

> 온라인 환경에서 이미 구성된 `~/.vscode-server` 디렉터리를 tar로 복사하여 사용 가능  
> 예: `tar -czf vscode-server-copy.tar.gz ~/.vscode-server`   
> → 오프라인 서버로 옮겨 `tar -xzf vscode-server-copy.tar.gz -C ~` 실행

---

### 6. 클라이언트 설치 절차 (Windows 10)

#### (1) VSCode 설치

`VSCodeUserSetup-x64-1.105.1.exe` 파일을 실행하여 기본 옵션으로 설치합니다.

#### (2) 확장 설치

다운로드한 `.vsix` 파일을 사용해 확장을 설치합니다.

VSCode 실행 → 확장 탭 → 우측 상단 점 3개 메뉴 → "`.vsix` 파일에서 설치" 선택 후 파일 지정

#### (3) 설정 파일 수정

폐쇄망 환경에서 VSCode가 자동 업데이트 및 서버 다운로드를 시도하지 않도록 설정합니다.

```
Ctrl + Shift + P → Preferences: Open User Settings (JSON)
```

다음 내용을 추가합니다.

```json
{
  "update.mode": "none",
  "extensions.autoUpdate": false,
  "extensions.autoCheckUpdates": false,
  "remote.SSH.useLocalServer": false,
  "remote.SSH.localServerDownload": "off",
  "remote.SSH.allowLocalServerDownload": false,
  "remote.SSH.showLoginTerminal": true,
  "http.proxySupport": "off"
}
```

> `"allowLocalServerDownload": false` 옵션으로 서버 자동 다운로드 차단  

#### (4) SSH 구성 추가

SSH 연결 정보를 설정 파일에 추가합니다.

```
Ctrl + Shift + P → Remote-SSH: Open SSH Configuration File
```

`C:\Users\<사용자명>\.ssh\config` 파일을 편집합니다.

```
Host gpu-server
    HostName 192.168.0.50
    User idxkim
    Port 22
```


#### (5) Dev Container 연결 (폐쇄망 환경 수동 설정)

폐쇄망에서는 VSCode가 Dev Container 서버 파일을 자동으로 다운로드할 수 없습니다.  
따라서 한 번 연결 시도 후 생성되는 캐시 경로에 서버 파일을 직접 복사해야 합니다.

1. Dev Container 연결을 한 번 시도합니다.
2. 연결이 실패하면 다음 경로가 자동으로 생성됩니다.

```
C:\Users\<username>\AppData\Local\Temp\vsch\serverCache\<commit_id>\
```

3. 위 폴더에 `vscode-server-linux-x64.tar.gz` 파일을 복사합니다.
4. VSCode에서 재연결 시도 시 해당 파일을 사용해 컨테이너 내부로 서버가 복사됩니다.

> 캐시 폴더는 임시 디렉터리이므로 삭제되면 동일한 절차를 다시 수행해야 함 

---

##### 작동 원리 참고

- VSCode는 Dev Container를 연결할 때 `vscode-server-linux-x64.tar.gz` 파일을 다운로드하려 시도하며, 그 직전에 `serverCache` 폴더를 생성합니다.  
- 폐쇄망에서는 다운로드가 실패하지만, 폴더는 오류 직전 단계에서 자동으로 만들어집니다.  
- 이 폴더에 서버 파일을 넣으면 VSCode가 다음 연결 시 해당 파일을 재사용해   
  컨테이너 내부에 서버를 설치합니다.  
- 단, 한 번도 실행을 시도하지 않았다면 `serverCache` 폴더는 존재하지 않으며  
  수동으로 폴더를 만들어도 VSCode가 인식하지 못할 수 있습니다.  

---

## 14-1. VSCode 비밀번호 자동 입력 (SSH Key 설정)

### 1. SSH 키 생성 (Windows)

RSA 키 쌍을 생성합니다.  
경로와 패스프레이즈는 기본값을 사용합니다(엔터 3회).

```powershell
ssh-keygen -t rsa -b 4096
```

---

### 2. 공개키 확인

생성된 공개키를 확인합니다.

```powershell
Get-Content ~/.ssh/id_rsa.pub
```

---

### 3. 서버 계정 전환 (RHEL)

SSH 키를 등록할 계정으로 전환합니다.

```bash
su - <계정명>
```

---

### 4. SSH 디렉터리 생성

SSH 키를 저장할 디렉터리를 생성하고 권한을 설정합니다.

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

---

### 5. 공개키 등록

Windows에서 확인한 공개키(`ssh-rsa ...`)를 `authorized_keys` 파일에 추가합니다.  
파일 편집 후 `Ctrl+O` → `Enter`로 저장하고 `Ctrl+X`로 종료합니다.

```bash
nano ~/.ssh/authorized_keys
```

---

### 6. 파일 권한 설정

SSH 인증을 위한 파일 권한을 설정합니다.

```bash
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
```

---

### 7. SELinux 컨텍스트 복원

SELinux가 활성화된 경우 SSH 디렉터리의 보안 컨텍스트를 복원합니다.

```bash
restorecon -R -v ~/.ssh
```

---

### 8. SSH 설정 파일 열기 (Windows)

VSCode 명령 팔레트에서 SSH 설정 파일을 엽니다.

```
Ctrl + Shift + P → Remote-SSH: Open SSH Configuration File
```

---

### 9. SSH 호스트 추가

`C:\Users\<사용자명>\.ssh\config` 파일에 서버 정보를 추가합니다.

```
Host gpu-server
    HostName 192.168.1.50
    User idxkim
    IdentityFile ~/.ssh/id_rsa
```

---

### 10. 연결 테스트

VSCode에서 서버로 연결을 시도합니다.  
비밀번호 입력 없이 자동으로 로그인되면 SSH 키 인증이 정상적으로 설정된 것입니다.

```
Ctrl + Shift + P → Remote-SSH: Connect to Host → gpu-server
```

---

## 마무리

폐쇄망 환경에서 AI Solution 개발환경 구축의 핵심 인프라 구성이 완료되었습니다.  
이번 글에서는 GPU 기반 컨테이너 환경과 원격 개발 환경을 중심으로 다음 과정을 진행했습니다.

1. GPU 드라이버 및 런타임 구성
   - NVIDIA 드라이버 설치 및 `nouveau` 모듈 비활성화
   - NVIDIA Container Toolkit 설치로 GPU 컨테이너 연동 구성

2. 컨테이너 엔진 설치 및 GPU 연동
   - Docker 및 Podman 설치
   - Docker(runtime configure)와 Podman(CDI generate) 설정을 통한 GPU 접근 구성
   - 데이터 저장 경로 변경 및 최적화

3. 원격 개발 환경 구성
   - VSCode Remote-SSH 오프라인 설치
   - Dev Container 수동 연결 및 SSH Key 기반 자동 인증 설정

이로써 폐쇄망 환경에서도 GPU를 활용한 컨테이너 기반 AI 개발 환경이 구축되었습니다.

> 다음 편에서는 AI Solution 서비스 구성 및 배포로 이어집니다.
