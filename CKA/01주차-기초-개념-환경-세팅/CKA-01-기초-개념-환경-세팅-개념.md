---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 01 - 기초 개념 & 환경 세팅 (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 01 - 기초 개념 & 환경 세팅 (실습)](CKA-01-기초-개념-환경-세팅-실습.md) | 다음: [CKA 02 - Cluster Architecture & Installation (개념)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-개념.md)

---

## Kubernetes란 무엇인가?

Kubernetes(약칭 **K8s**)는 **컨테이너화된 애플리케이션을 자동으로 배포, 확장, 관리해주는 오케스트레이션 플랫폼**이다. Google이 내부적으로 사용하던 Borg 시스템의 경험을 바탕으로 만들어 2014년 오픈소스로 공개했으며, 현재는 CNCF(Cloud Native Computing Foundation)가 관리한다.

### 한 줄 요약

> "Kubernetes는 컨테이너를 어디에, 몇 개나, 어떻게 실행할지 대신 판단하고 계속 그 상태를 유지시켜주는 시스템이다."

### 왜 Kubernetes가 필요한가?

#### 문제 상황 — 컨테이너를 손으로 운영한다면

```
서버 A: docker run app:v1  (컨테이너 다운되면? 재시작 필요)
서버 B: docker run app:v1  (트래픽 몰리면? 수동으로 컨테이너 추가)
서버 C: docker run app:v2  (배포마다 SSH 접속해서 일일이 교체)
```

- 컨테이너가 죽으면 사람이 직접 재시작해야 함
- 트래픽 증가에 맞춰 컨테이너 수를 수동으로 조절해야 함
- 여러 서버에 분산 배치하고, 어느 서버에 자원이 남는지 직접 계산해야 함
- 배포/롤백을 서버마다 일일이 수행해야 함 (실수 발생 가능성 높음)
- 서버 한 대가 죽으면 그 위의 컨테이너를 다른 서버로 옮기는 것도 수동

#### 해결책 — Kubernetes에게 "원하는 상태"를 선언한다

```
사용자: "app:v2 컨테이너를 3개 띄우고, 항상 3개를 유지해줘"
            |
            v
      [Kubernetes]
            |
   +--------+--------+--------+
   |        |        |        |
[Node A] [Node B] [Node C]  (자원 여유에 따라 자동 배치)
 Pod      Pod      Pod
 (죽으면 자동으로 다른 Node에 재생성)
```

- 사람은 "몇 개를 어떤 상태로 유지할지"만 선언(Desired State)
- Kubernetes가 실제 상태(Current State)를 계속 감시하며 차이를 자동으로 보정
- 이 방식을 **선언적 관리(Declarative Management)**라고 부르며 Kubernetes 전체 철학의 핵심이다

---

## 핵심 아키텍처: Control Plane과 Worker Node

Kubernetes 클러스터는 크게 두 종류의 노드로 구성된다: **클러스터의 두뇌 역할을 하는 Control Plane**과 **실제로 컨테이너가 실행되는 Worker Node**.

```
┌───────────────────────────────── Control Plane ─────────────────────────────────┐
│                                                                                    │
│   ┌───────────┐        ┌────────┐        ┌───────────┐      ┌──────────────────┐ │
│   │  kubectl  │──REST─>│  API   │<──────>│   etcd    │      │   Scheduler       │ │
│   │  (사용자) │        │ Server │        │ (key-value│      │  (어느 Node에    │ │
│   └───────────┘        │        │        │  저장소,  │      │   배치할지 결정) │ │
│                         │        │<──────>│  클러스터 │<────>│                   │ │
│                         │        │        │  상태 원본)│      └──────────────────┘ │
│                         │        │                                                │
│                         │        │<──────────────────────>┌──────────────────┐   │
│                         │        │                         │ Controller       │   │
│                         └────────┘                         │ Manager          │   │
│                              ^                              │ (원하는 상태 유지)│   │
│                              │                              └──────────────────┘   │
└──────────────────────────────┼───────────────────────────────────────────────────┘
                                │ HTTPS (watch / report)
                ┌───────────────┼───────────────┐
                │                               │
        ┌───────▼────────┐             ┌────────▼───────┐
        │   Worker Node 1 │             │  Worker Node 2 │
        │  ┌───────────┐  │             │  ┌───────────┐ │
        │  │  kubelet  │  │             │  │  kubelet  │ │
        │  └─────┬─────┘  │             │  └─────┬─────┘ │
        │        │ 관리    │             │        │ 관리   │
        │  ┌─────▼─────┐  │             │  ┌─────▼─────┐ │
        │  │ Pod / Pod │  │             │  │ Pod / Pod │ │
        │  └───────────┘  │             │  └───────────┘ │
        │  ┌───────────┐  │             │  ┌───────────┐ │
        │  │kube-proxy │  │             │  │kube-proxy │ │
        │  │(네트워크  │  │             │  │(네트워크  │ │
        │  │ 규칙 관리)│  │             │  │ 규칙 관리)│ │
        │  └───────────┘  │             │  └───────────┘ │
        │  ┌───────────┐  │             │  ┌───────────┐ │
        │  │ Container │  │             │  │ Container │ │
        │  │ Runtime   │  │             │  │ Runtime   │ │
        │  │(containerd)│ │             │  │(containerd)│ │
        │  └───────────┘  │             │  └───────────┘ │
        └─────────────────┘             └─────────────────┘
```

### Control Plane 컴포넌트

#### 1. API Server (kube-apiserver)

클러스터의 **유일한 진입점(Front Door)**. 모든 요청(kubectl, 다른 컴포넌트, 외부 시스템)은 반드시 API Server를 거친다.

```
kubectl apply -f pod.yaml
      |
      v
[API Server] ── 인증(Authentication) ── 인가(Authorization) ── Admission Control
      |
      v
etcd에 원하는 상태 저장
```

- REST API 형태로 노출되며, 모든 통신은 이 서버를 통해서만 이루어짐
- 다른 컴포넌트끼리는 서로 직접 통신하지 않고 항상 API Server를 경유
- 인증/인가/유효성 검증을 모두 담당하는 "관문" 역할

#### 2. etcd

클러스터의 **모든 상태 정보를 저장하는 분산 key-value 저장소**. Kubernetes의 "데이터베이스"라고 생각하면 된다.

```
etcd에 저장되는 것들:
 ├── 어떤 Pod가 존재해야 하는가 (Desired State)
 ├── 각 Node의 상태
 ├── ConfigMap, Secret
 └── 클러스터 전체 설정
```

- API Server만이 etcd에 직접 접근 (다른 컴포넌트는 API Server를 통해 간접 접근)
- etcd가 손상되면 클러스터 전체 상태를 잃으므로 **정기 백업이 필수** (2주차에서 다룸)

#### 3. Scheduler (kube-scheduler)

새로 생성된 Pod를 **어느 Worker Node에 배치할지 결정**하는 컴포넌트.

```
새 Pod 생성 요청 (아직 Node 미배정)
      |
      v
[Scheduler]
  1. Filtering: 자원이 부족하거나 조건이 안 맞는 Node 제외
  2. Scoring: 남은 Node들에 점수를 매겨 최적의 Node 선정
      |
      v
Pod.spec.nodeName = "worker-1"  (API Server를 통해 etcd에 기록)
```

- 실제로 Pod를 실행하지는 않음. "어디에 실행할지" 결정만 하고 기록
- CPU/메모리 요청량, taint/toleration, affinity 등을 고려 (이후 주차에서 상세히 다룸)

#### 4. Controller Manager (kube-controller-manager)

**"원하는 상태"와 "실제 상태"를 끊임없이 비교하며 차이를 좁히는** 다양한 컨트롤러들의 집합.

```
Controller Manager 내부의 대표 컨트롤러:
 ├── Node Controller     — Node 상태 감시, 다운되면 Pod 재스케줄
 ├── Replication Controller — Pod 개수가 부족하면 새로 생성
 ├── Deployment Controller  — 롤링 업데이트 등 배포 전략 관리
 └── Endpoint Controller    — Service와 Pod 연결 관리
```

이 반복 루프를 **Reconciliation Loop(조정 루프)**라고 하며, Kubernetes 동작 원리의 핵심이다.

```
   ┌──────────────────────────────┐
   │  1. 현재 상태 관찰(Observe)   │
   │           |                  │
   │           v                  │
   │  2. 원하는 상태와 비교(Diff)  │
   │           |                  │
   │           v                  │
   │  3. 차이를 좁히는 조치(Act)   │
   │           |                  │
   └───────────┴──────────────────┘
        (계속 반복)
```

### Worker Node 컴포넌트

#### 1. kubelet

각 Worker Node에서 실행되며, **API Server의 지시를 받아 실제로 Pod가 실행되도록 관리**하는 에이전트.

```
[API Server] ──(이 Node에 배정된 Pod 목록)──> [kubelet]
                                                    |
                                                    v
                                          Container Runtime에게
                                          "이 컨테이너들을 실행해라" 지시
                                                    |
                                                    v
                                          Pod 상태를 주기적으로
                                          API Server에 보고
```

- Node 위의 유일한 "관리자". kubelet이 없으면 그 Node는 클러스터의 일원이 아님
- Pod의 상태(Running, Failed 등)를 지속적으로 감시하고 보고

#### 2. kube-proxy

각 Node에서 실행되며, **Service에 대한 네트워크 규칙(iptables 또는 IPVS)을 관리**한다.

```
Service (ClusterIP: 10.96.0.5) 로 들어온 트래픽
      |
      v
[kube-proxy가 설정한 iptables 규칙]
      |
      v
실제 Pod IP (10.244.1.5, 10.244.2.7, ...) 중 하나로 라우팅
```

- Pod는 재생성될 때마다 IP가 바뀌지만, Service는 고정된 진입점을 제공
- kube-proxy가 이 매핑을 각 Node에서 실시간으로 갱신 (4주차에서 상세히 다룸)

#### 3. Container Runtime

실제로 컨테이너를 실행하는 소프트웨어. Kubernetes는 **CRI(Container Runtime Interface)** 규격을 통해 런타임을 추상화한다.

```
kubelet ──CRI(gRPC)──> containerd / CRI-O 등
```

- Docker Engine은 더 이상 Kubernetes의 기본 런타임이 아님 (Kubernetes 1.24부터 dockershim 제거)
- 현재 가장 널리 쓰이는 런타임은 **containerd**이며, CKA 시험 환경도 containerd 기반이다

---

## 핵심 오브젝트: Pod, Node, Cluster, Namespace

### Pod

Kubernetes에서 **배포 가능한 최소 단위**. 하나 이상의 컨테이너를 감싸는 논리적 그룹이다.

```
┌─────────────────── Pod ───────────────────┐
│  ┌─────────────┐        ┌───────────────┐  │
│  │ 메인 컨테이너 │       │  사이드카      │  │
│  │  (app)      │        │ 컨테이너(log)  │  │
│  └─────────────┘        └───────────────┘  │
│                                             │
│  공유: 네트워크(같은 IP, 같은 포트 공간),    │
│        볼륨(같은 스토리지 마운트 가능)       │
└─────────────────────────────────────────────┘
```

- 같은 Pod 안의 컨테이너들은 `localhost`로 서로 통신 가능
- Pod는 "일회용"으로 취급된다. 죽으면 재시작이 아니라 **새 Pod로 재생성**되며 IP도 바뀜
- 컨테이너를 직접 배포하지 않고 반드시 Pod로 감싸서 배포 (직접 실행하는 컨테이너는 없음)

### Node

Pod가 실제로 실행되는 **물리 서버 또는 가상 머신 한 대**.

- Control Plane Node: 클러스터를 관리 (API Server, etcd, Scheduler, Controller Manager)
- Worker Node: 실제 워크로드(Pod)가 실행됨

### Cluster

**하나 이상의 Control Plane과 여러 Worker Node의 집합**. Kubernetes가 관리하는 전체 단위.

```
Cluster
 ├── Control Plane (1개 또는 고가용성 구성 시 여러 개)
 ├── Worker Node 1
 ├── Worker Node 2
 └── Worker Node N
```

### Namespace

하나의 클러스터 안에서 **리소스를 논리적으로 격리하는 가상 구역**. 폴더처럼 리소스를 분류한다.

```
Cluster
 ├── namespace: default      (별도 지정 없으면 여기 생성됨)
 ├── namespace: kube-system  (Kubernetes 자체 시스템 컴포넌트)
 ├── namespace: kube-public
 ├── namespace: dev
 └── namespace: prod
```

- 같은 이름의 리소스라도 Namespace가 다르면 공존 가능
- Node, PersistentVolume처럼 클러스터 전체에 걸친 리소스는 Namespace에 속하지 않음(cluster-scoped)
- **시험에서 Namespace를 빠뜨리는 실수가 가장 흔하다** — 문제에 지정된 `-n <namespace>`를 항상 확인할 것

---

## kubectl의 동작 원리

`kubectl`은 API Server에 REST 요청을 보내는 **클라이언트 CLI**일 뿐, 그 자체가 클러스터의 일부는 아니다.

```
kubectl get pods
      |
      v
~/.kube/config (kubeconfig 파일) 읽기
      |
      v
현재 context 확인 → 어떤 cluster / 어떤 user / 어떤 namespace 사용할지 결정
      |
      v
해당 cluster의 API Server에 HTTPS 요청 (인증서 or 토큰으로 인증)
      |
      v
API Server가 etcd 조회 후 응답
```

### kubeconfig 구조

```yaml
apiVersion: v1
kind: Config
clusters:
  - name: my-cluster
    cluster:
      server: https://192.168.0.10:6443
      certificate-authority-data: ...
users:
  - name: admin
    user:
      client-certificate-data: ...
      client-key-data: ...
contexts:
  - name: my-context
    context:
      cluster: my-cluster
      user: admin
      namespace: dev
current-context: my-context
```

| 요소 | 역할 |
|---|---|
| `clusters` | 접속할 API Server 주소와 인증 기관 정보 |
| `users` | 인증에 사용할 자격 증명(인증서, 토큰 등) |
| `contexts` | cluster + user + (선택) namespace를 묶은 조합 |
| `current-context` | 현재 kubectl이 사용 중인 context |

- 시험에서는 **여러 클러스터를 오가며** 문제를 풀어야 하므로 `kubectl config use-context <이름>`에 익숙해져야 한다
- `kubectl config get-contexts`로 사용 가능한 context 목록 확인 가능

---

## Imperative vs Declarative

Kubernetes 리소스를 다루는 방식은 크게 두 가지로 나뉜다.

### Imperative(명령형) 방식

"무엇을 하라"고 직접 명령한다.

```bash
kubectl run nginx --image=nginx
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80
kubectl delete pod nginx
```

- 빠르고 간단하지만, 클러스터에 "무엇이 실행됐었는지" 기록이 남지 않음
- 복잡한 설정(volume, probe 등)은 명령줄 옵션만으로 표현하기 어려움

### Declarative(선언형) 방식

"원하는 최종 상태"를 YAML로 정의하고 적용한다.

```bash
kubectl apply -f deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  ...
```

- YAML 파일이 곧 문서화이자 버전 관리 대상 (GitOps의 기반)
- `apply`는 기존 상태와 비교해 **변경분만 반영**하므로 재실행해도 안전 (멱등성)
- 실무에서는 선언형이 표준

### 왜 시험에서는 두 방식을 섞어 쓰는가?

CKA 시험은 시간 압박이 매우 크다(문제당 평균 5~10분). YAML을 처음부터 손으로 타이핑하면 문법 오류로 시간을 낭비하기 쉽다. 그래서 실전에서는 **imperative 명령으로 뼈대를 생성하고, `--dry-run=client -o yaml`로 실제 적용 없이 YAML만 뽑아낸 뒤, 필요한 부분만 수정해서 apply하는 하이브리드 방식**을 사용한다.

```bash
# 1. imperative 명령 + dry-run으로 YAML 뼈대 생성 (실제로 생성되지 않음)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# 2. 필요한 부분만 vim으로 수정 (예: resources, volume, probe 추가)
vim pod.yaml

# 3. 선언형으로 실제 적용
kubectl apply -f pod.yaml
```

```
[imperative 명령]  →  --dry-run=client -o yaml  →  [YAML 뼈대]  →  수정  →  [apply]
      빠름                  검증 없이 생성만               정확함           재실행 안전
```

| 구분 | `--dry-run=client` | `--dry-run=server` |
|---|---|---|
| 동작 | 클라이언트에서만 검증, 서버에 요청 전송 안 함 | 서버까지 요청을 보내 검증하되 실제 저장은 안 함 |
| 속도 | 매우 빠름 | 상대적으로 느림 (API Server 왕복) |
| 시험 활용 | YAML 뼈대 생성용으로 압도적으로 많이 사용 | admission webhook 검증이 필요할 때만 사용 |

이 조합(`kubectl run/create ... --dry-run=client -o yaml`)은 CKA 시험 합격의 핵심 기술 중 하나로 꼽힐 만큼 중요하다.

---

## Kubernetes 오브젝트의 기본 구조

모든 Kubernetes 리소스 YAML은 공통적으로 다음 4가지 최상위 필드를 가진다.

```yaml
apiVersion: apps/v1        # 어떤 API 그룹/버전을 사용하는가
kind: Deployment           # 어떤 종류의 오브젝트인가
metadata:                  # 이름, namespace, label 등 식별 정보
  name: web
  namespace: dev
  labels:
    app: web
spec:                      # 원하는 상태(Desired State) 정의
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.25
```

| 필드 | 설명 |
|---|---|
| `apiVersion` | 리소스가 속한 API 그룹과 버전 (예: `v1`, `apps/v1`, `batch/v1`) |
| `kind` | 리소스 타입 (Pod, Deployment, Service, ConfigMap 등) |
| `metadata` | name, namespace, labels, annotations 등 리소스를 식별하는 정보 |
| `spec` | 사용자가 선언하는 "원하는 상태". 리소스 타입마다 구조가 다름 |
| `status` | (읽기 전용) 클러스터가 채워주는 "실제 상태". 사용자가 직접 작성하지 않음 |

- `apiVersion`을 모를 때는 `kubectl api-resources`로 조회
- `spec` 필드 구조를 모를 때는 `kubectl explain <kind>.spec` 으로 즉시 확인 가능 (실습 노트에서 다룸)

---

## 주요 용어 요약

| 용어 | 설명 |
|---|---|
| Cluster | Control Plane과 Worker Node 전체의 집합 |
| Control Plane | 클러스터를 관리하는 두뇌 (API Server, etcd, Scheduler, Controller Manager) |
| Worker Node | 실제 Pod가 실행되는 서버 |
| API Server | 클러스터의 유일한 진입점, 모든 요청의 관문 |
| etcd | 클러스터 상태를 저장하는 분산 key-value 저장소 |
| Scheduler | Pod를 어느 Node에 배치할지 결정 |
| Controller Manager | 원하는 상태와 실제 상태의 차이를 계속 좁히는 컨트롤러 집합 |
| kubelet | Node에서 Pod 실행을 관리하는 에이전트 |
| kube-proxy | Service 네트워크 규칙을 관리 |
| Container Runtime | 실제로 컨테이너를 실행하는 소프트웨어 (containerd 등) |
| Pod | 배포 가능한 최소 단위, 하나 이상의 컨테이너를 감싼 그룹 |
| Namespace | 클러스터 내 리소스를 논리적으로 격리하는 구역 |
| kubeconfig | 클러스터 접속 정보(주소, 인증, context)를 담은 설정 파일 |
| Context | cluster + user + namespace의 조합 |
| Reconciliation Loop | 원하는 상태와 실제 상태를 비교해 계속 조정하는 루프 |
| Imperative | 명령으로 즉시 실행하는 방식 |
| Declarative | YAML로 원하는 상태를 선언하는 방식 |

---

## 흔한 실수

- **실수 1**: "Pod를 직접 재시작하면 문제가 해결된다"
  - 정정: Pod는 재시작이 아니라 **재생성**되며 IP가 바뀐다. Deployment 등 상위 컨트롤러 없이 단독 Pod를 지우면 다시 생성되지 않는다.

- **실수 2**: "kubectl은 클러스터의 일부다"
  - 정정: kubectl은 그냥 클라이언트 CLI다. kubeconfig에 정의된 context를 통해 API Server에 요청을 보낼 뿐이며, 클러스터 컴포넌트가 아니다.

- **실수 3**: "Namespace를 지정하지 않으면 클러스터 전체에서 보인다"
  - 정정: 지정하지 않으면 `default` Namespace가 사용된다. 다른 Namespace의 리소스는 `-n` 옵션 없이는 조회되지 않는다.

- **실수 4**: "`--dry-run=client`는 서버에도 변경을 적용한다"
  - 정정: `client` 모드는 로컬에서만 검증하고 API Server에 요청 자체를 보내지 않는다. 실제로 적용하려면 별도로 `kubectl apply -f`를 실행해야 한다.

- **실수 5**: "Scheduler가 Pod를 직접 실행한다"
  - 정정: Scheduler는 "어느 Node에 배치할지"만 결정한다. 실제 실행은 해당 Node의 kubelet이 Container Runtime을 통해 수행한다.

---

## 빠른 복습

- Kubernetes는 컨테이너의 배포/확장/자동 복구를 원하는 상태 선언만으로 처리하는 오케스트레이션 플랫폼이다.
- Control Plane(API Server, etcd, Scheduler, Controller Manager)은 클러스터를 관리하고, Worker Node(kubelet, kube-proxy, Container Runtime)는 실제 Pod를 실행한다.
- Pod는 배포의 최소 단위이며, 죽으면 재시작이 아니라 재생성된다.
- kubectl은 kubeconfig의 context(cluster+user+namespace)를 통해 API Server와 통신하는 클라이언트일 뿐이다.
- 시험에서는 imperative 명령 + `--dry-run=client -o yaml`로 YAML 뼈대를 빠르게 만든 뒤 수정하는 하이브리드 방식이 핵심 전략이다.
- 모든 오브젝트는 `apiVersion`, `kind`, `metadata`, `spec` 4대 필드로 구성되며, `status`는 클러스터가 채우는 읽기 전용 필드다.

---

> 다음 단계: [CKA 01 - 기초 개념 & 환경 세팅 (실습)](CKA-01-기초-개념-환경-세팅-실습.md) — kubeadm으로 직접 3노드 클러스터를 구축하고 kubectl 기본기를 손에 익힌다.
