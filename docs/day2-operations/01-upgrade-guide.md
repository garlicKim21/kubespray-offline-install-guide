# Kubespray 클러스터 업그레이드 가이드

오프라인 환경에서 Kubespray를 사용한 Kubernetes 클러스터 업그레이드 절차를 설명합니다.

## 1. 업그레이드 개요

### 1.1. 업그레이드 유형

| 유형 | 설명 | 예시 |
|------|------|------|
| **Kubernetes 버전 업그레이드** | 동일 Kubespray 버전에서 K8s 버전만 변경 | K8s 1.32.1 → 1.32.3 |
| **Kubespray 버전 업그레이드** | Kubespray 버전 변경 (K8s 버전도 함께 변경) | Kubespray 2.28.1 → 2.29.1 |

### 1.2. 업그레이드 원칙

> ⚠️ **중요**: 마이너 버전을 건너뛰지 마세요!

```
✅ 올바른 업그레이드 경로:
   v2.27.0 → v2.28.0 → v2.29.0

❌ 잘못된 업그레이드 경로:
   v2.27.0 → v2.29.0 (v2.28.x 건너뜀)
```

패치 버전은 건너뛸 수 있습니다:
```
✅ v2.28.0 → v2.28.3 → v2.29.1
```

## 2. 사전 준비

### 2.1. 업그레이드 전 체크리스트

| # | 항목 | 확인 |
|---|------|------|
| 1 | 현재 클러스터 상태 정상 확인 | ☐ |
| 2 | etcd 백업 완료 | ☐ |
| 3 | 새 버전 바이너리 파일 서버에 업로드 | ☐ |
| 4 | 새 버전 컨테이너 이미지 프록시 캐시 확인 | ☐ |
| 5 | custom-config.yml 호환성 검토 | ☐ |
| 6 | 릴리즈 노트 확인 | ☐ |
| 7 | **오프라인 리소스 검증 완료** (verify-offline-resources.yml) | ☐ |
| 8 | **연결성 테스트 완료** (facts.yml) | ☐ |

### 2.2. 클러스터 상태 확인

```bash
# 노드 상태 확인
kubectl get nodes

# 모든 노드가 Ready 상태인지 확인
kubectl get nodes -o wide

# 시스템 Pod 상태 확인
kubectl get pods -n kube-system

# etcd 클러스터 상태 확인
kubectl get pods -n kube-system -l component=etcd
```

### 2.3. etcd 백업

업그레이드 전 반드시 etcd를 백업합니다.

```bash
# Control Plane 노드에서 실행
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 백업 파일 확인
ETCDCTL_API=3 etcdctl snapshot status /tmp/etcd-backup-$(date +%Y%m%d).db
```

### 2.4. 새 버전 파일 준비

#### 2.4.1. 바이너리 파일 준비

온라인 환경에서 새 버전의 바이너리 파일을 다운로드합니다.

```bash
# 새 Kubespray 버전으로 체크아웃
cd kubespray
git checkout v2.29.1

# 파일 목록 생성
cd contrib/offline
bash generate_list.sh

# 파일 다운로드
export FILES_LIST=temp/files.list
NO_HTTP_SERVER=true bash manage-offline-files.sh

# 버전별 압축
mv offline-files.tar.gz offline-files-v2.29.1.tar.gz
```

#### 2.4.2. 파일 서버 업데이트

```bash
# 레포 서버에 파일 전송
scp offline-files-v2.29.1.tar.gz root@<repo-server>:/root/

# 레포 서버에서 압축 해제
ssh root@<repo-server>
mkdir -p /root/offline-files/v2.29.1
tar -xvf offline-files-v2.29.1.tar.gz \
  --strip-components=1 \
  -C /root/offline-files/v2.29.1
```

### 2.5. custom-config.yml 호환성 검토

Kubespray 버전 간 변수 변경사항을 확인합니다.

```bash
# 기본 변수 파일 비교
diff -u kubespray-2.28.1/roles/kubespray_defaults/defaults/main/main.yml \
        kubespray-2.29.1/roles/kubespray_defaults/defaults/main/main.yml

# download 관련 변수 비교
diff -u kubespray-2.28.1/roles/kubespray_defaults/defaults/main/download.yml \
        kubespray-2.29.1/roles/kubespray_defaults/defaults/main/download.yml
```

**확인 포인트:**
- 제거된 변수가 custom-config.yml에 있는지
- 새로 추가된 필수 변수가 있는지
- 변수명이 변경된 것이 있는지

버전별 상세 변경사항은 [upgrade-notes/](upgrade-notes/) 디렉토리를 참조하세요.

## 3. 업그레이드 실행

### 3.1. Kubespray 소스 준비

#### 3.1.1. 새 버전 압축 해제

`sources/kubespray` 디렉토리에서 새 버전의 Kubespray를 압축 해제합니다.

```bash
cd /path/to/kubespray-offline-install-guide/sources/kubespray

# 새 버전 압축 해제
tar -xzf kubespray-2.29.1.tar.gz

# 디렉토리 구조 확인
ls -la
# kubespray-2.28.1/
# kubespray-2.29.1/  <- 새로 생성됨
```

#### 3.1.2. Python 가상환경 설정

> ⚠️ **중요**: Kubespray 버전마다 의존성이 다르므로, 새 버전에 맞는 가상환경을 설정해야 합니다.

```bash
# 작업 디렉토리 설정
KUBESPRAY_BASE=/path/to/kubespray-offline-install-guide/sources/kubespray
KUBESPRAY_VERSION=2.29.1
VENVDIR=${KUBESPRAY_BASE}/venv-${KUBESPRAY_VERSION}
KUBESPRAYDIR=${KUBESPRAY_BASE}/kubespray-${KUBESPRAY_VERSION}

# 가상환경 생성 및 활성화
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate

# 의존성 설치
cd $KUBESPRAYDIR
pip install -U pip
pip install -r requirements.txt

# 설치 확인
ansible --version
```

**참고**: 버전별로 별도의 가상환경(`venv-2.28.1`, `venv-2.29.1`)을 유지하면 롤백 시에도 유용합니다.

### 3.2. 인벤토리 준비

기존 인벤토리를 새 Kubespray 버전 디렉토리로 복사합니다.

```bash
# 가상환경이 활성화된 상태에서 진행
cd $KUBESPRAYDIR

# 기존 인벤토리 복사
cp -r ../kubespray-2.28.1/inventory/offline-test inventory/

# custom-config.yml 버전 업데이트
cd inventory/offline-test
```

`custom-config.yml`에서 버전 관련 설정을 업데이트합니다:

```yaml
# 변경 전
kubespray_version: "v2.28.1"

# 변경 후
kubespray_version: "v2.29.1"
```

### 3.3. 사전 검증 (권장)

실제 업그레이드 전에 문제가 없는지 확인하는 단계입니다.

#### 3.3.1. `--check` 모드의 한계

> ⚠️ **주의**: Ansible의 `--check` 모드는 dry-run으로 동작하지만, **이전 태스크의 결과에 의존하는 후속 태스크**에서 실패할 수 있습니다.

예를 들어:
1. 바이너리 다운로드 태스크 → `--check`에서는 실제 다운로드 안 함
2. 다운로드된 파일 복사 태스크 → 파일이 없어서 실패

```
TASK [download : Prep_kubeadm_images | Copy kubeadm binary from download dir to system path]
fatal: [...]: FAILED! => {"msg": "Source /tmp/releases/kubeadm-x.x.x-amd64 not found"}
```

이는 **예상된 동작**이며, 실제 배포 시에는 문제가 없습니다.

#### 3.3.2. 권장 검증 순서

`--check` 모드 대신 다음 순서로 검증하는 것을 권장합니다:

**Step 1: 오프라인 리소스 검증**

파일 서버와 레지스트리에 필요한 리소스가 있는지 확인합니다.

```bash
cd $KUBESPRAYDIR

# 리소스 검증 플레이북 실행 (기본값 사용)
ansible-playbook ../../../playbooks/verify-offline-resources.yml

# 또는 Kubernetes 버전 지정
ansible-playbook ../../../playbooks/verify-offline-resources.yml \
  -e kube_version=1.33.7
```

> 💡 이 플레이북은 localhost에서만 실행되므로 `-i` 옵션(인벤토리)은 불필요합니다.

이 플레이북은 다음을 검증합니다:
- 파일 서버 접근 가능 여부
- 필수 바이너리(kubeadm, kubelet, kubectl) 존재 여부
- 컨테이너 레지스트리 접근 가능 여부
- 필수 컨테이너 이미지 존재 여부

**Step 2: 문법 검사**

```bash
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --syntax-check
```

**Step 3: 연결성 테스트**

```bash
ansible-playbook playbooks/facts.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml
```

**Step 4: Control Plane 1대만 먼저 테스트 (선택사항)**

안전하게 진행하려면 Control Plane 노드 1대만 먼저 업그레이드합니다:

```bash
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --limit "ctrl01.example.com"
```

#### 3.3.3. --check 모드 (참고용)

위 검증을 모두 완료한 후, 참고용으로 `--check` 모드를 실행할 수 있습니다:

```bash
ansible-playbook \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --check \
  upgrade-cluster.yml
```

> 💡 **참고**: 다운로드 관련 태스크에서 실패해도, 이전 검증 단계를 통과했다면 실제 배포는 성공합니다.

### 3.4. 업그레이드 실행

```bash
cd $KUBESPRAYDIR

ansible-playbook \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  upgrade-cluster.yml
```

### 3.5. 단계별 업그레이드 (권장)

대규모 클러스터에서는 단계별 업그레이드를 권장합니다.

#### 3.5.1. Facts 캐시 갱신

```bash
ansible-playbook playbooks/facts.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml
```

#### 3.5.2. Control Plane + etcd 먼저 업그레이드

```bash
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --limit "kube_control_plane:etcd"
```

#### 3.5.3. Worker 노드 업그레이드

```bash
# 특정 노드만
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --limit "node1:node2"

# 또는 패턴으로
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --limit "worker*"
```

### 3.6. 업그레이드 속도 제어

동시에 업그레이드되는 노드 수를 제어할 수 있습니다.

```bash
# 한 번에 1개 노드씩 업그레이드
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  -e "serial=1"
```

### 3.7. 일시 중지 옵션

업그레이드 중 수동 확인이 필요한 경우:

```bash
# 각 노드 업그레이드 전 확인 (yes 입력 필요)
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  -e "upgrade_node_confirm=true"

# 각 노드 업그레이드 전 60초 대기
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  -e "upgrade_node_pause_seconds=60"
```

## 4. 업그레이드 검증

### 4.1. 버전 확인

```bash
# Kubernetes 버전 확인
kubectl version

# 노드별 버전 확인
kubectl get nodes -o wide
```

### 4.2. 클러스터 상태 확인

```bash
# 노드 상태
kubectl get nodes

# 시스템 Pod 상태
kubectl get pods -n kube-system

# 컴포넌트 상태
kubectl get componentstatuses
```

### 4.3. 워크로드 확인

```bash
# 모든 네임스페이스의 Pod 상태
kubectl get pods -A

# 비정상 Pod 확인
kubectl get pods -A | grep -v Running | grep -v Completed
```

## 5. 트러블슈팅

### 5.1. 업그레이드 실패 시 재실행

Kubespray는 멱등성을 보장하므로 실패 시 동일 명령어로 재실행할 수 있습니다.

```bash
# 동일 명령어로 재실행
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml
```

### 5.2. 특정 노드만 재실행

```bash
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  --limit "failed-node-name"
```

### 5.3. 일반적인 오류

| 오류 | 원인 | 해결 |
|------|------|------|
| `Source /tmp/releases/xxx not found` (--check 모드) | `--check` 모드의 한계 | 예상된 동작. [3.3.1 참조](#331-check-모드의-한계) |
| `Source /tmp/releases/xxx not found` (실제 실행) | 바이너리 다운로드 실패 | 파일 서버 URL 확인, `verify-offline-resources.yml` 실행 |
| `ImagePullBackOff` | 컨테이너 이미지 pull 실패 | 프록시 레지스트리 확인 |
| `connection refused` | 노드 접근 불가 | 노드 상태 및 SSH 확인 |
| `etcd cluster is unhealthy` | etcd 쿼럼 문제 | etcd 멤버 상태 확인 |

### 5.4. 롤백

> ⚠️ **주의**: Kubespray는 공식적인 롤백 기능을 제공하지 않습니다.

롤백이 필요한 경우:
1. etcd 백업에서 복원
2. 이전 버전 Kubespray로 클러스터 재배포

## 6. 버전별 업그레이드 노트

특정 버전 간 업그레이드 시 주의사항은 아래 문서를 참조하세요:

- [2.28.x → 2.29.x 업그레이드 노트](upgrade-notes/2.28-to-2.29.md)

## 부록: 유용한 명령어

### A.1. 컴포넌트별 업그레이드

```bash
# Docker/Containerd만 업그레이드
ansible-playbook cluster.yml --tags=docker

# etcd만 업그레이드
ansible-playbook cluster.yml --tags=etcd

# kubelet만 업그레이드
ansible-playbook cluster.yml --tags=node --skip-tags=k8s-gen-certs

# Control Plane 컴포넌트만 업그레이드
ansible-playbook cluster.yml --tags=master

# 네트워크 플러그인만 업그레이드
ansible-playbook cluster.yml --tags=network

# 애드온만 업그레이드
ansible-playbook cluster.yml --tags=apps
```

### A.2. 시스템 패키지 업그레이드

노드의 시스템 패키지도 함께 업그레이드하려면:

```bash
ansible-playbook upgrade-cluster.yml \
  -i inventory/offline-test/inventory.ini \
  -e @inventory/offline-test/custom-config.yml \
  -e "system_upgrade=true"
```

패키지 업그레이드 후 재부팅 옵션:
- `system_upgrade_reboot: on-upgrade` (기본값) - 패키지 변경 시만 재부팅
- `system_upgrade_reboot: always` - 항상 재부팅
- `system_upgrade_reboot: never` - 재부팅 안 함

