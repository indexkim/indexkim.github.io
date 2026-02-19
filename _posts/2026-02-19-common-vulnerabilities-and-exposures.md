---
title: 폐쇄망 Docker 이미지 CVE 대응
tags: [docker, infra, security]
toc: true
toc_sticky: true
post_no: 11
---

CVE(Common Vulnerabilities and Exposures)는 소프트웨어·하드웨어에서 발견된 보안 취약점을  
고유하게 식별하기 위해 부여되는 표준 ID 체계입니다.  
Docker 이미지에 포함된 OS 패키지나 Python 패키지 중 알려진 CVE가 존재하면  
보안 검수에서 반려될 수 있으며, 배포 전에 이를 식별하고 제거해야 합니다.

일반적인 프로젝트에서는 Docker 이미지의 CVE를 검수하지 않는 경우가 많지만,  
보안 요건이 높은 환경에서는 HIGH/CRITICAL 취약점이 0건이어야 배포가 승인됩니다.

온라인 환경에서는 `apt upgrade`나 `pip install --upgrade` 한 줄이면 끝나는 작업이지만,  
폐쇄망에서는 패키지 저장소에 접근할 수 없어 패치 파일을 직접 준비해야 하고,  
스캐너의 취약점 DB조차 수동으로 반입해야 합니다.  
실제로 대응을 진행해 보니, 패키지 업데이트 외에도  
커널 CVE의 검수 범위 문제나 패치 미제공 상황 등 예외 케이스가 많았습니다.

이 글은 폐쇄망 환경에서 Docker 이미지의 CVE를 스캔하고,  
패키지를 패치하고, 예외 상황에 대응하는 전체 과정을 정리한 것입니다.

> 특정 이미지나 프로젝트에 한정되지 않으며  
> Debian/Ubuntu 기반 Docker 이미지라면 동일한 방식으로 적용할 수 있습니다.

---

## 1. CVE(Common Vulnerabilities and Exposures)

### CVE란?

CVE는 MITRE Corporation이 운영하며, `CVE-연도-일련번호` 형식으로 표기됩니다.

```
CVE-2026-0861
 │    │    │
 │    │    └── 일련번호 (해당 연도 내 순번)
 │    └─────── 취약점이 등록된 연도
 └──────────── CVE 접두어
```

CVE가 등록되면 NVD(National Vulnerability Database)에서 해당 취약점의 심각도를 평가합니다.  
심각도는 CVSS(Common Vulnerability Scoring System)라는 점수 체계로 산정되며,  
0.0~10.0 범위의 점수가 다음과 같은 등급으로 분류됩니다.

| CVSS 점수 | 등급 | 설명 |
|-----------|------|------|
| 9.0 ~ 10.0 | Critical | 원격 코드 실행, 인증 우회 등 즉각 대응 필요 |
| 7.0 ~ 8.9 | High | 권한 상승, 정보 유출 등 우선 대응 권장 |
| 4.0 ~ 6.9 | Medium | 제한적 조건에서 악용 가능 |
| 0.1 ~ 3.9 | Low | 악용 가능성이 낮음 |

> 보안 검수에서는 일반적으로 HIGH(7.0 이상)와 CRITICAL(9.0 이상)이 대상  
> MEDIUM 이하는 조직 보안 정책에 따라 허용되는 경우가 많음

Docker 이미지에는 OS 패키지와 Python 패키지가 포함되어 있으며,  
이 중 알려진 CVE가 존재하는 패키지가 있으면 보안 검수에서 반려됩니다.

| 구분 | 자주 보고되는 패키지 예시 |
|------|------------------------|
| OS 레벨 CVE | `linux-libc-dev`, `gnupg`, `gpgv`, `openssl`, `libexpat1` 등 |
| Python 레벨 CVE | `requests`, `urllib3`, `setuptools`, `torch`, `transformers` 등 |

---

### 취약점 스캐너

컨테이너 이미지의 CVE를 식별하려면 취약점 스캐너가 필요합니다.  
스캐너는 이미지 내 설치된 패키지 목록을 추출하고,  
취약점 DB와 대조하여 알려진 CVE가 포함되어 있는지 확인합니다.

#### Trivy

[Trivy](https://github.com/aquasecurity/trivy)는 Aqua Security에서 개발한 오픈소스 취약점 스캐너입니다.  
컨테이너 이미지, 파일시스템, Git 저장소, Kubernetes 클러스터 등을 스캔할 수 있으며,  
취약점 외에도 Misconfiguration과 Secret 노출까지 탐지합니다.

| 항목 | 설명 |
|------|------|
| 스캔 대상 | 컨테이너 이미지, 파일시스템, Git 저장소, K8s 클러스터 |
| 탐지 범위 | CVE, Misconfiguration, Secret |
| 바이너리 크기 | 단일 바이너리 (~50MB), 별도 설치 불필요 |
| 오프라인 지원 | DB를 사전 다운로드하면 완전 오프라인 스캔 가능 |

#### Trivy DB 업데이트 방식

Trivy는 취약점 정보를 자체 DB에 저장하며,  
이 DB는 6시간 간격으로 빌드됩니다.  
Trivy CLI는 로컬 DB가 일정 시간(기본 12시간) 이상 경과하면  
스캔 전에 자동으로 최신 DB를 다운로드합니다.

```
Trivy 스캔 실행
    ↓
DB 최신 여부 확인
    ↓ (오래된 경우)
OCI 레지스트리에서 DB 다운로드
    ↓
스캔 수행
```

DB는 다음 OCI 레지스트리에서 배포됩니다.

| 레지스트리 | 주소 |
|-----------|------|
| GitHub Container Registry | `ghcr.io/aquasecurity/trivy-db` |
| Docker Hub | `aquasec/trivy-db` |
| AWS ECR | `public.ecr.aws/aquasecurity/trivy-db` |

그러나 폐쇄망에서는 OCI 레지스트리에 접근할 수 없으므로  
외부망에서 DB를 미리 다운로드하여 물리 매체로 반입해야 합니다.

```bash
# 외부망: DB 다운로드
./trivy image --download-db-only

# 외부망: DB 압축
cd ~/.cache/trivy
tar -czf trivy-db.tar.gz db

# 폐쇄망: DB 복원
mkdir -p /root/.cache/trivy
tar -xzf trivy-db.tar.gz -C /root/.cache/trivy

# 폐쇄망: 스캔 시 DB 업데이트 차단
trivy fs --skip-db-update --scanners vuln /
```

> 폐쇄망에서의 Trivy DB는 수동 반입·수동 갱신  
> DB가 오래되면 새로 발견된 CVE를 탐지하지 못하므로  
> 검수 시점에 맞춰 최신 DB를 다운로드하여 반입하는 것을 권장

#### 다른 스캐너와 비교

Trivy 외에도 컨테이너 이미지 취약점을 스캔할 수 있는 도구들이 있습니다.

| 도구 | 개발사 | 오프라인 지원 | 특징 |
|------|--------|-------------|------|
| Trivy | Aqua Security | O (DB 사전 반입) | 단일 바이너리, 빠른 스캔, 설정 간편 |
| Grype | Anchore | O (DB 사전 반입) | Syft(SBOM 생성 도구)와 연동, Trivy와 유사한 사용법 |
| Clair | Red Hat/CoreOS | O (복잡한 설정 필요) | Kubernetes 통합에 강점, 서버 데몬 방식 |
| Docker Scout | Docker | X | Docker Desktop 내장, 오프라인 미지원 |

> Trivy를 선택한 이유는 오프라인 지원이 간편하고 단일 바이너리로 동작하기 때문  
> Grype도 유사한 방식으로 오프라인 스캔이 가능하지만  
> Clair는 서버 데몬을 별도로 구성해야 해서 폐쇄망에서는 설정 부담이 큼  
> Docker Scout는 오프라인을 지원하지 않으므로 폐쇄망에서 사용 불가

---

## 2. 전체 전략

CVE 대응은 작업용 이미지와 배포용 이미지를 분리하여 진행합니다.

```
원본 이미지
    ↓
작업용 이미지 (bash ENTRYPOINT)
    → OS 패키지 제거
    → Python 패키지 업데이트
    → Trivy 스캔 (HIGH/CRITICAL = 0 확인)
    ↓
배포용 이미지 (원래 ENTRYPOINT 복구)
    → 서비스 실행용 최종 이미지
```

| 이미지 | ENTRYPOINT | 용도 |
|--------|-----------|------|
| 작업용 | `/bin/bash` | CVE 제거 작업 + Trivy 스캔 |
| 배포용 | 원본 ENTRYPOINT 복구 | 실제 서비스 실행 |

> 작업용 이미지에서 CVE를 제거한 뒤 `docker commit`으로 고정하고  
> 배포용 이미지에서 ENTRYPOINT만 복구하는 2단계 구조  
> 이렇게 분리하면 스캔 도구와 서비스 실행이 충돌하지 않음

---

## 3. 외부망 준비 (1회)

폐쇄망 반입 전에 외부망에서 다음 파일들을 준비합니다.

### 1. Trivy 바이너리

```bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.69.1/trivy_0.69.1_Linux-64bit.tar.gz
tar -xzf trivy_0.69.1_Linux-64bit.tar.gz
chmod +x trivy
```

> Trivy 릴리즈 페이지에서 최신 버전을 확인하여 다운로드

---

### 2. Trivy 취약점 DB

1장에서 설명한 대로, 폐쇄망에서는 Trivy DB를 수동으로 반입해야 합니다.  
1장의 DB 다운로드 및 압축 명령어를 참고하여 `trivy-db.tar.gz` 파일을 준비합니다.

> DB는 6시간마다 갱신되므로 보안 검수 직전에 최신 DB를 다운로드하는 것을 권장

---

### 3. Python whl 패키지

Trivy 스캔 결과에서 CVE가 보고된 Python 패키지의 패치된 버전을 `.whl` 파일로 다운로드합니다.

```bash
pip download <패키지명>==<패치_버전> ...
```

예를 들어 `urllib3`에 HIGH 취약점이 보고되었고, 패치된 버전이 2.6.0이라면:

```bash
pip download urllib3==2.6.0
```

> 패키지 버전은 CVE가 수정된 최소 버전 이상을 지정  
> 의존성 충돌을 방지하기 위해 설치 시 `--no-deps` 옵션을 사용할 예정이므로  
> 대상 패키지만 정확히 다운로드

---

### 4. 반입 대상 정리

```
trivy                  # Trivy 바이너리
trivy-db.tar.gz        # 취약점 DB
*.whl                  # 패치된 Python 패키지
```

---

## 4. 작업용 이미지 생성

### 1. 원본 이미지 태그

원본 이미지를 작업용 태그로 복제합니다.  
원본을 보존하면서 작업용 이미지를 분리하기 위함입니다.

```bash
docker tag <IMAGE>:<TAG> <IMAGE>:<TAG>-work
```

---

### 2. 작업용 컨테이너 실행

ENTRYPOINT를 `/bin/bash`로 오버라이드하여 인터랙티브 셸로 진입합니다.

```bash
docker rm -f cve-work 2>/dev/null || true

docker run -it \
  --entrypoint /bin/bash \
  --name cve-work \
  <IMAGE>:<TAG>-work
```

> `--entrypoint /bin/bash`로 실행해야 컨테이너 안에서 패키지 제거, 설치, 스캔 작업 가능  
> 원래 ENTRYPOINT로 실행하면 셸 접근이 제한됨

---

## 5. 컨테이너 내부 CVE 제거

### 1. 파일 복사 (호스트 → 컨테이너)

호스트에서 준비한 파일들을 작업용 컨테이너 내부로 복사합니다.

```bash
# 별도 터미널에서 실행 (호스트)
docker cp trivy cve-work:/usr/local/bin/trivy
docker cp trivy-db.tar.gz cve-work:/opt/trivy-db.tar.gz
docker cp /path/to/whl cve-work:/opt/whl
```

---

### 2. Trivy DB 복원

```bash
# 컨테이너 내부에서 실행
mkdir -p /root/.cache/trivy
cd /root/.cache/trivy
tar -xzf /opt/trivy-db.tar.gz
```

---

### 3. OS 레벨 CVE 제거

Trivy 스캔 결과에서 HIGH/CRITICAL로 보고된 OS 패키지를 제거합니다.  
Debian/Ubuntu 기반 이미지에서 자주 보고되는 패키지는 다음과 같습니다.

```bash
apt-get update

# 예시: 자주 보고되는 취약 패키지 제거
apt-get purge -y gnupg* dirmngr
apt-get purge -y gpg gpgv gpgconf

# 불필요한 의존성 정리
apt-get autoremove -y
apt-get clean
```

#### OS 패키지가 제거 가능한 이유

Docker 이미지에는 빌드 시점에 필요했지만  
실제 서비스 실행(런타임)에는 사용되지 않는 패키지가 포함되어 있는 경우가 많습니다.  
이러한 패키지는 제거해도 서비스에 영향이 없으며,  
CVE 제거와 이미지 경량화를 동시에 달성할 수 있습니다.

gnupg, dirmngr, gpg, gpgv, gpgconf는 GPG(GNU Privacy Guard) 스택을 구성하는 패키지들로,  
각각의 역할은 다음과 같습니다.

| 패키지 | 역할 |
|--------|------|
| `gnupg` | GPG 메타 패키지 (아래 도구들을 묶어서 설치) |
| `gpg` | 암호화/서명/검증 수행 |
| `gpgv` | 서명 검증 전용 (경량 버전) |
| `gpgconf` | GPG 설정 관리 도구 |
| `dirmngr` | 키 서버와의 네트워크 통신 담당 |

이들은 주로 `apt-get update` / `apt-get install` 시  
패키지 저장소의 서명을 검증하는 데 사용됩니다.  
이미지 빌드 과정에서 패키지를 설치할 때 필요하지만,  
빌드가 완료된 후에는 더 이상 패키지를 설치할 일이 없으므로 제거할 수 있습니다.

특히 폐쇄망 환경에서는 외부 패키지 저장소에 접근할 수 없으므로  
GPG 서명 검증 기능 자체가 사용될 가능성이 없습니다.

| 시점 | 필요 여부 | 이유 |
|------|----------|------|
| 이미지 빌드 시 | O | apt 패키지 설치 시 저장소 서명 검증 |
| 런타임 (서비스 실행) | X | 패키지 추가 설치 없음, 서명 검증 불필요 |
| 폐쇄망 환경 | X | 외부 저장소 접근 자체가 불가 |

#### 제거 전 확인 방법

패키지를 제거하기 전에 해당 패키지에 의존하는 다른 패키지가 있는지 확인합니다.

```bash
# 역의존성 확인: 이 패키지를 필요로 하는 다른 패키지 목록
apt-cache rdepends --installed <패키지명>
```

역의존성 목록에 서비스 실행에 필요한 패키지가 포함되어 있다면 제거하면 안 됩니다.

> 위는 자주 보고되는 예시이며, 실제 제거 대상은 Trivy 스캔 결과에 따라 다름  
> 핵심 판단 기준은 "이 패키지가 빌드 시점에만 필요한가, 런타임에도 필요한가"  
> 빌드 전용 패키지는 제거해도 서비스에 영향이 없음

---

### 4. Python 패키지 업데이트

사전에 준비한 `.whl` 파일로 취약 Python 패키지를 업데이트합니다.

```bash
pip install --no-index --find-links=/opt/whl --no-deps <패키지명>==<패치_버전>
```

| 옵션 | 설명 |
|------|------|
| `--no-index` | PyPI 접근 차단 (오프라인 환경이므로 필수) |
| `--find-links` | 로컬 whl 디렉터리 지정 |
| `--no-deps` | 의존성 자동 설치 방지 (기존 환경 보존) |

> `--no-deps`를 사용하는 이유는 의존성 자동 해결 시  
> 기존에 설치된 다른 패키지와 버전 충돌이 발생할 수 있기 때문  
> 대상 패키지만 정확히 교체하는 것이 안전

---

### 5. 버전 증빙

업데이트된 패키지 버전을 기록합니다.  
보안 검수 시 증빙 자료로 제출할 수 있습니다.

```bash
pip list | egrep '<패키지1>|<패키지2>|...' \
  | tee /tmp/python_versions.txt
```

---

## 6. Trivy 오프라인 스캔

### 1. 스캔 실행

```bash
export TRIVY_SKIP_DB_UPDATE=true

trivy fs \
  --skip-db-update \
  --scanners vuln \
  --severity HIGH,CRITICAL \
  --format table \
  / | tee /tmp/trivy_scan_result.txt
```

| 옵션 | 설명 |
|------|------|
| `--skip-db-update` | DB 업데이트 시도 차단 (오프라인 필수) |
| `--scanners vuln` | 취약점 스캐너만 실행 |
| `--severity HIGH,CRITICAL` | HIGH/CRITICAL 등급만 필터링 |
| `--format table` | 사람이 읽기 쉬운 테이블 형식 |
| `trivy fs /` | 컨테이너 내 전체 파일시스템 스캔 |

---

### 2. 합격 기준

```
Total: 0 (HIGH: 0, CRITICAL: 0)
```

HIGH 또는 CRITICAL 등급의 취약점이 0건이면 합격입니다.

> MEDIUM 이하는 일반적으로 검수에서 허용되지만  
> 조직 보안 정책에 따라 기준이 다를 수 있음

---

### 3. 불합격 시 대응

스캔 결과에 취약점이 남아 있으면 다음을 확인합니다.

| 상황 | 조치 |
|------|------|
| OS 패키지 잔여 | 추가 `apt-get purge` 실행 |
| Python 패키지 잔여 | 패치 버전 whl 재반입 후 재설치 |
| 패치 버전 미존재 | 해당 CVE에 대한 예외 신청 (보안팀 협의) |

모든 CVE가 패키지 제거나 업데이트로 해결되는 것은 아닙니다.  
패치가 존재하지 않거나, 제거하면 서비스가 깨지거나,  
컨테이너 레벨에서 대응할 수 없는 경우가 있습니다.  
이 경우 리스크 수용 또는 예외 처리를 보안팀에 요청해야 합니다.

---

### 4. 예외 처리가 필요한 경우

실제 CVE 대응 과정에서는 패키지 제거나 업데이트만으로 해결되지 않는 경우가 빈번합니다.  
아래는 실무에서 자주 발생하는 예외 패턴과 대응 논리입니다.

#### 1. 패치 버전이 배포되지 않은 경우

배포판(Debian, Ubuntu 등)에서 해당 CVE를 Minor issue(`<no-dsa>`)로 분류하고,  
보안 업데이트를 별도로 제공하지 않는 경우입니다.

```
[상황]
- Trivy에서 HIGH로 보고되었으나
  배포판 보안팀은 해당 CVE를 Minor issue로 분류
- 저장소에 적용 가능한 패치 버전이 존재하지 않음

[대응 논리]
- 취약점의 악용 조건이 매우 제한적 (예: 공격자가 특정 파라미터를 동시에 제어해야 함)
- 일반적인 런타임 환경에서 실질 악용 가능성이 낮음
- 패치 제공 시 즉시 반영을 전제로 리스크 수용 요청
```

> Trivy의 심각도 판정과 배포판의 판정이 다를 수 있음  
> NVD에서는 HIGH로 분류하더라도 Debian/Ubuntu 보안팀이 자체 평가 후  
> `<no-dsa>` (Debian Security Advisory 미발행)로 처리하는 경우가 있음

---

#### 2. 호스트 커널 CVE가 컨테이너에서 탐지되는 경우

Linux 커널 관련 CVE가 컨테이너 이미지 스캔에서 탐지되는 경우입니다.  
컨테이너는 자체 커널을 포함하지 않고 호스트 시스템의 커널을 공유하므로,  
컨테이너 이미지 레벨에서는 근본적인 해결이 불가능합니다.

```
[상황]
- 커널 소스 코드에 대한 CVE가 컨테이너 이미지 스캔에서 탐지
- 컨테이너 내부에서 패키지를 제거하거나 업데이트해도 효과 없음

[대응 논리]
- 컨테이너는 자체 커널을 포함/사용하지 않음 (호스트 커널 공유)
- 취약점의 영향 여부는 호스트 OS 커널 버전과 보안 패치 상태에 따라 결정
- 컨테이너 이미지가 아닌 호스트 OS 커널 패치 적용 여부를 기준으로 리스크 관리
```

대표적인 예가 `linux-libc-dev` 패키지입니다.  
이 패키지는 Linux 커널 헤더 파일을 제공하는 개발용 패키지로,  
이미지 빌드 시 C extension을 컴파일할 때 설치되는 경우가 많습니다.

Trivy는 이 패키지의 버전을 기준으로 커널 CVE를 매핑하여 보고합니다.  
문제는 이 패키지 하나에 매핑되는 커널 CVE가 매우 많다는 것입니다.  
일반적인 Docker 이미지를 스캔하면  
전체 탐지 건수의 절반 이상이 `linux-libc-dev` 기인인 경우가 흔합니다.

다만 `linux-libc-dev`는 커널 헤더 파일일 뿐, 실제 커널 코드가 컨테이너에 포함된 것이 아닙니다.  
컨테이너는 호스트의 커널을 공유하므로 이 패키지에서 보고되는 CVE는  
컨테이너 이미지 레벨에서는 대응할 수도, 대응할 필요도 없습니다.

따라서 이 유형의 CVE는 컨테이너 이미지 검수 범위에서 제외하고,  
호스트 OS의 커널 패치 상태를 기준으로 별도 관리하는 것이 일반적입니다.

> `linux-libc-dev`를 검수 범위에서 제외하면 전체 CVE 건수가 대폭 줄어듦  
> 보안팀과 사전에 "커널 CVE는 호스트 OS 영역"이라는 합의를 하면  
> 이후 검수에서 반복적으로 같은 논의를 할 필요가 없음

---

#### 3. 라이브러리가 존재하지만 런타임에 로드되지 않는 경우

취약 라이브러리가 이미지에 포함되어 있지만,  
실제 서비스 실행 시 해당 라이브러리가 메모리에 로드되지 않는 경우입니다.

```
[상황]
- 취약 라이브러리(.so)가 이미지에 존재
- 그러나 서비스 프로세스의 메모리 맵(/proc/<pid>/maps)에서 미검출
- 다른 핵심 패키지가 해당 라이브러리에 의존하여 제거 불가

[확인 방법]
# 컨테이너 내부에서 실행
cat /proc/<서비스_PID>/maps | grep <라이브러리명>
# → 출력 없으면 런타임에 로드되지 않음

# 역의존성 확인
apt-cache rdepends --installed <라이브러리_패키지명>
# → 핵심 패키지가 의존하고 있으면 제거 불가

[대응 논리]
- 라이브러리는 이미지에 포함되어 있으나 런타임에 사용(로드)되지 않음
- 의존성 관계로 인해 패키지 삭제 시 서비스 기능에 영향
- 실제 악용 가능성이 낮으므로 패치 버전 적용 가능 시까지 리스크 수용 요청
```

> 공유 라이브러리(.so)는 다른 패키지가 의존하더라도 실제 실행 시 로드되지 않을 수 있음  
> `/proc/<pid>/maps`로 런타임 로드 여부를 확인하면 실질적인 영향 범위를 입증할 수 있음

---

#### 4. 패치 완료되었으나 스캐너가 구버전으로 인식하는 경우

실제로는 패치된 버전의 파일을 사용하고 있지만,  
호환성을 위해 구버전 이름의 심볼릭 링크를 유지해 스캐너가 취약 버전으로 탐지하는 경우입니다.

```
[상황]
- 취약 버전의 라이브러리/JAR 파일은 이미 삭제
- 패치된 버전의 파일만 존재
- 그러나 애플리케이션 호환성을 위해 구버전 이름의 심볼릭 링크를 유지
- Trivy가 심볼릭 링크의 파일명(구버전)을 기준으로 취약 판정

[확인 방법]
# 실제 파일 확인
ls -la /path/to/<라이브러리>*
# → 심볼릭 링크가 패치된 버전의 실제 파일을 가리키는지 확인

[대응 논리]
- 실제 파일시스템에는 패치된 버전만 존재
- 취약 버전의 파일은 포함되어 있지 않음
- 심볼릭 링크는 이름만 구버전이며, 참조하는 실제 파일은 패치 완료
```

---

#### 5. 구버전 베이스 이미지로 인한 대량 CVE

베이스 이미지 자체가 오래된 OS 버전(예: Debian 11)으로 빌드되어  
다수의 CVE가 동시에 보고되며, 개별 패치로는 대응이 불가능한 경우입니다.

```
[상황]
- 구버전 OS 기반 이미지에서 수십~수백 건의 CVE 탐지
- 대부분의 패키지에 Fixed version이 배포되지 않음
- 개별 패키지 업데이트로는 해결 불가

[대응]
- 최신 OS(Debian 12/13 등)로 빌드된 새 이미지를 반입
- 주요 라이브러리 버전 업데이트에 따른 애플리케이션 호환성 검증 필요
- 이미지 재빌드 후 다시 스캔하여 잔여 CVE 확인
```

> 구버전 이미지는 개별 CVE 패치보다 이미지 재빌드가 효율적  
> 다만 라이브러리 메이저 버전이 변경되면 애플리케이션 코드 수정이 필요할 수 있으므로  
> 개발팀과의 사전 호환성 검증이 필수

---

#### 예외 요청 시 포함할 항목

예외 처리를 요청할 때는 다음 항목을 정리하여 보안팀에 전달합니다.

| 항목 | 내용 |
|------|------|
| CVE ID | `CVE-XXXX-XXXXX` |
| 심각도 | CVSS 점수 및 등급 (HIGH / CRITICAL) |
| 영향 패키지 | 패키지명 및 현재 설치 버전 |
| 배포판 분류 | 배포판 보안팀의 자체 분류 (예: `<no-dsa>`, Minor issue) |
| 악용 조건 | 취약점이 실제로 악용되려면 필요한 조건 |
| 런타임 영향 | 서비스 실행 시 해당 코드 경로의 사용 여부 |
| 제거 불가 사유 | 의존성, 호환성 등 패치/제거가 불가능한 이유 |
| 대응 계획 | 패치 제공 시 즉시 반영, 이미지 재빌드 예정 등 |

> "패치가 없다"로 끝내는 것이 아니라  
> "왜 현 시점에서 리스크가 낮은지"를 기술적으로 입증하는 것이 핵심  
> 런타임 로드 여부, 악용 조건의 제한성, 배포판의 자체 판단 등을 근거로 제시

---

## 7. 이미지 확정 및 배포용 빌드

### 1. 작업용 이미지 커밋

스캔을 통과한 작업용 컨테이너를 이미지로 고정합니다.

```bash
# 컨테이너에서 exit
exit

# 이미지 커밋
docker commit cve-work <IMAGE>:<TAG>-hardened
```

> `docker commit`은 현재 컨테이너 상태를 그대로 이미지로 저장  
> 이 시점의 이미지에는 CVE가 제거된 상태가 반영되어 있음

---

### 2. 배포용 이미지 빌드

작업용 이미지는 ENTRYPOINT가 `/bin/bash`로 되어 있으므로  
서비스 실행을 위해 원래 ENTRYPOINT를 복구해야 합니다.

```dockerfile
FROM <IMAGE>:<TAG>-hardened
ENTRYPOINT ["<원본_ENTRYPOINT>"]
```

```bash
docker build -t <IMAGE>:<TAG>-hardened-release .
```

원본 이미지의 ENTRYPOINT는 다음 명령으로 확인할 수 있습니다.

```bash
docker inspect --format='{{json .Config.Entrypoint}}' <IMAGE>:<TAG>
```

| 이미지 태그 | 설명 |
|------------|------|
| `<TAG>` | 원본 이미지 (CVE 포함) |
| `<TAG>-work` | 작업용 태그 |
| `<TAG>-hardened` | CVE 제거 완료, bash ENTRYPOINT |
| `<TAG>-hardened-release` | CVE 제거 + 서비스 ENTRYPOINT 복구 |

---

## 8. 서비스 실행

runtime 이미지로 서비스를 실행합니다.

```bash
docker rm -f <CONTAINER_NAME> 2>/dev/null || true

docker run -d \
  --name <CONTAINER_NAME> \
  --gpus '"device=0"' \
  -p <HOST_PORT>:<CONTAINER_PORT> \
  -v /path/to/data:/data \
  <IMAGE>:<TAG>-hardened-release
```

> GPU 옵션, 포트, 볼륨 마운트 등은 서비스에 맞게 조정

---

## 9. 최종 검증

### 1. 서비스 정상 동작 확인

```bash
# 컨테이너 로그
docker logs -f <CONTAINER_NAME>

# GPU 할당 확인 (GPU 사용 시)
docker exec -it <CONTAINER_NAME> nvidia-smi

# API 응답 확인 (API 서버인 경우)
curl http://localhost:<HOST_PORT>/health
```

CVE 패치 과정에서 Python 패키지 버전이 변경되면  
deprecated된 함수나 변경된 API로 인해 기존 코드가 동작하지 않을 수 있습니다.  
컨테이너 기동 확인 외에 주요 기능에 대한 호환성 검증도 필요합니다.

| 확인 항목 | 방법 |
|----------|------|
| import 오류 | 컨테이너 내부에서 `python -c "import <패키지명>"` 실행 |
| 기능 테스트 | 핵심 API 엔드포인트 호출 및 응답 확인 |
| 로그 확인 | DeprecationWarning, AttributeError 등 경고/에러 유무 |

> `--no-deps`로 대상 패키지만 교체하므로 마이너 패치에서는 문제가 드물지만  
> 메이저/마이너 버전이 변경된 경우 함수 시그니처나 모듈 구조가 달라질 수 있으므로  
> 패치 전후 changelog를 확인하는 것을 권장

---

### 2. CVE 스캔 결과 추출

보안 검수 제출용 증빙 파일을 추출합니다.

```bash
# 작업용 컨테이너에서 저장한 파일 추출
docker cp cve-work:/tmp/trivy_scan_result.txt ./
docker cp cve-work:/tmp/python_versions.txt ./
```

제출 증빙 목록:

| 파일 | 내용 |
|------|------|
| `trivy_scan_result.txt` | Trivy 스캔 결과 (HIGH/CRITICAL = 0) |
| `python_versions.txt` | 패치 완료된 Python 패키지 버전 목록 |

---

## 마무리

이 글에서 다룬 CVE 대응 절차를 정리하면 다음과 같습니다.

```
1. 외부망: Trivy 바이너리 + DB + Python whl 준비
    ↓
2. 폐쇄망: 작업용 컨테이너 생성 (bash ENTRYPOINT)
    ↓
3. 컨테이너 내부: OS 패키지 제거 + Python 패키지 업데이트
    ↓
4. Trivy 오프라인 스캔 (HIGH/CRITICAL = 0 확인)
    ↓
5. 이미지 커밋 → 배포용 이미지 빌드 (ENTRYPOINT 복구)
    ↓
6. 서비스 실행 및 최종 검증
```

| 단계 | 핵심 |
|------|------|
| 외부망 준비 | 스캐너, DB, 패치 파일을 1회 준비하여 반입 |
| 작업용/배포용 분리 | 작업용과 실행용 이미지를 분리하여 ENTRYPOINT 충돌 방지 |
| 오프라인 스캔 | Trivy DB를 수동 반입하여 인터넷 없이 스캔 |
| 증빙 관리 | 스캔 결과와 패키지 버전을 파일로 저장하여 검수 대응 |

폐쇄망에서는 CVE 하나를 수정하는 것도  
"whl 다운로드 → 물리 매체 반입 → 설치 → 재스캔"의 사이클을 거쳐야 합니다.  
외부망에서는 한 줄이면 끝나는 작업이 폐쇄망에서는 하루가 걸릴 수 있습니다.

사전에 어떤 CVE가 보고되는지 파악하고,  
필요한 패치 파일을 한 번에 준비하여 반입하는 것이 재작업을 줄이는 핵심입니다.
