# Cilium Offline 설치 파일 준비

오프라인 환경에서 Cilium을 설치하기 위한 파일을 준비한다.

**사전 요구 사항:**
- 인터넷 연결이 가능한 환경 (파일 다운로드용)
- curl 또는 wget

## 1. 개요

### 1.1. 필요한 파일

| 구성 요소 | 설명 | 위치 |
|-----------|------|------|
| Cilium CLI | Cilium 관리 도구 | `sources/cilium/cli/` |
| Helm Chart | Cilium 배포 패키지 | `sources/cilium/helm-charts/` |
| Values 파일 | 커스텀 설정 | `sources/cilium/values/` |

### 1.2. 이 레포지토리 사용 시

이 레포지토리를 clone하면 `sources/cilium/` 디렉토리에 필요한 파일이 이미 포함되어 있다.

```bash
git clone <repository-url>
cd kubespray-offline-install-guide

# 포함된 파일 확인
tree sources/cilium/
```

**버전 업데이트가 필요한 경우** 아래 절차를 따라 파일을 다운로드한다.

## 2. Cilium CLI 다운로드

### 2.1. 최신 버전 확인 및 다운로드

```bash
# 작업 디렉토리
cd sources/cilium/cli

# 최신 stable 버전 확인
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
echo "CLI Version: $CILIUM_CLI_VERSION"

# 바이너리 다운로드
curl -L -o cilium-linux-amd64.tar.gz \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"

# 체크섬 다운로드
curl -L -o cilium-linux-amd64.tar.gz.sha256sum \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz.sha256sum"

# 버전 파일 생성
echo "$CILIUM_CLI_VERSION" > VERSION
```

### 2.2. 체크섬 검증

```bash
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
```

## 3. Cilium Helm Chart 다운로드

### 3.1. Helm Chart 직접 다운로드

Helm이 설치되어 있지 않아도 직접 다운로드할 수 있다.

```bash
# 작업 디렉토리
cd sources/cilium/helm-charts

# 초기 배포용 (Kubespray 2.28.1 환경)
CILIUM_VERSION_INITIAL=1.17.10
curl -L -o cilium-${CILIUM_VERSION_INITIAL}.tgz \
  "https://helm.cilium.io/cilium-${CILIUM_VERSION_INITIAL}.tgz"

# 업그레이드용 (Kubespray 2.29.1 환경)
CILIUM_VERSION_UPGRADE=1.18.5
curl -L -o cilium-${CILIUM_VERSION_UPGRADE}.tgz \
  "https://helm.cilium.io/cilium-${CILIUM_VERSION_UPGRADE}.tgz"
```

### 3.2. 다운로드 가능한 버전 확인

[Cilium Helm Repository](https://helm.cilium.io/)에서 사용 가능한 버전을 확인할 수 있다.

```bash
# Helm이 설치된 경우
helm repo add cilium https://helm.cilium.io/
helm search repo cilium/cilium --versions
```

## 4. Values 파일 관리

### 4.1. 기본 Values 파일 추출

Helm Chart에서 기본 values.yaml을 추출하여 참고할 수 있다.

```bash
cd sources/cilium

# 설치할 버전 설정
CILIUM_VERSION=1.18.5

# Chart 압축 해제 후 버전별 디렉토리로 이름 변경
tar -xzf helm-charts/cilium-${CILIUM_VERSION}.tgz
mv cilium cilium-${CILIUM_VERSION}

# 기본 values 확인
cat cilium-${CILIUM_VERSION}/values.yaml

# 정리
rm -rf cilium-${CILIUM_VERSION}/
```

### 4.2. 버전별 Values 파일

| 파일 | 용도 | Cilium 버전 |
|------|------|-------------|
| `values/cilium-values-1.17.yaml` | 초기 배포 | 1.17.x |
| `values/cilium-values-1.18.yaml` | 업그레이드 | 1.18.x |

### 4.3. Values 파일 수정 시 주의사항

버전 간 설정 차이가 있을 수 있으므로, 업그레이드 전 릴리즈 노트를 확인한다.

**주요 확인 항목:**
- Deprecated 설정 제거
- 새로운 필수 설정 추가
- 기본값 변경 사항

**참고:** [Cilium Upgrade Guide](https://docs.cilium.io/en/stable/operations/upgrade/)

## 5. 컨테이너 이미지 준비

### 5.1. 필요한 이미지 목록

Cilium 설치에 필요한 이미지는 Values 파일의 설정에 따라 달라진다.

```bash
# Helm template으로 이미지 목록 추출 (Helm 필요)
helm template cilium helm-charts/cilium-1.18.5.tgz \
  -f values/cilium-values-1.18.yaml \
  | grep "image:" | sort -u
```

### 5.2. 주요 이미지

| 이미지 | 용도 |
|--------|------|
| `quay.io/cilium/cilium` | Cilium Agent |
| `quay.io/cilium/operator-generic` | Cilium Operator |
| `quay.io/cilium/hubble-relay` | Hubble Relay |
| `quay.io/cilium/hubble-ui` | Hubble UI |
| `quay.io/cilium/hubble-ui-backend` | Hubble UI Backend |

### 5.3. 프록시 레지스트리 설정

오프라인 환경에서는 프록시 레지스트리를 통해 이미지를 제공한다.
상세 내용은 [05-proxy-container-registry.md](05-proxy-container-registry.md)를 참고한다.

## 6. 디렉토리 구조 확인

최종 디렉토리 구조는 다음과 같아야 한다.

```
sources/cilium/
├── cli/
│   ├── cilium-linux-amd64.tar.gz
│   ├── cilium-linux-amd64.tar.gz.sha256sum
│   └── VERSION
├── helm-charts/
│   ├── cilium-1.17.10.tgz
│   └── cilium-1.18.5.tgz
└── values/
    ├── cilium-values-1.17.yaml
    └── cilium-values-1.18.yaml
```

## 7. 다음 단계

- [05-proxy-container-registry.md](05-proxy-container-registry.md) - 프록시 레지스트리 설정
- [Day 1 - Cilium 설치](../day1-deployment/02-cilium-install.md) - 오프라인 환경에서 Cilium 설치

