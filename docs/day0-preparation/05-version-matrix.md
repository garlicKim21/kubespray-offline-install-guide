# 버전 호환성 매트릭스

Kubespray, Kubernetes, Cilium 간의 버전 호환성 정보를 정리합니다.

## 1. Kubespray ↔ Kubernetes 버전

Kubespray 버전별 기본 Kubernetes 버전입니다.

| Kubespray | Kubernetes (기본) | Kubernetes (지원 범위) | 릴리즈 |
|-----------|------------------|----------------------|--------|
| v2.29.1 | 1.33.x | 1.31.x - 1.33.x | 2024.12 |
| v2.28.1 | 1.32.x | 1.30.x - 1.32.x | 2024.09 |
| v2.27.0 | 1.31.x | 1.29.x - 1.31.x | 2024.06 |
| v2.26.0 | 1.30.x | 1.28.x - 1.30.x | 2024.03 |

**참고**: [Kubespray Releases](https://github.com/kubernetes-sigs/kubespray/releases)

## 2. Kubespray ↔ Cilium 버전

Kubespray가 기본으로 배포하는 Cilium 버전입니다.

| Kubespray | Cilium (기본) | 비고 |
|-----------|--------------|------|
| v2.29.1 | 1.18.4 | |
| v2.28.1 | 1.17.7 | |
| v2.27.0 | 1.16.x | |
| v2.26.0 | 1.15.x | |

**참고**: `kube_network_plugin: 'cni'`로 설정 시 Kubespray는 Cilium을 설치하지 않으며, Helm으로 별도 관리 가능합니다.

## 3. Kubernetes ↔ Cilium 호환성

Cilium은 [Kubernetes Patch Releases](https://kubernetes.io/releases/patch-releases/)에서 정의한 **현재 지원되는 Kubernetes 버전**에 대해 e2e 테스트를 수행합니다.

### 현재 e2e 테스트된 Kubernetes 버전

| Cilium (Stable) | e2e 테스트된 Kubernetes |
|-----------------|------------------------|
| 1.18.x | 1.30, 1.31, 1.32, 1.33 |

**참고**: [Cilium Kubernetes Compatibility](https://docs.cilium.io/en/stable/network/kubernetes/compatibility/)

> 💡 **일반 원칙**: Cilium 릴리즈 브랜치가 생성된 시점에 지원되던 Kubernetes 버전들이 해당 릴리즈의 유지보수 기간 동안 테스트됩니다.

## 4. Cilium 시스템 요구사항

### Linux 커널 요구사항

| 요구사항 | 최소 버전 |
|----------|----------|
| **기본 요구사항** | Linux kernel >= 5.10 |
| **RHEL 8.10** | 4.18 (동등 기능 지원) |

**참고**: [Cilium System Requirements](https://docs.cilium.io/en/stable/operations/system_requirements/)

## 5. Cilium 지원 정책

Cilium은 **최근 3개 마이너 버전**을 지원합니다.

```
지원 타임라인 (예시):

v1.18.x ────────────────────────────────► (Stable)
v1.17.x ──────────────────────►           (지원 중)
v1.16.x ────────────►                     (지원 종료 예정)
v1.15.x ─────►                            (EOL)

        ├────────┼────────┼────────┼────────►
       Q1       Q2       Q3       Q4      시간
```

**업그레이드 권장 주기**: 6개월 ~ 1년

## 6. Cilium 기능별 커널 요구사항

특정 Cilium 기능 사용 시 추가 커널 요구사항입니다.

| 기능 | 최소 커널 | 비고 |
|------|----------|------|
| 기본 요구사항 | 5.10 (또는 RHEL 8.10의 4.18) | |
| Iptables-based Masquerading | 5.10 | |
| VXLAN Tunnel | 5.10 | |
| Geneve Tunnel | 5.10 | |
| IPsec | 5.10 | |
| Bandwidth Manager | 5.1 | EDT 기반 |
| Netkit Device Mode | 6.8 | 고성능 데이터패스 |

**참고**: [Required Kernel Versions for Advanced Features](https://docs.cilium.io/en/stable/operations/system_requirements/#required-kernel-versions-for-advanced-features)

## 7. Linux 배포판별 커널 버전

주요 배포판별 기본 커널 버전입니다.

| 배포판 | 커널 | Cilium 호환성 |
|--------|------|--------------|
| RHEL 10 | 6.12.x | ✅ 모든 기능 |
| RHEL 9.x | 5.14.x | ✅ 기본 요구사항 충족 |
| RHEL 8.10 | 4.18.x | ✅ 공식 지원 (동등 기능) |
| Ubuntu 24.04 | 6.8.x | ✅ 모든 기능 |
| Ubuntu 22.04 | 5.15.x | ✅ 기본 요구사항 충족 |

## 8. 버전 선택 가이드

### 신규 설치 시

```
1. Kubespray 최신 Stable 선택
       ↓
2. 해당 버전의 기본 Cilium 버전 확인
       ↓
3. 해당 Cilium 라인의 최신 패치 버전 선택
       ↓
4. 커널 요구사항 확인
```

### 업그레이드 시

```
1. 현재 Kubespray/Cilium 버전 확인
       ↓
2. 목표 Kubespray 버전 선택 (마이너 버전 순차 업그레이드)
       ↓
3. 목표 Cilium 버전 선택 (Kubernetes 호환성 확인)
       ↓
4. 테스트 환경에서 검증
       ↓
5. 프로덕션 적용
```

## 9. 호환성 검증 체크리스트

업그레이드 또는 신규 설치 전 확인 항목:

| # | 항목 | 확인 |
|---|------|------|
| 1 | Kubespray ↔ Kubernetes 버전 호환 | ☐ |
| 2 | Kubernetes ↔ Cilium 버전 호환 | ☐ |
| 3 | Linux 커널 ↔ Cilium 기능 호환 | ☐ |
| 4 | 사용 중인 Cilium 기능의 최소 버전 충족 | ☐ |
| 5 | Cilium Helm values 호환성 확인 | ☐ |

## 참고 자료

- [Kubespray Releases](https://github.com/kubernetes-sigs/kubespray/releases)
- [Cilium Releases](https://github.com/cilium/cilium/releases)
- [Cilium Upgrade Guide](https://docs.cilium.io/en/stable/operations/upgrade/)
- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)

