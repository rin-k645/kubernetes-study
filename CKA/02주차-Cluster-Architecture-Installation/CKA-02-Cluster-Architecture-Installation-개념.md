---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 02 - Cluster Architecture & Installation (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 02 - Cluster Architecture & Installation (실습)](CKA-02-Cluster-Architecture-Installation-실습.md) | 이전: [CKA 01 - 기초 개념 & 환경 세팅 (개념)](../01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-개념.md) | 다음: [CKA 03 - Workloads & Scheduling (개념)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-개념.md)

---

## 이 주차가 왜 중요한가

Cluster Architecture, Installation & Configuration은 CKA 시험 비중 **25%**로 Troubleshooting(30%) 다음으로 크다. 게다가 이 영역의 상당수는 **etcd 백업/복구**와 **kubeadm 업그레이드**라는 두 가지 시나리오에 집중되어 있어서, 개념을 정확히 알고 명령어를 손에 익히면 실제 시험에서 안정적으로 점수를 확보할 수 있는 "가성비 좋은" 영역이다.

### 한 줄 요약

> "kubeadm이 클러스터를 어떻게 만들고, etcd가 그 상태를 어떻게 저장하며, 그 둘을 어떻게 안전하게 백업/업그레이드하는지를 다루는 주차다."

---

## kubeadm 기반 클러스터 설치

### kubeadm이 하는 일

`kubeadm`은 Kubernetes 클러스터를 **부트스트랩**하는 공식 도구다. 클러스터 자체를 실행하는 것이 아니라, 클러스터가 스스로 실행될 수 있도록 필요한 설정 파일과 인증서, 컨트롤 플레인 컴포넌트를 준비해주는 역할을 한다.

```
kubeadm의 역할
 ├── 사전 점검 (preflight checks)
 ├── 클러스터 CA 및 각종 인증서 생성
 ├── kubeconfig 파일 생성 (admin, kubelet, controller-manager, scheduler용)
 ├── 컨트롤 플레인 컴포넌트를 Static Pod로 배치
 ├── etcd를 Static Pod로 배치 (기본: stacked etcd)
 └── 클러스터 조인용 토큰 발급
```

### `kubeadm init` 내부 동작 순서

```
1. Preflight Checks
   - swap 비활성화 여부, 포트 충돌, cgroup driver 일치 여부 확인

2. Certificate 생성 (/etc/kubernetes/pki/)
   - CA 인증서(ca.crt, ca.key) 생성 — 클러스터의 신뢰 루트
   - API Server 인증서, kubelet 클라이언트 인증서 등 파생 인증서 생성
   - etcd 전용 CA와 인증서도 별도로 생성 (etcd/ca.crt 등)

3. kubeconfig 파일 생성 (/etc/kubernetes/)
   - admin.conf, kubelet.conf, controller-manager.conf, scheduler.conf

4. Static Pod manifest 생성 (/etc/kubernetes/manifests/)
   - kube-apiserver.yaml
   - kube-controller-manager.yaml
   - kube-scheduler.yaml
   - etcd.yaml (stacked etcd 구성일 때)

5. kubelet이 manifest 경로를 감시하다가 위 파일들을 읽고
   컨트롤 플레인 컴포넌트들을 Pod로 기동

6. 컴포넌트들이 정상 기동하면 클러스터가 "Ready" 상태로 전환

7. join 토큰과 CA 인증서 해시를 출력 (worker 노드 조인용)
```

핵심은 **kubeadm이 직접 컨트롤 플레인 프로세스를 실행하는 게 아니라, kubelet에게 "이 매니페스트대로 Pod를 띄워라"고 지시만 한다는 점**이다. 이 원리는 뒤에 나오는 Static Pod 개념과 직결된다.

### `kubeadm join` 내부 동작

```
Worker 노드에서 kubeadm join 실행 시:

1. --discovery-token-ca-cert-hash로 받은 해시로
   control plane의 CA 인증서를 검증 (중간자 공격 방지)

2. Bootstrap Token으로 API Server에 인증

3. kubelet 인증서 서명 요청(CSR)을 자동 생성/승인
   (TLS Bootstrapping)

4. kubelet.conf 생성 후 kubelet 서비스 시작

5. kubelet이 API Server에 자신을 Node 오브젝트로 등록
```

- Bootstrap Token은 기본 **24시간** 후 만료된다. 오래된 경우 `kubeadm token create --print-join-command`로 새 조인 명령을 발급해야 한다.
- worker 노드에는 컨트롤 플레인 인증서(CA의 **개인키**)가 배포되지 않는다. CA 인증서(공개 부분)의 해시만으로 신뢰를 검증한다.

---

## etcd — 클러스터의 유일한 진실 공급원

### etcd란

etcd는 **분산 key-value 저장소**로, Kubernetes 클러스터의 **모든 상태(state)**를 저장하는 유일한 데이터베이스다. Pod, Deployment, Secret, ConfigMap 등 `kubectl get`으로 조회 가능한 모든 오브젝트가 결국 etcd에 저장된 데이터다.

```
kubectl / API Server 관점에서의 데이터 흐름

[kubectl apply] → [kube-apiserver] → [etcd에 저장]
                        ↑                  |
                        └──── watch ───────┘
                   (Controller들이 변경을 감지)
```

- **etcd에 직접 쓰거나 읽는 컴포넌트는 kube-apiserver뿐**이다. Scheduler, Controller Manager, kubelet 모두 API Server를 거쳐서만 상태를 조회/변경한다.
- Raft 합의 알고리즘을 사용하며, 홀수 개(3, 5, 7...) 노드로 구성해야 과반수(quorum) 기반의 장애 허용이 가능하다.
- etcd가 죽으면 클러스터의 "기억"이 사라진다 — 이미 떠 있는 Pod는 계속 동작할 수 있지만, 새 스케줄링/스케일링/설정 변경이 전부 멈춘다. **etcd 백업은 클러스터 재해복구의 핵심**이다.

### etcd 백업(스냅샷)의 원리

`etcdctl snapshot save`는 etcd의 현재 데이터를 **단일 파일(.db)**로 직렬화하는 명령이다. 내부적으로는 etcd가 유지하는 BoltDB 기반 저장소의 일관된 스냅샷을 가져오는 것과 같다.

```
etcd 데이터 디렉토리 (--data-dir)
        │
        │ snapshot save
        ▼
   snapshot.db  (특정 시점의 전체 key-value 상태를 담은 파일)
```

- 스냅샷은 **특정 시점(point-in-time)**의 전체 상태를 담는다. 증분 백업이 아니다.
- 스냅샷 파일 하나만 있으면 클러스터 전체 상태(모든 네임스페이스, 모든 리소스, Secret 포함)를 복원할 수 있다. → **보안적으로 매우 민감한 파일**이므로 접근 권한 관리가 중요하다.
- etcdctl은 API Server를 거치지 않고 **etcd에 직접** TLS로 연결한다. 그래서 명령 실행 시 반드시 인증서 3종(CA, 클라이언트 인증서, 클라이언트 키)과 endpoint를 지정해야 한다.

```
etcdctl이 etcd에 접속하기 위해 필요한 것
 ├── --endpoints        : etcd가 리스닝하는 주소 (기본 127.0.0.1:2379)
 ├── --cacert           : etcd CA 인증서 (etcd/ca.crt)
 ├── --cert             : 클라이언트 인증서 (etcd/server.crt 또는 peer 인증서)
 └── --key              : 클라이언트 키
```

이 경로들은 kube-apiserver의 static pod manifest(`/etc/kubernetes/manifests/etcd.yaml`)를 열어보면 `--cert-file`, `--key-file`, `--trusted-ca-file` 옵션으로 정확히 확인할 수 있다. **시험에서는 항상 이 매니페스트를 먼저 확인하는 습관**이 필요하다.

### 복원(restore)의 원리

`etcdctl snapshot restore`는 스냅샷 파일을 읽어 **완전히 새로운 데이터 디렉토리**를 생성한다. 기존 etcd 프로세스를 덮어쓰는 것이 아니라, 별도 디렉토리에 새 데이터를 풀어놓고, 이후 etcd Static Pod manifest가 그 **새 디렉토리를 바라보도록 수정**해야 한다.

```
1. etcdctl snapshot restore snapshot.db --data-dir=/var/lib/etcd-restore
     → 새로운 데이터 디렉토리 생성 (기존 etcd는 건드리지 않음)

2. /etc/kubernetes/manifests/etcd.yaml 의
   hostPath volume 경로를 --data-dir=/var/lib/etcd-restore 로 변경

3. kubelet이 manifest 변경을 감지 → etcd Static Pod 재시작
   → 새 데이터 디렉토리로 etcd가 재기동되며 스냅샷 시점 상태로 복원됨
```

복원은 반드시 **컨트롤 플레인 노드에서, API Server가 접근하는 etcd와 동일한 클러스터 멤버 구성**으로 이루어져야 하며, 복원 후에는 기존에 있던 리소스 중 스냅샷 이후에 생성/삭제된 것들이 스냅샷 시점 상태로 되돌아간다는 점을 이해해야 한다.

---

## 클러스터 업그레이드 절차

### 왜 순서가 중요한가

Kubernetes는 **컨트롤 플레인이 워커 노드보다 항상 같거나 높은 버전**이어야 정상 동작을 보장한다(API 하위 호환성). 따라서 업그레이드는 항상 다음 순서를 따른다.

```
업그레이드 순서

1순위: kubeadm 도구 자체 업그레이드 (control plane 노드)
2순위: control plane 컴포넌트 업그레이드 (kubeadm upgrade apply)
       - API Server, Controller Manager, Scheduler, etcd
3순위: control plane 노드의 kubelet, kubectl 업그레이드
4순위: (다중 control plane인 경우) 나머지 control plane 노드 반복
5순위: 각 worker 노드를 하나씩 drain → kubeadm upgrade node
       → kubelet 업그레이드 → uncordon
```

- 한 번에 **하나의 마이너 버전(minor version)만** 올릴 수 있다. 예: 1.28 → 1.29는 가능하지만 1.28 → 1.30은 불가 (반드시 1.29를 거쳐야 함).
- `kubeadm upgrade plan`은 현재 버전과 업그레이드 가능한 버전, 각 컴포넌트의 예상 변경 사항을 **미리 보여주기만** 하고 실제 변경은 하지 않는다.
- `kubeadm upgrade apply <version>`이 실제로 컨트롤 플레인 컴포넌트의 Static Pod manifest를 새 버전으로 교체하고, 인증서 갱신(만료 임박 시) 등을 수행한다.
- worker 노드는 `kubeadm upgrade apply`가 아니라 `kubeadm upgrade node`를 사용한다 (컨트롤 플레인 설정을 새로 만드는 게 아니라 kubelet 설정만 갱신).

### drain이 필요한 이유

```
kubectl drain <node>
 ├── 해당 노드를 스케줄링 불가로 표시 (cordon과 동일 효과)
 ├── 노드 위의 Pod들을 안전하게 축출(evict)
 │     - PodDisruptionBudget을 존중하며 순차적으로 제거
 │     - DaemonSet Pod는 기본적으로 무시 (--ignore-daemonsets 필요)
 └── 축출된 Pod는 Deployment/ReplicaSet에 의해 다른 노드에 재스케줄링
```

업그레이드 중 kubelet이 재시작되면 해당 노드의 Pod들이 순간적으로 불안정해질 수 있으므로, 미리 다른 노드로 옮겨놓고(drain) 작업한 뒤 끝나면 다시 스케줄링을 허용(uncordon)한다.

---

## Static Pod

### 개념

Static Pod는 **API Server가 아니라 kubelet이 직접 관리하는 Pod**다. kubelet은 지정된 디렉토리(기본 `/etc/kubernetes/manifests/`)를 주기적으로 감시(watch)하다가, 그 안에 있는 YAML 파일을 읽어 Pod를 생성하고, 파일이 삭제되면 Pod도 삭제한다.

```
kubelet 설정(--pod-manifest-path 또는 config의 staticPodPath)
        │
        ▼
/etc/kubernetes/manifests/
 ├── kube-apiserver.yaml         → kubelet이 감시 → Pod 생성
 ├── kube-controller-manager.yaml
 ├── kube-scheduler.yaml
 └── etcd.yaml
```

- Static Pod는 API Server 없이도 동작해야 하므로(닭이 먼저냐 달걀이 먼저냐 문제), kubeadm은 **컨트롤 플레인 컴포넌트 자체를 Static Pod로 배치**한다. API Server가 없어도 kubelet 혼자서 이 컴포넌트들을 띄울 수 있기 때문이다.
- kubelet은 각 Static Pod에 대응하는 **미러 Pod(mirror pod)**를 API Server에 등록한다. 그래서 `kubectl get pods -n kube-system`으로 `kube-apiserver-<node>` 같은 Pod가 보이지만, 이걸 `kubectl delete pod`로 지워도 kubelet이 manifest를 보고 즉시 재생성한다 (수정하려면 manifest 파일 자체를 고쳐야 함).
- Static Pod 이름은 자동으로 `<pod-name>-<node-hostname>` 형식이 된다.

---

## 인증서 관리

### 클러스터 CA와 인증서 체계

```
클러스터 CA (/etc/kubernetes/pki/ca.crt, ca.key)
   │
   ├── API Server 인증서 (apiserver.crt)
   ├── API Server 클라이언트용 kubelet 인증서
   ├── kubeadm이 생성한 각종 kubeconfig 내 클라이언트 인증서
   └── etcd 전용 CA (별도, /etc/kubernetes/pki/etcd/ca.crt)
        ├── etcd 서버 인증서
        └── etcd peer 인증서 (etcd 멤버 간 통신용)
```

- kubeadm으로 만든 클러스터의 인증서는 기본적으로 **1년** 유효기간을 가진다 (CA 자체는 10년).
- 인증서가 만료되면 컨트롤 플레인 컴포넌트 간 통신이 끊어져 `kubectl` 명령이 전부 실패하는 심각한 장애로 이어진다.
- `kubeadm certs check-expiration`으로 모든 인증서의 만료일을 확인할 수 있고, `kubeadm certs renew all`로 갱신할 수 있다. `kubeadm upgrade apply`를 수행하면 자동으로 인증서도 갱신된다.

### kubeconfig 구조

kubeconfig 파일은 **어떤 클러스터에, 누구로, 어떻게 인증해서 접속할지**를 정의하는 YAML이다.

```
kubeconfig
 ├── clusters:      접속할 클러스터 목록 (API Server 주소, CA 인증서)
 ├── users:         인증 정보 (클라이언트 인증서/키, 토큰 등)
 ├── contexts:      (cluster + user + namespace)의 조합에 이름을 붙인 것
 └── current-context: 현재 활성화된 context 이름
```

```
                context = "my-context"
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
    cluster       user       namespace
  (어디로 접속?) (누구로 인증?) (기본 네임스페이스)
```

- `kubectl config get-contexts` / `use-context` / `set-context` 로 여러 클러스터를 오가며 작업한다. CKA 시험은 문제마다 다른 컨텍스트를 지정하는 경우가 많아 이 부분이 매우 중요하다.
- `KUBECONFIG` 환경 변수에 콜론(`:`)으로 여러 파일 경로를 나열하면 병합(merge)된 뷰로 다룰 수 있다.

---

## RBAC (Role-Based Access Control)

### 핵심 개념 4종

```
"누가"                "무엇을 할 수 있는가"
Subject      ──바인딩──   Role / ClusterRole
 ├── User
 ├── Group
 └── ServiceAccount
```

| 오브젝트 | 범위 | 정의하는 것 |
|---|---|---|
| Role | 네임스페이스 단위 | 특정 네임스페이스 안의 리소스에 대한 권한(verb) 목록 |
| ClusterRole | 클러스터 전체 (네임스페이스 없음) | 클러스터 범위 리소스(Node, PV 등) 권한 또는 여러 네임스페이스에 재사용할 권한 |
| RoleBinding | 네임스페이스 단위 | 특정 네임스페이스에서 Subject에게 Role(또는 ClusterRole)을 부여 |
| ClusterRoleBinding | 클러스터 전체 | 모든 네임스페이스에서 Subject에게 ClusterRole을 부여 |

- **Role vs ClusterRole**: Namespace 리소스(Pod, Deployment 등)는 Role로 충분하지만, Node나 PersistentVolume처럼 네임스페이스가 없는 클러스터 범위 리소스는 반드시 ClusterRole이 필요하다.
- **RoleBinding + ClusterRole 조합**: ClusterRole을 만들어두고 RoleBinding으로 바인딩하면, "정의는 클러스터 전역에서 재사용, 적용은 특정 네임스페이스에만"이라는 흔한 패턴을 구현할 수 있다. (반대로 ClusterRoleBinding + Role은 불가능 — Role은 네임스페이스에 종속적이라 클러스터 전체에 바인딩할 수 없다.)
- **ServiceAccount**: 사람이 아닌 **Pod(애플리케이션)**가 API Server에 인증할 때 사용하는 계정. 모든 네임스페이스에는 `default` ServiceAccount가 자동 생성되며, Pod는 명시하지 않으면 이를 사용한다. Pod 내부에서는 `/var/run/secrets/kubernetes.io/serviceaccount/token`에 토큰이 자동 마운트된다.

```
Pod ──uses──> ServiceAccount ──bound by──> RoleBinding ──grants──> Role
```

---

## 고가용성(HA) 클러스터 구성 개요

시험에서 HA 클러스터를 처음부터 구축하는 문제는 드물지만, 개념은 반드시 알아야 한다.

```
                    [Load Balancer]
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  [Control Plane 1]  [Control Plane 2]  [Control Plane 3]
   API Server           API Server         API Server
   Scheduler            Scheduler          Scheduler
   Controller Mgr       Controller Mgr     Controller Mgr
   etcd                 etcd               etcd
        │                 │                 │
        └────── etcd Raft 클러스터(3 멤버) ──┘
```

- **Stacked etcd**: etcd가 각 컨트롤 플레인 노드에 함께(stacked) 배치되는 기본 구성. 구성이 단순하지만 컨트롤 플레인 노드 장애 시 etcd 멤버도 함께 잃는다.
- **External etcd**: etcd를 컨트롤 플레인과 분리된 별도 노드 그룹에 구성. 더 견고하지만 노드가 더 필요하고 구성이 복잡하다.
- 여러 API Server 앞단에 **Load Balancer**(또는 keepalived+haproxy 조합)를 두어 하나의 VIP로 접근하게 하며, worker 노드는 이 VIP를 API Server 주소로 사용한다.
- Controller Manager, Scheduler는 **Leader Election**을 통해 여러 인스턴스 중 하나만 활성화되도록 동작한다 (동시에 여러 개가 스케줄링 결정을 내리면 충돌이 발생하기 때문).

---

## 용어 요약

| 용어 | 설명 |
|---|---|
| kubeadm | 클러스터 부트스트랩 도구 (init/join/upgrade) |
| Static Pod | kubelet이 manifest 디렉토리를 감시해 직접 관리하는 Pod |
| Mirror Pod | Static Pod를 API Server에서 조회할 수 있게 등록한 읽기 전용 표현 |
| etcd | 클러스터 전체 상태를 저장하는 분산 key-value 저장소 |
| etcdctl | etcd에 직접 접속해 조작하는 CLI (API Server 우회) |
| Snapshot | 특정 시점의 etcd 전체 데이터를 담은 단일 파일 |
| kubeadm upgrade plan | 업그레이드 가능 버전과 변경 사항을 미리 보여줌 (실제 변경 없음) |
| kubeadm upgrade apply | 컨트롤 플레인 컴포넌트를 실제로 업그레이드 |
| kubeadm upgrade node | worker 노드의 kubelet 설정을 업그레이드 |
| drain / cordon / uncordon | 노드에서 Pod 축출 / 스케줄링 차단 / 스케줄링 재개 |
| CA (Certificate Authority) | 클러스터 신뢰의 뿌리가 되는 인증서 |
| kubeconfig | 클러스터 접속 정보(주소/인증)를 담은 설정 파일 |
| Context | (cluster, user, namespace) 조합에 붙인 별칭 |
| Role / ClusterRole | 네임스페이스 범위 / 클러스터 범위 권한 정의 |
| RoleBinding / ClusterRoleBinding | 권한을 Subject에 실제로 부여하는 연결 |
| ServiceAccount | Pod가 API Server에 인증할 때 사용하는 비-사람 계정 |
| Stacked / External etcd | etcd를 컨트롤 플레인에 합칠지 분리할지에 대한 HA 구성 방식 |

---

## 흔한 실수

- **실수 1**: "etcd 백업은 `kubectl` 명령으로도 할 수 있다"
  - 정정: `kubectl`은 API Server를 거치는 도구이고, 스냅샷은 `etcdctl`로 etcd에 **직접** 접속해야만 가능하다. 인증서 경로를 지정하지 않으면 항상 연결이 거부된다.

- **실수 2**: "etcd 복원 후 별도 조치 없이 바로 반영된다"
  - 정정: `snapshot restore`는 새 데이터 디렉토리를 만들 뿐이다. `/etc/kubernetes/manifests/etcd.yaml`의 볼륨 경로를 그 새 디렉토리로 바꿔줘야 kubelet이 새 etcd를 기동한다.

- **실수 3**: "노드 업그레이드 시 worker부터 해도 상관없다"
  - 정정: 컨트롤 플레인이 워커보다 낮은 버전이면 안 되므로, 반드시 **control plane → worker** 순서로 진행해야 한다.

- **실수 4**: "마이너 버전을 한 번에 여러 단계 건너뛰어도 된다"
  - 정정: kubeadm 업그레이드는 **한 번에 마이너 버전 1단계씩만** 지원한다. 1.27→1.30은 1.28, 1.29를 순서대로 거쳐야 한다.

- **실수 5**: "ClusterRoleBinding에 Role을 바인딩할 수 있다"
  - 정정: 불가능하다. ClusterRoleBinding은 ClusterRole만 참조할 수 있다. Role은 RoleBinding에서만 사용 가능하다.

- **실수 6**: "Static Pod는 `kubectl delete pod`로 완전히 지울 수 있다"
  - 정정: kubelet이 manifest 파일을 계속 감시하고 있으므로 즉시 재생성된다. 삭제하려면 manifest 파일 자체를 `/etc/kubernetes/manifests/`에서 옮기거나 지워야 한다.

---

## 빠른 복습

- kubeadm init은 CA/인증서 생성 → kubeconfig 생성 → Static Pod manifest 생성 → kubelet이 이를 읽어 컨트롤 플레인을 기동하는 순서로 동작한다.
- etcd는 클러스터의 유일한 상태 저장소이며, API Server만 직접 접근한다. 스냅샷 백업/복원은 `etcdctl`로 인증서 경로를 정확히 지정해야 한다.
- 업그레이드는 항상 control plane 먼저, worker는 한 노드씩 drain 후 진행하며, 마이너 버전은 한 단계씩만 올릴 수 있다.
- Static Pod는 kubelet이 manifest 디렉토리를 감시해 직접 관리하는 Pod로, 컨트롤 플레인 컴포넌트가 Static Pod인 이유는 API Server 없이도 부팅 가능해야 하기 때문이다.
- RBAC는 Role/ClusterRole(무엇을) + RoleBinding/ClusterRoleBinding(누구에게)의 조합이며, ServiceAccount는 Pod가 API Server에 인증하는 수단이다.
- HA 클러스터는 Load Balancer + 다중 컨트롤 플레인 + etcd Raft 클러스터로 구성되며, Controller Manager/Scheduler는 Leader Election으로 단일 활성 인스턴스를 유지한다.

---

> 다음 단계: [CKA 02 - Cluster Architecture & Installation (실습)](CKA-02-Cluster-Architecture-Installation-실습.md) — etcd 백업/복구와 kubeadm 업그레이드를 직접 손으로 실행한다.
