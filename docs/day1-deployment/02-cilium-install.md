# Cilium 설치 가이드

오프라인 환경에서 Cilium CNI를 설치한다.

**사전 요구 사항:**
- [01-cluster-install.md](01-cluster-install.md) - Kubernetes 클러스터 설치 완료
- [Day 0 - Cilium 오프라인 준비](../day0-preparation/04-cilium-offline-prepare.md) 완료
- kubectl이 설치되어 있고 클러스터에 접근 가능한 노드

## 1. 파일 준비

### 1.1. 소스 디렉토리 확인

이 레포지토리의 `sources/cilium/` 디렉토리에서 필요한 파일을 확인한다.

```bash
# 레포지토리 루트로 이동
cd /path/to/kubespray-offline-install-guide

# 파일 확인
tree sources/cilium/
```

**예상 출력:**

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

### 1.2. 대상 노드로 파일 전송

kubectl을 실행할 노드(일반적으로 Control Plane)로 필요한 파일을 전송한다.

```bash
# 설치할 Cilium 버전 설정
CILIUM_VERSION=1.17.10  # 또는 1.18.5

# 파일 전송
TARGET_NODE=<control-plane-ip>

scp sources/cilium/cli/cilium-linux-amd64.tar.gz root@${TARGET_NODE}:/root/
scp sources/cilium/cli/cilium-linux-amd64.tar.gz.sha256sum root@${TARGET_NODE}:/root/
scp sources/cilium/helm-charts/cilium-${CILIUM_VERSION}.tgz root@${TARGET_NODE}:/root/
scp sources/cilium/values/cilium-values-${CILIUM_VERSION%.*}.yaml root@${TARGET_NODE}:/root/cilium-values.yaml
```

**참고:** `<control-plane-ip>`는 실제 대상 노드의 IP 주소로 변경한다.

## 2. Cilium CLI 설치

대상 노드에서 Cilium CLI를 설치한다.

### 2.1. 체크섬 검증

```bash
ssh root@${TARGET_NODE}
cd /root

# 체크섬 검증
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
```

### 2.2. 바이너리 설치

```bash
# 압축 해제 및 설치
tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin

# 설치 확인
cilium version --client

# 임시 파일 정리
rm -f cilium-linux-amd64.tar.gz cilium-linux-amd64.tar.gz.sha256sum
```

**예상 출력:**

```
cilium-cli: v0.18.9
```

## 3. Cilium 설치 (Helm)

### 3.1. Helm Chart 압축 해제

```bash
cd /root

# 설치할 버전
CILIUM_VERSION=1.17.10

# Chart 압축 해제 후 버전별 디렉토리로 이름 변경
tar xzf cilium-${CILIUM_VERSION}.tgz
mv cilium cilium-${CILIUM_VERSION}
```

### 3.2. Values 파일 확인

설치 전 Values 파일의 주요 설정을 확인한다.

```bash
# 주요 설정 확인
grep -E "^[a-zA-Z]|k8sServiceHost|k8sServicePort|clusterPoolIPv4PodCIDRList|routingMode" cilium-values.yaml
```

**필수 확인 항목:**

| 설정 | 설명 | 확인 사항 |
|------|------|----------|
| `k8sServiceHost` | API Server 주소 | 실제 VIP 또는 LB 주소 |
| `k8sServicePort` | API Server 포트 | 기본 6443 |
| `clusterPoolIPv4PodCIDRList` | Pod CIDR | 클러스터 설정과 일치 |

### 3.3. Cilium 설치

```bash
helm install cilium ./cilium-${CILIUM_VERSION} \
  --namespace kube-system \
  -f cilium-values.yaml
```

**Cilium CLI 사용 시 (Helm 미설치 환경):**

```bash
cilium install \
  --chart-directory ./cilium-${CILIUM_VERSION} \
  --helm-values ./cilium-values.yaml \
  --version ${CILIUM_VERSION}
```

### 3.4. 설치 확인

```bash
# Pod 상태 확인
kubectl get pods -n kube-system -l app.kubernetes.io/part-of=cilium

# Cilium 상태 확인 (CLI 사용)
cilium status --wait

# 모든 노드에서 Cilium Agent 실행 확인
kubectl get pods -n kube-system -l k8s-app=cilium -o wide
```

**예상 출력:**

```
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE
cilium-xxxxx   1/1     Running   0          5m    192.168.1.101   node1
cilium-yyyyy   1/1     Running   0          5m    192.168.1.102   node2
```

## 4. 연결성 테스트

### 4.1. Cilium 연결성 테스트

```bash
# 기본 연결성 테스트
cilium connectivity test

# 특정 테스트만 실행 (빠른 확인)
cilium connectivity test --test pod-to-pod
```

**참고:** 연결성 테스트는 테스트용 Pod를 생성하므로 몇 분 소요될 수 있다.

### 4.2. 수동 테스트

```bash
# 테스트 Pod 생성
kubectl run test-pod --image=busybox --rm -it --restart=Never -- sh

# Pod 내부에서 테스트
nslookup kubernetes.default.svc.cluster.local
ping -c 3 <other-pod-ip>
```

## 5. Hubble 확인 (활성화된 경우)

### 5.1. Hubble 상태 확인

```bash
cilium hubble status
```

### 5.2. Hubble UI 접근

```bash
# NodePort 서비스 확인
kubectl get svc -n kube-system hubble-ui

# Ingress 확인 (설정된 경우)
kubectl get ingress -n kube-system
```

**참고:** Values 파일에서 Hubble UI NodePort가 31235로 설정되어 있다.

## 6. 설치 후 정리

```bash
# Helm Chart 디렉토리 정리
rm -rf /root/cilium-${CILIUM_VERSION}/
rm -f /root/cilium-*.tgz
rm -f /root/cilium-values.yaml
```

## 7. 트러블슈팅

### 7.1. Pod가 Running 상태가 아닌 경우

```bash
# Pod 상세 정보 확인
kubectl describe pod -n kube-system -l k8s-app=cilium

# Pod 로그 확인
kubectl logs -n kube-system -l k8s-app=cilium --tail=100
```

### 7.2. API Server 연결 실패

**증상:** `Unable to connect to API server`

**해결:**

```bash
# API Server 엔드포인트 확인
kubectl get endpoints kubernetes -n default

# Values 파일의 k8sServiceHost/Port 설정 확인
grep -E "k8sServiceHost|k8sServicePort" cilium-values.yaml
```

### 7.3. IPAM 할당 실패

**증상:** `IPAM allocation failed`

**해결:**

```bash
# 노드의 Pod CIDR 확인
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.podCIDR}{"\n"}{end}'

# Values 파일의 IPAM 설정 확인
grep -A5 "ipam:" cilium-values.yaml
```

### 7.4. 이미지 Pull 실패

**증상:** `ImagePullBackOff` 또는 `ErrImagePull`

**해결:**

```bash
# 이미지 정보 확인
kubectl get pods -n kube-system -l k8s-app=cilium -o jsonpath='{.items[*].spec.containers[*].image}'

# containerd 레지스트리 설정 확인
cat /etc/containerd/config.toml | grep -A10 "registry.mirrors"
```

프록시 레지스트리 설정은 [05-proxy-container-registry.md](../day0-preparation/05-proxy-container-registry.md)를 참고한다.

## 8. 다음 단계

- [Day 2 - Cilium 업그레이드](../day2-operations/02-cilium-upgrade.md) - Cilium 버전 업그레이드
