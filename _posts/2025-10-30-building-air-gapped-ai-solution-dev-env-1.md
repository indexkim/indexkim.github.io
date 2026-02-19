---
title: 폐쇄망 AI Solution 개발환경 구축 (1)
tags: [infra, os, system]
toc: true
toc_sticky: true
post_no: 8
---

폐쇄망(Air-Gapped Network)은 외부 인터넷과 물리적으로 격리된 네트워크 환경을 의미합니다.
주로 금융기관, 공공기관, 연구소 등 민감한 정보를 다루는 환경에서 사용됩니다.

폐쇄망에서는 다음과 같은 제약이 있습니다:

- 인터넷을 통한 패키지 다운로드 불가 (`apt install`, `pip install` 등)
- 클라우드 서비스 접근 불가 (GitHub, Docker Hub, PyPI 등)
- 모든 소프트웨어와 데이터는 USB, 외장 SSD 등 물리적 매체로 전달

따라서, 폐쇄망에서 AI Solution 개발환경을 구축하려면  
사전에 모든 필요한 소프트웨어, 패키지, 모델 파일을 준비하고,  
오프라인 상태에서도 완전히 작동하는 독립적인 환경을 만들어야 합니다.

이 글은 실제 폐쇄망 환경 구축 경험을 바탕으로 작성되었습니다.  
RHEL 9을 기준으로 하지만, 다른 Linux 배포판에서도 동일한 방식으로 적용할 수 있습니다.

> 특정 조직의 내부 시스템이나 정책, 네트워크 설정은 포함하지 않으며  
> 모든 내용은 공개된 기술과 일반적인 Linux 환경을 기반으로 구성되었습니다.

---

## 1. 준비물

### 하드웨어

| 항목 | 설명 |
|------|------|
| GPU 탑재 워크스테이션 | AI 모델 학습 및 추론 환경 구축용 메인 시스템 (서버 역할 가능) |
| 클라이언트용 노트북 또는 PC | 워크스테이션 제어 및 환경 테스트용 |
| 랜선 (LAN Cable) | 워크스테이션과 클라이언트 간 유선 연결용 |
| L2 스위치(Layer 2 Switch) | 폐쇄망 내부 네트워크 구성용 (공유기 대신 사용) |
| USB 메모리 | RHEL 또는 Ubuntu 설치용 부팅 USB 제작 (16GB 이상 권장) |
| 외장 SSD 또는 대용량 USB | 오프라인 패키지, Docker 이미지, 모델 파일 전송용 (256GB 이상 권장) |
| 모니터 / 키보드 / 마우스 | 초기 설치 및 설정용 (KVM 스위치 사용 가능) |

---

### 소프트웨어 (사전 다운로드 필요)

| 항목 | 설명 | 용도 |
|------|------|------|
| Linux ISO 파일 | RHEL 9.x DVD ISO (~12GB) 또는 Ubuntu 22.04 LTS ISO (~4GB) | OS 설치 및 오프라인 리포지토리 구성 |
| USB 부팅 툴 | Rufus (Windows) 또는 Etcher (Mac/Linux) | 설치 USB 제작 |
| SFTP 클라이언트 | WinSCP 또는 FileZilla | 클라이언트 ↔ 워크스테이션 간 파일 전송 |
| NVIDIA 드라이버 | GPU 모델에 맞는 `.run` 파일 (예: `NVIDIA-Linux-x86_64-550.54.15.run`) | GPU 드라이버 설치 |
| NVIDIA Container Toolkit | 오프라인 `.rpm` 패키지 모음 (예: `libnvidia-container`, `nvidia-container-toolkit` 등) | Docker GPU 연동 |
| Docker 오프라인 패키지 | `docker-ce`, `docker-ce-cli`, `containerd.io` (`.rpm` 또는 `.deb`) | 컨테이너 런타임 설치 |
| Docker 이미지 | 프로젝트용 Docker 이미지를 `.tar`로 저장 (예: `idxkim_image:v0.1`) | AI 개발 환경 |
| VSCode 관련 패키지 | 설치 파일(`.exe`/`.rpm`), 확장(`.vsix`), Server 모듈(`vscode-server-linux-x64.tar.gz`) | 코드 편집 및 Remote SSH 환경 |
| 사전학습 모델 (선택) | Hugging Face 등에서 다운로드한 모델 (`.safetensors`, `.bin` 등) | AI 모델 학습 및 추론 |
| 추가 개발 도구 (선택) | `git`, `vim`, `tmux` 등의 오프라인 `.rpm`/`.deb` 패키지 | 개발 편의 도구 설치 |

> 폐쇄망 반입 전 모든 파일의 무결성 검증(해시 체크) 및 보안 스캔을 권장함

---

## 2. 설치 미디어 준비

### ISO 다운로드

- 경로:
  [Red Hat Developers 사이트](https://developers.redhat.com/products/rhel/download#getredhatenterpriselinux7163)
  (공식 포털이 아닌 developers 사이트에서 무료 회원가입 후 다운로드 가능)
- 파일 형태: `rhel-9.x-x86_64-dvd.iso`
- 용량: 약 12GB
- 버전 확인 및 무결성 검증:

  ```bash
  sha256sum rhel-9.x-x86_64-dvd.iso
  ```

  → 제공된 해시값과 동일한지 확인

> Developers용 ISO는 구독 없이도 다운로드 가능하며, 테스트·폐쇄망 개발용으로 적합

---

## 3. 설치 USB 제작

### 도구: Rufus

- [Rufus 공식 사이트](https://rufus.ie)에서 다운로드
- ISO 파일(`rhel-9.x-x86_64-dvd.iso`) 선택 후 USB 굽기
- 설정 권장값:

  - 파티션 방식: GPT
  - 대상 시스템: UEFI
  - 파일 시스템: NTFS (4GB 초과 파일 복사 가능)
  - 빠른 포맷 체크
- USB 용량: 최소 16GB 이상 권장

> 복사 후 USB에 `EFI`, `BaseOS`, `AppStream` 폴더가 보이면 성공

---

## 4. ISO 구조 확인

### ISO 내부 주요 폴더

- `/BaseOS` – 운영체제 핵심 패키지 (커널, 라이브러리 등)
- `/AppStream` – 개발 도구 및 응용 프로그램 모듈

설치용 ISO를 마운트하면 이 두 폴더를 확인할 수 있습니다.

ISO 파일을 임시 디렉터리에 마운트하여 내부 구조를 확인합니다:

```bash
mkdir -p /mnt/iso
mount -o loop rhel-9.x-x86_64-dvd.iso /mnt/iso
```

이후 `/mnt/iso/BaseOS`와 `/mnt/iso/AppStream`을 로컬 리포지토리로 등록하여   
폐쇄망에서도 `dnf install`이 가능하도록 구성합니다.

---

## 5. BIOS 부팅 및 설치 진입

### 부팅 절차

1. Rufus로 만든 설치 USB를 서버에 꽂기
2. BIOS/UEFI 진입 (`DEL`, `F2`, `F12`, `ESC` 등 제조사별 키)
3. Boot 메뉴에서 USB 선택
   - 중요: USB가 두 개로 보이면 UEFI: USB 이름 을 선택
   - 예시: `UEFI: SanDisk` (O) vs `SanDisk` (X)
   - UEFI 모드를 최우선 순위로 설정
4. 메뉴에서 `Install Red Hat Enterprise Linux 9.x` 선택

> UEFI 모드 필수 (Legacy BIOS로 설치 시 부팅 문제 발생 가능)

---

## 6. 설치 UI 단계

### 설정 요약

| 항목 | 선택/설정 | 설명 |
|------|------------|------|
| 언어 | English (US) | 설치 중 한글 깨짐 방지 |
| 파티션 | 자동(Auto) | 간편하지만 비효율적 ( `/home`, `/var` 분리되지 않음 ) |
| 네트워크 | 비활성화 | 폐쇄망 환경이므로 설정 불필요 |
| 설치 타입 | Workstation | GUI 포함, 초기 환경 구성에 편리함 |
| `root` 계정 | SSH 허용 | 원격 관리 편의성 확보 |
| 사용자 계정 | 모든 옵션 체크 | 관리자 권한 포함 |

> 파티션은 `/`, `/var`, `/home`을 분리하여 설정하는 것을 권장  
> 로그 누적 및 사용자 데이터로 인한 루트 파티션 포화 방지를 위해 구조 분리 필요

---

## 7. 로컬 리포지토리 등록

Linux 패키지 관리자(`dnf`, `apt` 등)는 일반적으로 인터넷의 원격 저장소(repository)에서 패키지를 다운로드합니다.
폐쇄망 환경에서는 인터넷 접근이 불가능하므로, RHEL 설치 ISO의 `BaseOS`와 `AppStream` 폴더를 워크스테이션 내부 디스크로 복사하여 로컬 리포지토리를 구성합니다.


### 오프라인 dnf repo 구성 절차

#### (1) ISO 마운트 및 디렉터리 복사

설치 USB 또는 ISO를 임시로 마운트합니다:

```bash
mkdir -p /mnt/iso
mount -o loop /path/to/rhel-9.x-x86_64-dvd.iso /mnt/iso
```

ISO 내부 디렉터리를 로컬 디스크로 복사합니다:

```bash
mkdir -p /opt/repos
cp -r /mnt/iso/BaseOS /opt/repos/
cp -r /mnt/iso/AppStream /opt/repos/
```

마운트 해제 후 USB를 제거합니다:

```bash
umount /mnt/iso
```

---

#### (2) 로컬 repo 설정 파일 생성

복사한 로컬 디렉터리를 리포지토리로 등록하기 위해 설정 파일을 생성합니다:

```bash
sudo vi /etc/yum.repos.d/local.repo
```

입력 내용:

```ini
[BaseOS]
name=RHEL-9-BaseOS
baseurl=file:///opt/repos/BaseOS
enabled=1
gpgcheck=0

[AppStream]
name=RHEL-9-AppStream
baseurl=file:///opt/repos/AppStream
enabled=1
gpgcheck=0
```

---

#### (3) repo 인덱스 갱신 및 확인

로컬 리포지토리 설정이 완료되면 패키지 캐시를 갱신하고
등록된 리포지토리 목록을 확인합니다.

```bash
sudo dnf clean all
sudo dnf repolist
```

출력 예시:

```
repo id                        repo name
BaseOS                         RHEL-9-BaseOS
AppStream                      RHEL-9-AppStream
```

---

#### (4) 패키지 설치 테스트

로컬 리포지토리가 정상적으로 동작하는지 확인하기 위해
간단한 패키지를 설치해봅니다.

```bash
sudo dnf install -y vim
```

정상 설치되면 로컬 리포지토리 구성이 완료된 것입니다.  

> USB를 제거해도 완전한 오프라인 환경 유지됨  
> 추후 새 버전 ISO로 업데이트할 때는 `/opt/repos` 디렉터리 덮어쓰기

---

## 8. 폐쇄망 네트워크 설정

### 유선 네트워크 구성


워크스테이션과 클라이언트(노트북/PC)는 L2 스위치(Layer 2 Switch) 를 통해 유선(LAN)으로 연결합니다.
L2 스위치는 MAC 주소를 기반으로 프레임을 전달하지만,
사용자가 별도로 MAC 주소를 설정할 필요는 없습니다.
각 장비의 네트워크 인터페이스에 기본 내장된 MAC 정보를 스위치가 자동으로 학습해 처리합니다.
공유기(Router)는 외부망 연결용 장비이므로 폐쇄망 환경에서는 사용하지 않습니다.

구성 예시:

```
[Workstation]──LAN──┐
                    │  (Switch/Hub)
[Client Laptop]──LAN─┘
```

---

### RHEL 워크스테이션 IP 설정

RHEL 9의 Settings → Network → Wired → IPv4 → Manual 에서 아래 값 입력:

| 항목           | 값             |
| ------------ | ------------- |
| IPv4 Address | 192.168.0.50  |
| Subnet Mask  | 255.255.255.0 |
| Gateway      | (비움)        |
| DNS          | (비움)        |

> 폐쇄망에서는 외부 네트워크(인터넷)와의 연결이 없으므로  
> 기본 게이트웨이와 DNS는 설정하지 않음

---

### 클라이언트 IP 설정(윈도우 기준)

#### (1) 네트워크 설정 진입

1. 제어판 → 네트워크 및 공유 센터 (Network and Sharing Center) 클릭
2. 왼쪽 메뉴의 어댑터 설정 변경(Change adapter settings) 클릭
3. 이더넷(Ethernet) 아이콘 찾기 (Wi-Fi가 아닌 유선 연결 항목)

---

#### (2) 속성 변경

1. Ethernet 아이콘 우클릭 → 속성(Properties)
2. 목록에서 Internet Protocol Version 4 (TCP/IPv4) 선택
3. 속성(Properties) 버튼 클릭

---

#### (3) IP 수동 입력

| 항목       | 값             |
| -------- | ------------- |
| IP 주소    | 192.168.0.51  |
| 서브넷 마스크  | 255.255.255.0 |
| 기본 게이트웨이 | 비워둠           |
| DNS 서버   | 비워둠           |

확인(OK) → 닫기(Close) 선택

---

#### (4) 연결 확인

명령 프롬프트(`cmd`)를 열고 다음을 입력합니다:

```bash
ping 192.168.0.50
```

워크스테이션에서 응답이 오면 네트워크 연결이 완료된 것입니다.

---

#### (5) 설정 확인

네트워크 설정이 올바르게 적용되었는지 확인합니다:

```bash
ipconfig
```

`IPv4 Address` 항목이 `192.168.0.51`로 표시되면 정상 설정된 것입니다.

---

## 9. Ubuntu vs RHEL(Red Hat Enterprise Linux)

폐쇄망 환경에서 AI Solution 개발환경 구축 시 운영체제 선택은 전체 복잡도에 직접적인 영향을 줍니다.
Ubuntu와 RHEL은 모두 Linux 계열이지만 철학, 관리 방식, 보안 정책, 지원 체계가 다릅니다.


| 구분               | Ubuntu                                    | RHEL |
| ---------------- | --------------------------------------------- | ----------------------------------- |
| 계열           | Debian 계열 (Community 기반)                      | Red Hat 계열 (Enterprise 상용 지원)       |
| 목적           | 범용 데스크톱 및 연구/개발 환경                            | 기업용 서버 및 안정성 중시 환경                  |
| 패키지 관리자      | `apt` (Advanced Package Tool)                 | `dnf` (Dandified YUM)               |
| 패키지 포맷       | `.deb`                                        | `.rpm`                              |
| 대표 무료 대안     | Linux Mint, Debian                            | Rocky Linux, AlmaLinux              |
| 라이선스 / 비용    | 무료 (Canonical Account 선택사항)                   | 유료 구독 기반 (Red Hat Subscription)     |
| LTS 정책       | 5년 기본 + 5년 ESM(Extended)                      | 10년 지원 (기업 고객 대상)                   |
| 설치 편의성       | GUI 기반 설치 간편, 바로 사용 가능                        | 초기 설정은 복잡하지만 재현 가능한 표준 절차 제공      |
| AI 프레임워크 지원성 | 공식 CUDA, PyTorch, TensorFlow 대부분 Ubuntu 기준 배포 | 대부분 지원되지만 일부 수동 설치 필요               |
| 패키지 최신성      | 최신 버전 빠르게 반영 (Rolling-ish)                    | 검증된 안정 버전 유지 (Conservative release) |
| 오프라인 repo 구성   | `apt-mirror` 사용 (의존성 관리 필요)                       | ISO 복사만으로 가능 (패키지는 제한적)         |
| 주 사용처        | 연구소, 스타트업, 개발 환경                              | 대기업, 공공기관, 납품 프로젝트                  |

---

### 차이점

#### (1) 설치 및 초기 세팅

| 항목 | Ubuntu | RHEL |
|------|--------|------|
| 설치 난이도 | GUI 기반 설치, 클릭 몇 번으로 완료 | Subscription 등록 또는 ISO 기반 수동 Repo 구성 필요 |
| 폐쇄망 구성 | `apt-mirror`, `apt-offline` 등 별도 도구 필요 | ISO 내 BaseOS/AppStream 복사만으로 오프라인 Repo 구성 가능 |

> Ubuntu는 인터넷 저장소 의존 구조로 인해 오프라인 설정이 복잡  
> (패키지 의존성, GPG 키, 미러 구성 등을 수동으로 준비해야 함)  
> RHEL은 ISO 내부에 저장소가 포함되어 있어 복사와 등록만으로 오프라인 운용 가능

---

#### (2) 패키지 관리 구조

| 항목 | Ubuntu | RHEL |
|------|--------|------|
| 저장소 관리 | `/etc/apt/sources.list` 단일 파일 | `/etc/yum.repos.d/*.repo` 다중 리포지토리 구조 |
| 구조적 특징 | 단일 구성 파일로 관리 단순 | 리포지토리 분리로 구성 유연 |
| 장점 | 설정이 간단하고 유지보수 용이 | 버전 충돌 방지 및 일관성 확보 |

> Ubuntu는 단일 저장소 구조로 관리가 직관적  
> RHEL은 리포지토리 단위 관리로 확장성과 안정성이 높음

---

#### (3) 보안 정책 (SELinux vs AppArmor)

| 항목 | Ubuntu (AppArmor) | RHEL (SELinux) |
|------|--------------------|----------------|
| 기본 보안 모듈 | AppArmor | SELinux |
| 접근 제어 방식 | 프로파일 기반 (경로 중심) | 라벨 기반 (보안 컨텍스트 중심) |
| 특징 | 설정이 간단하고 유연, 개발 친화적 | 세밀한 제어 가능, 강력한 정책 적용 |
| 단점 | 세부 제어 어려움, 기업 보안 표준 미흡 | 설정 까다로움, 디버깅 난이도 높음 |
| 폐쇄망 운영 시 | Permission 오류 적음 | Permission denied 자주 발생 (context 문제) |

> AppArmor는 유연하고 개발에 친화적  
> SELinux는 정밀한 제어와 보안 감사 기능이 강력함  
> 폐쇄망에서는 SELinux를 `Permissive` 모드로 완화 운영하는 경우가 많음

---

#### (4) AI 프레임워크 호환성

| 항목 | Ubuntu | RHEL |
|------|--------|------|
| 공식 지원 | PyTorch, TensorFlow, CUDA 등 대부분 공식 지원 | 수동 wheel 설치 필요 (특히 폐쇄망) |
| 업데이트 | 최신 버전 즉시 반영 | 안정성 위주, 업데이트 느림 |

> Ubuntu는 AI 개발 환경 구성이 용이  
> RHEL은 장기적인 운영 안정성이 높음

---

#### (5) 오프라인 패키지 리포지토리 구성

| 항목 | Ubuntu | RHEL |
|------|--------|------|
| 구성 도구 | `apt-mirror`, `apt-offline` | ISO(BaseOS/AppStream) 복사로 바로 구성 |
| 복잡도 | 의존성 관리 복잡 | 구성 간단, 제한적 패키지 |

> Ubuntu는 최신 패키지와 다양한 버전을 제공  
> RHEL은 단순한 구조로 오프라인 환경 구성에 용이

---

#### (6) 컨테이너 관리 (Docker vs Podman)

| 항목 | Ubuntu | RHEL |
|------|--------|------|
| 기본 컨테이너 엔진 | Docker | Podman |
| 실행 권한 | `root` 필요 | `rootless` 지원 (보안성 우수) |
| 호환성 | 풍부한 이미지 및 Compose 지원 | Docker Compose 별도 설치 필요 |
| 보안 | Docker 데몬 권한 위험 | Podman은 프로세스 격리 강화 |

> Ubuntu는 개발 및 배포 환경 구성이 편리  
> RHEL은 컨테이너 보안성과 격리성이 우수

---

### 요약 비교 및 선택 가이드

| 목적 | 추천 OS | 이유 |
|------|----------|------|
| AI 개발 환경 (일반) | Ubuntu LTS | 설치 간단, AI 프레임워크 호환성 우수, 최신 패키지 반영 속도 빠름 |
| 오프라인 repo 구성 | RHEL / Rocky Linux | ISO 복사만으로 로컬 리포지토리 구성 가능 (의존성 안정적) |
| 보안 감사 필수 환경 | RHEL / Rocky Linux | SELinux 기본 활성, 엔터프라이즈 보안 정책 및 인증 기준 충족 |
| 폐쇄망 + AI 개발 병행 | Ubuntu LTS | 패키지 다양성, Docker/CUDA 생태계와 높은 호환성, 개발 생산성 우수 |

> Ubuntu는 개발 및 테스트 환경에 적합  
> RHEL은 보안과 안정성을 요구하는 운영 환경에 적합

---

### APT ↔ DNF 명령어 대응표

Ubuntu와 RHEL 계열의 가장 큰 차이 중 하나는 패키지 관리 도구입니다.  
Ubuntu는 `apt`, RHEL은 `dnf`를 사용하지만, 명령 체계와 기능은 매우 유사합니다.
아래 표는 동일 작업을 수행할 때의 명령어 대응 관계입니다.

| 목적           | Ubuntu (apt)                     | RHEL / CentOS / Rocky (dnf)             | 비고                          |
| ------------ | ------------------------------------ | ------------------------------------------- | --------------------------- |
| 패키지 목록 갱신    | `sudo apt update`                    | `sudo dnf clean all`<br>`sudo dnf repolist` | `dnf update`는 업그레이드용        |
| 패키지 검색       | `apt search docker`                   | `dnf search podman`                          | 결과는 repo별로 구분 표시            |
| 패키지 설치       | `sudo apt install -y docker.io`             | `sudo dnf install -y podman`                    | `-y` 옵션으로 자동 yes                       |
| 패키지 제거       | `sudo apt remove -y docker.io`              | `sudo dnf remove -y podman`                     | 의존성 자동 제거                   |
| 필요없는 패키지 제거  | `sudo apt autoremove -y`                | `sudo dnf autoremove -y`                       | 미사용 패키지 정리                  |
| 전체 업그레이드     | `sudo apt upgrade -y`                   | `sudo dnf upgrade -y`                          | RHEL은 `update` 대신 `upgrade` |
| 특정 패키지 정보 확인 | `apt show podman`                     | `dnf info podman`                            | 버전, 의존성 확인                  |
| 로컬 파일 직접 설치  | `sudo dpkg -i file.deb`              | `sudo rpm -ivh file.rpm`                    | 포맷 다름 (.deb vs .rpm)        |
| GPG 키 추가     | `apt-key add key.gpg`                | `rpm --import key.gpg`                      | repo 인증 키 등록                |
| 저장소 추가       | `add-apt-repository ppa:example/ppa` | `dnf config-manager --add-repo URL`         | `.repo` 파일 생성됨              |
| 저장소 목록       | `/etc/apt/sources.list`              | `/etc/yum.repos.d/*.repo`                   | RHEL은 repo 분리 구조            |
| 패키지 캐시 위치    | `/var/cache/apt/archives`            | `/var/cache/dnf`                            | 캐시 삭제 가능                    |
| 의존성 확인       | `apt depends python3`                  | `dnf repoquery --requires python3`            | dnf가 더 세부적                  |
| 설치된 패키지 목록   | `apt list --installed`               | `dnf list installed`                        | 동일 기능                       |
| 잠금/병렬 설치 구조  | 단일 프로세스                              | 트랜잭션 기반                                     | dnf가 더 안정적                  |



> RHEL 8 이후 `yum`은 `dnf`로 완전히 대체  
> Ubuntu는 `.deb`, RHEL은 `.rpm` 포맷을 사용하므로 교차 설치 불가  
> 폐쇄망 환경에서는 `dnf`가 ISO 또는 디렉터리 기반 리포지토리(`baseurl=file:///opt/repos/...`) 구성에 유리
  
---

## 마무리

폐쇄망 환경에서 AI Solution 개발환경을 구축하는 과정은  
OS 설치를 넘어 완전한 독립 생태계 구성을 목표로 합니다.

이번 글에서는 폐쇄망 환경 구축의 기초 단계를 다음과 같이 진행했습니다.

1. 준비 및 OS 설치  
   - 하드웨어·소프트웨어 목록 정리 및 RHEL ISO 기반 부팅 USB 제작  
   - UEFI 모드 부팅 및 Workstation 설치 완료  

2. 오프라인 패키지 저장소 구성  
   - ISO 내부 BaseOS/AppStream을 로컬 디스크로 복사  
   - `/etc/yum.repos.d/local.repo` 설정으로 인터넷 없이 `dnf install` 가능한 환경 구축  

3. 폐쇄망 네트워크 설정  
   - Switch/Hub 기반 유선 연결 및 고정 IP 할당  
   - 워크스테이션–클라이언트 간 통신 확인  

4. 운영체제 비교 (Ubuntu vs RHEL)  
   - 패키지 관리, 보안 정책, AI 프레임워크 호환성 분석  
   - 폐쇄망 환경에 적합한 OS 선택 기준 제시  

이로써 인터넷 연결 없이도 패키지 설치와 네트워크 통신이 가능한  
폐쇄망 기본 인프라 환경이 완성되었습니다.

> 다음 편에서는 컨테이너와 GPU 기반 AI 개발 환경 구축 과정으로 이어집니다.
