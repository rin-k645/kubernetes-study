---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 03 - Workloads & Scheduling (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 03 - Workloads & Scheduling (실습)](CKA-03-Workloads-Scheduling-실습.md) | 이전: [CKA 02 - Cluster Architecture & Installation (개념)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-개념.md) | 다음: [CKA 04 - Services & Networking (개념)](../04주차-Services-Networking/CKA-04-Services-Networking-개념.md)

---

## 이 주차의 목표

Workloads & Scheduling은 CKA 시험 비중 **15%**를 차지하는 도메인이다. "무엇을 어떻게 실행할 것인가(워크로드)"와 "그것을 어느 노드에 배치할 것인가(스케줄링)"라는 두 축으로 구성된다.

### 한 줄 요약

> "Deployment는 파드를 선언적으로 관리하는 상위 컨트롤러이고, 스케줄링은 filtering과 scoring 두 단계를 거쳐 파드가 실행될 노드를 결정하는 과정이다."

---

## 1. 워크로드 리소스 계층

### Pod → ReplicaSet → Deployment

```
Deployment
   │  (버전 이력 관리, 롤링 업데이트/롤백)
   ▼
ReplicaSet (v1)        ReplicaSet (v2)
   │  (replica 수 유지)     │
   ▼                        ▼
Pod  Pod  Pod           Pod  Pod  Pod
```

- **Pod**: 하나 이상의 컨테이너를 묶은 Kubernetes 최소 배포 단위
- **ReplicaSet**: 지정된 수의 Pod 복제본이 항상 떠 있도록 유지. `selector`로 관리 대상 Pod를 식별
- **Deployment**: ReplicaSet을 관리하는 상위 리소스. 새 버전을 배포할 때마다 **새 ReplicaSet을 생성**하고, 점진적으로 트래픽(replica 수)을 옮긴다

> ReplicaSet을 직접 만들 일은 실무에서 거의 없다. 항상 Deployment를 통해 간접적으로 다룬다.

### 롤링 업데이트(Rolling Update) 원리

```
초기 상태: RS-v1 (3/3 Running)

이미지 변경 (kubectl set image ...)
        │
        ▼
Step 1: RS-v2 생성 → 1개 Pod 생성 (RS-v1: 2, RS-v2: 1)
Step 2: RS-v1에서 1개 축소, RS-v2 1개 추가 (RS-v1: 1, RS-v2: 2)
Step 3: RS-v1 0개, RS-v2 3개 (완료)
```

- `maxSurge`: 목표 replica 수보다 **초과 생성 가능한 Pod 수** (기본 25%)
- `maxUnavailable`: 업데이트 중 **동시에 사용 불가능해질 수 있는 Pod 수** (기본 25%)
- 이 두 값을 조정해 "빠르지만 위험한 배포" ↔ "느리지만 안전한 배포"를 튜닝한다

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # 무중단 배포를 원할 때
```

### 롤백(Rollback) 원리

Deployment는 `revisionHistoryLimit`(기본 10)개까지 과거 ReplicaSet을 보존한다. 롤백은 **이전 ReplicaSet을 다시 스케일업**하는 것과 같다.

```
kubectl rollout history deployment/app        → 리비전 목록 확인
kubectl rollout undo deployment/app           → 직전 리비전으로 롤백
kubectl rollout undo deployment/app --to-revision=2   → 특정 리비전으로 롤백
```

> 주의: `kubectl edit`로 metadata만 바꾸거나 `kubectl scale`만 실행하면 새 리비전이 생기지 않는다. **Pod template(`spec.template`)이 바뀌어야만** 새 리비전이 기록된다.

---

## 2. DaemonSet, Job, CronJob

세 리소스 모두 "일반적인 상시 서비스"가 아닌 **특수한 실행 패턴**을 위해 존재한다.

| 리소스 | 용도 | 실행 대상 | 종료 여부 |
|---|---|---|---|
| Deployment | 상시 서비스 (웹 서버 등) | replica 수만큼 아무 노드에나 | 계속 실행 |
| DaemonSet | 노드마다 정확히 1개씩 필요한 에이전트 | **모든(또는 조건에 맞는) 노드에 1개씩** | 계속 실행 |
| Job | 1회성 배치 작업 | 조건에 맞는 노드 중 하나 | **완료 후 종료** |
| CronJob | 주기적으로 반복되는 배치 작업 | 스케줄마다 Job 생성 | 매번 완료 후 종료 |

### DaemonSet

```
Node 1          Node 2          Node 3
┌─────────┐     ┌─────────┐     ┌─────────┐
│ log-agent│     │ log-agent│     │ log-agent│
└─────────┘     └─────────┘     └─────────┘
```

- 대표 사례: 로그 수집기(Fluentd), 모니터링 에이전트(node-exporter), CNI 플러그인, kube-proxy
- `nodeSelector`나 `affinity`로 특정 노드군만 대상으로 제한 가능
- **스케줄러를 거치지 않고** DaemonSet 컨트롤러가 직접 노드마다 Pod를 배치한다 (단, K8s 1.12+부터는 스케줄러가 관여하되 nodeAffinity로 강제 배치)
- 새 노드가 클러스터에 조인하면 자동으로 Pod가 추가됨

### Job

```yaml
spec:
  completions: 5     # 총 5번 성공적으로 완료돼야 함
  parallelism: 2     # 동시에 최대 2개 Pod 실행
  backoffLimit: 3    # 실패 시 재시도 횟수
```

- `completions`와 `parallelism`으로 배치 처리량 조절
- Pod가 실패하면 `backoffLimit`까지 재시도
- 완료된 Job의 Pod는 기본적으로 삭제되지 않음 (`ttlSecondsAfterFinished`로 자동 정리 가능)

### CronJob

```
schedule: "*/5 * * * *"
        │
        ▼
  5분마다 새로운 Job 생성 → 각 Job이 completions/parallelism 규칙에 따라 Pod 생성
```

- `schedule`: 표준 cron 표현식 (분 시 일 월 요일)
- `concurrencyPolicy`: `Allow`(기본, 중복 실행 허용) / `Forbid`(이전 Job 실행 중이면 스킵) / `Replace`(이전 Job 종료 후 새로 실행)
- `startingDeadlineSeconds`: 스케줄된 시각을 이 초만큼 넘기면 실행 포기
- `successfulJobsHistoryLimit` / `failedJobsHistoryLimit`: 보관할 과거 Job 기록 수 (기본 3/1)

---

## 3. 스케줄링 개념

### kube-scheduler의 동작 원리: Filtering → Scoring

```
새로운 Pod (Pending 상태)
        │
        ▼
┌───────────────────┐
│  1. Filtering      │  ← "이 Pod를 실행할 수 없는 노드"를 제거
│  (예전 이름: Predicate) │
└───────────────────┘
        │  살아남은 노드들 (feasible nodes)
        ▼
┌───────────────────┐
│  2. Scoring        │  ← 남은 노드마다 점수(0~100)를 매김
│  (예전 이름: Priority) │
└───────────────────┘
        │
        ▼
   최고 점수 노드에 바인딩 (Bind)
```

- **Filtering**: 리소스 부족, taint 미허용, nodeSelector 불일치, 포트 충돌 등으로 애초에 실행 불가능한 노드를 제외
- **Scoring**: 남은 노드 중 리소스 여유, affinity 선호도, 이미지 존재 여부 등을 기준으로 가장 적합한 노드를 선택
- 최종적으로 선택된 노드 이름을 Pod의 `spec.nodeName`에 기록 (바인딩)

### nodeSelector — 가장 단순한 배치 제어

```yaml
spec:
  nodeSelector:
    disktype: ssd
```

- 노드에 `disktype=ssd` 라벨이 있어야만 스케줄링 가능
- 단순 equality 매칭만 지원. 복잡한 조건(OR, NOT, 우선순위)은 불가 → 그럴 땐 affinity 사용

### nodeAffinity / podAffinity / podAntiAffinity

```
nodeAffinity        → Pod를 "어떤 노드 특성"에 배치할지
podAffinity         → Pod를 "다른 Pod와 같은 위치"에 배치할지 (묶고 싶을 때)
podAntiAffinity     → Pod를 "다른 Pod와 다른 위치"에 배치할지 (분산시키고 싶을 때)
```

각각 두 가지 강도를 지원한다.

| 강도 | 의미 | 조건 불만족 시 |
|---|---|---|
| `requiredDuringSchedulingIgnoredDuringExecution` | 필수 조건 | 스케줄링 자체가 안 됨 (Pending) |
| `preferredDuringSchedulingIgnoredDuringExecution` | 선호 조건 | 안 지켜져도 스케줄링은 됨 (가중치만 반영) |

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname   # 같은 노드에 web Pod가 몰리지 않도록
```

- `topologyKey`는 "같은 위치"의 기준을 정의한다 (`kubernetes.io/hostname` = 노드 단위, `topology.kubernetes.io/zone` = 가용영역 단위)
- podAntiAffinity는 고가용성 배치(HA)에 필수: 같은 Deployment의 Pod들을 서로 다른 노드/존에 분산

### Taints & Tolerations — nodeAffinity와의 방향 차이

```
nodeAffinity   : Pod → "나는 이런 노드를 원해"   (Pod가 노드를 선택)
Taint/Toleration: Node → "나는 이런 Pod만 받아줄게" (노드가 Pod를 거부/허용)
```

- **Taint**는 노드에 설정하며, 해당 taint를 "견딜 수 있는(tolerate)" toleration이 없는 Pod는 스케줄링되지 않는다
- **Toleration**은 Pod에 설정하며, 특정 taint를 허용한다는 의미. 단, toleration이 있다고 **그 노드에 반드시 배치되는 것은 아니다** (배치를 유도하려면 nodeAffinity를 함께 사용해야 함)

```
kubectl taint nodes node1 key=value:NoSchedule
```

| Effect | 의미 |
|---|---|
| `NoSchedule` | 새 Pod는 스케줄링 불가 (기존 실행 중인 Pod는 유지) |
| `PreferNoSchedule` | 가능하면 피하되, 다른 노드가 없으면 스케줄링 허용 |
| `NoExecute` | 새 Pod 스케줄링 불가 + **기존 실행 중인 Pod도 축출(evict)** |

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

- `operator: Exists`를 쓰면 value 값과 무관하게 key만 일치하면 toleration 성립
- `NoExecute` taint에는 `tolerationSeconds`를 지정해 "이 시간 동안만 버티다가 축출됨"으로 설정 가능 (노드 장애 시 Pod 재배치 지연 시간 제어에 사용)

### Taint/Toleration + nodeAffinity 조합 패턴

```
전용 노드(GPU 노드)를 특정 워크로드 전용으로 만들고 싶을 때:

1. GPU 노드에 taint 설정        → 일반 Pod가 실수로 배치되는 것을 막음
2. GPU가 필요한 Pod에 toleration → 해당 taint를 견디게 함
3. GPU가 필요한 Pod에 nodeAffinity → 실제로 GPU 노드에 배치되도록 유도

(2)만 있으면 "GPU 노드에도 배치될 수 있음"이지 "반드시 배치됨"은 아니다.
```

### Manual Scheduling — 스케줄러 없이 직접 배치

```yaml
spec:
  nodeName: node01   # 스케줄러를 거치지 않고 이 노드에 강제 배치
```

- `spec.nodeName`을 지정하면 kube-scheduler의 filtering/scoring 과정을 완전히 건너뛴다
- 이미 생성된 Pod의 `nodeName`은 수정 불가 (Pod를 삭제 후 재생성해야 함)
- 시험에서 "스케줄러가 다운된 상황을 가정하고 Pod를 특정 노드에 배치하라"는 문제 유형으로 출제됨

---

## 4. Resource Requests/Limits와 QoS 클래스

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "64Mi"
  limits:
    cpu: "500m"
    memory: "128Mi"
```

- **requests**: 스케줄링 시 "이만큼은 보장받아야 한다"는 값. kube-scheduler는 노드의 남은 할당 가능 용량과 requests를 비교해 filtering
- **limits**: 컨테이너가 사용할 수 있는 **최대치**. CPU는 초과 시 throttling, 메모리는 초과 시 **OOMKilled**

### QoS(Quality of Service) 클래스

| 클래스 | 조건 | 우선순위(Eviction 시) |
|---|---|---|
| **Guaranteed** | 모든 컨테이너의 requests == limits (CPU, memory 둘 다) | 가장 나중에 축출 |
| **Burstable** | requests와 limits가 설정되어 있지만 값이 다름, 또는 일부만 설정 | 중간 |
| **BestEffort** | requests/limits를 아예 설정하지 않음 | 가장 먼저 축출 |

```
노드 메모리 부족 상황
        │
        ▼
BestEffort Pod부터 축출 → Burstable → Guaranteed 순서
```

> QoS 클래스는 직접 지정하는 필드가 아니라 requests/limits 설정에 따라 **자동으로 결정**된다. `kubectl describe pod`의 `QoS Class` 필드로 확인 가능.

---

## 5. ConfigMap / Secret을 이용한 설정 주입

애플리케이션 설정과 이미지를 분리하는 것이 Kubernetes의 기본 철학이다.

| | ConfigMap | Secret |
|---|---|---|
| 용도 | 일반 설정값 (URL, 포트, 환경명 등) | 민감 정보 (비밀번호, 토큰, 인증서) |
| 저장 방식 | 평문 | base64 인코딩 (암호화 아님! etcd 암호화를 별도 설정해야 진짜 보호됨) |
| 생성 | `kubectl create configmap` | `kubectl create secret generic` |

### 주입 방식 1: 환경 변수(env)

```yaml
env:
- name: DB_HOST
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: db_host
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: app-secret
      key: password
```

또는 통째로 주입:

```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secret
```

- 장점: 애플리케이션 코드 수정 없이 흔히 쓰는 환경변수 패턴 그대로 사용 가능
- 단점: **ConfigMap/Secret이 바뀌어도 이미 실행 중인 Pod의 환경변수는 갱신되지 않음** (Pod 재시작 필요)

### 주입 방식 2: 볼륨 마운트(volume)

```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
containers:
- volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

- ConfigMap/Secret의 각 key가 마운트 경로 아래 **파일**로 생성됨 (key 이름 = 파일 이름, value = 파일 내용)
- 장점: ConfigMap이 업데이트되면 (kubelet의 sync 주기에 따라) **마운트된 파일 내용이 자동 갱신됨** (env 방식과 가장 큰 차이점)
- 애플리케이션이 파일 변경을 감지(watch)하도록 구현되어 있어야 실질적으로 반영됨

---

## 용어 요약

| 용어 | 설명 |
|---|---|
| ReplicaSet | 지정된 수의 Pod 복제본을 유지하는 컨트롤러 |
| Deployment | ReplicaSet을 관리하며 롤링 업데이트/롤백을 지원하는 상위 리소스 |
| DaemonSet | 조건에 맞는 모든 노드에 Pod를 1개씩 배치 |
| Job | 완료를 목표로 하는 1회성 배치 작업 |
| CronJob | 스케줄에 따라 주기적으로 Job을 생성 |
| Filtering | 스케줄링 1단계, 실행 불가능한 노드 제거 |
| Scoring | 스케줄링 2단계, 남은 노드에 점수를 매겨 최적 노드 선택 |
| nodeSelector | 라벨 기반 단순 노드 지정 |
| nodeAffinity | Pod가 원하는 노드 특성을 조건부(필수/선호)로 지정 |
| podAffinity/AntiAffinity | Pod 간 같은 위치/다른 위치 배치 조건 |
| Taint | 노드가 특정 Pod를 거부하는 설정 |
| Toleration | Pod가 특정 taint를 견딜 수 있다는 설정 |
| nodeName | 스케줄러를 거치지 않고 강제로 지정하는 노드 |
| requests/limits | 컨테이너의 최소 보장/최대 허용 리소스 |
| QoS Class | requests/limits 설정에 따라 자동 결정되는 우선순위 등급 |

---

## 흔한 실수

- **실수 1**: "toleration만 설정하면 그 노드에 배치된다"
  - 정정: toleration은 taint를 "견딜 수 있음"을 의미할 뿐, 배치를 강제하지 않는다. 특정 노드에 반드시 배치하려면 nodeAffinity(또는 nodeSelector)를 함께 써야 한다.

- **실수 2**: "Deployment의 replica 수를 바꾸면 새 리비전이 생긴다"
  - 정정: `kubectl scale`은 리비전을 늘리지 않는다. `spec.template`(Pod 스펙)이 바뀔 때만 새 ReplicaSet(리비전)이 생성된다.

- **실수 3**: "limits만 설정하면 QoS는 Guaranteed다"
  - 정정: Guaranteed가 되려면 **requests와 limits 값이 CPU/메모리 모두 동일**해야 한다. limits만 있고 requests가 없으면 Kubernetes가 requests를 limits와 같은 값으로 자동 설정해 Guaranteed가 되는 경우도 있으니 실제로는 `kubectl describe pod`로 QoS Class를 확인하는 습관이 중요하다.

- **실수 4**: "ConfigMap을 수정하면 Pod의 환경변수도 즉시 바뀐다"
  - 정정: env로 주입된 값은 Pod 재시작 전까지 갱신되지 않는다. 볼륨 마운트 방식만 (지연은 있지만) 자동 갱신된다.

- **실수 5**: "DaemonSet은 스케줄러를 완전히 무시한다"
  - 정정: 현재 버전의 Kubernetes(1.12+)는 DaemonSet Pod도 kube-scheduler를 거치되, 내부적으로 nodeAffinity를 자동 주입해 대상 노드에 강제 배치한다.

---

## 빠른 복습

- Deployment → ReplicaSet → Pod 계층에서, 새 버전 배포 시마다 새 ReplicaSet이 생기고 `maxSurge`/`maxUnavailable`로 전환 속도를 조절한다.
- DaemonSet(노드당 1개), Job(1회성 완료), CronJob(주기적 Job 생성)은 각각 다른 실행 패턴을 위한 워크로드다.
- kube-scheduler는 Filtering(실행 불가능한 노드 제거) → Scoring(최적 노드 선정) 2단계로 동작한다.
- nodeAffinity는 Pod가 노드를 선택, Taint/Toleration은 노드가 Pod를 거부/허용 — 방향이 반대다. 강제 배치는 둘을 조합해야 한다.
- requests는 스케줄링 보장치, limits는 사용 상한치이며, 둘의 관계에 따라 QoS 클래스(Guaranteed/Burstable/BestEffort)가 자동 결정된다.
- ConfigMap/Secret은 env(즉시 반영 안 됨) 또는 volume(자동 갱신) 방식으로 주입할 수 있다.

---

> 다음 단계: [CKA 03 - Workloads & Scheduling (실습)](CKA-03-Workloads-Scheduling-실습.md) — 직접 클러스터에서 Deployment 롤링 업데이트, taint/toleration, 리소스 제한을 실습한다.
