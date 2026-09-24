---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 06 - Troubleshooting (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 06 - Troubleshooting (실습)](CKA-06-Troubleshooting-실습.md) | 이전: [CKA 05 - Storage (개념)](../05주차-Storage/CKA-05-Storage-개념.md) | 다음: [CKA 07 - 통합 실습 & 속도 훈련 (개념)](../07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-개념.md)

---

## 왜 이 영역이 가장 중요한가

Troubleshooting은 CKA 시험에서 **단일 도메인 중 최대 비중인 30%**를 차지한다. 다른 도메인(Architecture 25%, Networking 20%, Workloads 15%, Storage 10%)을 다 합쳐도 이 하나의 영역이 시험 합격/불합격을 가를 만큼 크다.

게다가 Troubleshooting은 별도의 지식 영역이 아니라 **다른 4개 도메인 전부를 진단 관점에서 다시 보는 것**이다. 즉 Networking을 제대로 이해하지 못하면 네트워크 장애를 진단할 수 없고, Storage를 모르면 PVC 관련 장애를 못 잡는다. 그래서 이 노트는 "새로운 지식"보다 "이미 배운 것을 실패 시나리오로 재구성"하는 데 집중한다.

### 한 줄 요약

> "Troubleshooting은 정답을 외우는 게 아니라, 증상에서 원인까지 좁혀가는 절차를 몸에 익히는 것이다."

---

## 체계적인 트러블슈팅 접근법

시험장에서 가장 위험한 태도는 "감으로 찍어서 이것저것 고쳐보기"다. 시간 제한(문제당 5~10분) 안에 문제를 풀려면 다음 절차를 기계적으로 따라야 한다.

```
1. 증상 파악
   "무엇이 안 되는가?" (Pod가 안 뜬다 / 접속이 안 된다 / 노드가 이상하다)
        |
        v
2. 로그 / 이벤트 확인
   kubectl get events, kubectl describe, kubectl logs
        |
        v
3. 컴포넌트별 상태 확인
   Node -> kubelet -> 컨테이너 런타임 -> 컨트롤 플레인 -> 네트워크
   (계층을 위/아래로 이동하며 어디서 끊기는지 확인)
        |
        v
4. 원인 격리 (Isolate)
   "이 계층까지는 정상, 이 계층부터 비정상" 경계를 찾는다
        |
        v
5. 조치 (Fix)
   가장 단순한 수정부터 적용 -> 재확인 -> 필요시 반복
```

### 왜 순서가 중요한가

증상만 보고 바로 5번(조치)로 뛰면 엉뚱한 곳을 고치다가 시간을 낭비한다. 예를 들어 Pod가 `Pending` 상태일 때 무작정 Pod를 재생성해봐야 원인(리소스 부족, 스케줄링 제약)이 그대로면 똑같이 실패한다. 반드시 2~4번(확인 → 격리)을 거쳐야 한다.

### 계층적 사고 — "어디서 끊겼는가"

Kubernetes 장애는 대부분 여러 계층 중 하나에서 발생한다. 문제를 만나면 아래 계층을 위에서 아래로(또는 사용자 요청 방향으로) 훑으며 "정상 / 비정상" 경계를 찾는 것이 핵심이다.

```
사용자 요청
   |
[Ingress/Service]  <- Service가 올바른 Pod를 가리키는가?
   |
[kube-proxy]        <- iptables/ipvs 규칙이 정상인가?
   |
[Pod/Container]     <- 컨테이너가 실행/정상 상태인가?
   |
[kubelet]           <- 노드에서 kubelet이 컨테이너를 관리하고 있는가?
   |
[컨테이너 런타임]    <- containerd/CRI-O가 정상인가?
   |
[Node OS/리소스]     <- 디스크, 메모리, CPU, 네트워크 인터페이스는 정상인가?
```

컨트롤 플레인도 마찬가지로 계층이 있다.

```
kubectl 명령
   |
[API Server]        <- 요청을 받고 있는가? (kubectl 응답 여부로 확인)
   |
[etcd]               <- 상태 저장이 되는가?
   |
[Scheduler]          <- Pod가 노드에 할당되는가?
   |
[Controller Manager] <- Deployment/ReplicaSet 등이 원하는 상태로 수렴하는가?
```

---

## 노드 장애 진단

CKA에서 가장 자주 나오는 유형 중 하나가 "특정 노드가 `NotReady`다. 원인을 찾아 복구하라"이다.

### 진단 순서

```
kubectl get nodes                     # 어떤 노드가 NotReady인지 확인
kubectl describe node <노드명>        # Conditions, Events 확인
ssh <노드>                            # 해당 노드에 직접 접속
systemctl status kubelet              # kubelet 프로세스 상태
journalctl -u kubelet -f              # kubelet 로그 실시간 확인
journalctl -u kubelet --since "10 min ago"  # 최근 로그만
```

`kubectl describe node`의 `Conditions` 섹션에서 확인할 항목:

| Condition | True일 때 의미 |
|---|---|
| Ready | 노드가 정상이며 Pod를 받을 수 있음 |
| MemoryPressure | 메모리 부족 상태 |
| DiskPressure | 디스크 공간 부족 상태 |
| PIDPressure | 프로세스 ID 고갈 |
| NetworkUnavailable | 노드 네트워크 설정이 안 됨 |

`Ready`가 아니거나 `Unknown`이면 kubelet이 API Server에게 heartbeat를 보내지 못하고 있다는 뜻이다 (기본적으로 40초 이상 응답이 없으면 `NotReady`로 표시됨).

### NotReady 노드의 흔한 원인

```
NotReady
  |
  +-- kubelet 프로세스 다운
  |     -> systemctl status kubelet 이 inactive/failed
  |     -> systemctl start kubelet 로 복구
  |
  +-- 컨테이너 런타임(containerd 등) 다운
  |     -> kubelet 로그에 "container runtime is down" 류의 메시지
  |     -> systemctl status containerd 로 확인, restart로 복구
  |
  +-- 디스크 공간 부족 (DiskPressure)
  |     -> df -h 로 확인, 불필요한 이미지/로그 정리
  |     -> crictl rmi --prune 등으로 미사용 이미지 제거
  |
  +-- 메모리 부족 (MemoryPressure)
  |     -> free -h, top 으로 확인
  |     -> 불필요한 프로세스 종료, 워크로드 재배치
  |
  +-- 인증서(kubelet client cert) 만료
  |     -> journalctl -u kubelet 에 "certificate has expired" 메시지
  |     -> kubeadm certs check-expiration 으로 확인
  |     -> kubeadm certs renew all 로 갱신 후 kubelet/컨트롤 플레인 재시작
  |
  +-- 네트워크 설정 문제 (CNI 플러그인 이상)
        -> /etc/cni/net.d/ 설정 파일 확인
        -> CNI 플러그인 Pod(예: calico-node) 상태 확인
```

가장 먼저 확인할 것은 **kubelet이 살아있는가**이고, 그다음이 **컨테이너 런타임이 살아있는가**다. 이 둘이 정상인데도 NotReady라면 리소스 압박이나 인증서, 네트워크 쪽을 의심한다.

---

## 애플리케이션(Pod) 장애 상태별 원인

Pod 상태 이상은 시험에서 가장 빈번하게 출제되는 유형이다. 상태 이름만 보고 원인을 바로 떠올릴 수 있어야 한다.

### 상태별 진단 매트릭스

| 상태 | 의미 | 주요 원인 | 확인 명령어 |
|---|---|---|---|
| `Pending` | 스케줄링이 안 됨 | 리소스 부족, taint/toleration 불일치, nodeSelector/affinity 불일치, PVC 바인딩 대기 | `kubectl describe pod` (Events) |
| `ImagePullBackOff` / `ErrImagePull` | 이미지를 못 받아옴 | 이미지 이름/태그 오타, private 레지스트리 인증 누락, 레지스트리 접근 불가 | `kubectl describe pod` (Events) |
| `CrashLoopBackOff` | 컨테이너가 시작 후 반복적으로 죽음 | 앱 내부 오류, 잘못된 커맨드/엔트리포인트, 헬스체크 실패, 의존 서비스 없음 | `kubectl logs`, `kubectl logs --previous` |
| `OOMKilled` | 메모리 초과로 강제 종료 | limits.memory보다 실제 사용량이 큼, 메모리 누수 | `kubectl describe pod` (Last State: Reason) |
| Init 컨테이너 실패 | Init 컨테이너가 완료되지 못함 | Init 컨테이너 스크립트 오류, 의존 리소스(ConfigMap/Secret) 없음, 대상 서비스 미기동 | `kubectl logs <pod> -c <init-container-name>` |

### Pending — 스케줄링 실패

```
kubectl describe pod <pod명>
```

`Events` 섹션에 스케줄러가 왜 실패했는지 그대로 적혀 있다.

```
Events:
  Warning  FailedScheduling  0/3 nodes are available:
           1 Insufficient cpu, 1 node(s) had taint {...},
           1 node(s) didn't match Pod's node affinity/selector.
```

확인 순서:
1. 클러스터 전체 리소스 여유 확인 → `kubectl describe nodes`의 `Allocated resources`
2. taint/toleration 확인 → `kubectl describe node`의 `Taints`, Pod spec의 `tolerations`
3. nodeSelector/affinity 확인 → Pod spec과 노드 label 비교
4. PVC를 쓰는 경우, PVC가 `Bound` 상태인지 확인 (Storage 문제로 이어짐, [CKA 05 - Storage (개념)](../05주차-Storage/CKA-05-Storage-개념.md) 참고)

### ImagePullBackOff / ErrImagePull

컨테이너 런타임이 이미지를 받아오지 못할 때 발생한다.

```
kubectl describe pod <pod명>
```

```
Events:
  Warning  Failed     Failed to pull image "myapp:v9999":
           rpc error: code = NotFound desc = failed to pull
           and unpack image: ... not found
```

흔한 원인:
- 이미지 이름 또는 태그 오타 (`nginx:lastest` 같은 오타)
- private 레지스트리인데 `imagePullSecrets` 미설정
- 레지스트리 서버가 네트워크상 접근 불가

### CrashLoopBackOff

컨테이너가 시작은 되지만 곧바로 종료되고, kubelet이 재시작 간격을 점점 늘려가며 반복 재시도하는 상태다.

```
kubectl logs <pod명>            # 현재(재시작 후) 컨테이너 로그
kubectl logs <pod명> --previous # 직전에 죽은 컨테이너의 로그 (핵심!)
kubectl describe pod <pod명>    # Last State, Exit Code, Reason 확인
```

`kubectl describe pod`에서 확인할 핵심 필드:

```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```

Exit Code로 원인을 추정할 수 있다.

| Exit Code | 의미 |
|---|---|
| 0 | 정상 종료 (그런데도 재시작되면 restartPolicy 확인) |
| 1 | 애플리케이션 일반 오류 |
| 137 | SIGKILL (128+9) — 대부분 OOMKilled |
| 139 | SIGSEGV (128+11) — 세그멘테이션 폴트 |
| 143 | SIGTERM (128+15) — 정상적인 종료 신호 수신 후 종료 |

`--previous` 옵션이 핵심인 이유는, 현재 컨테이너는 방금 재시작되어 로그가 비어있거나 초기화 단계일 수 있기 때문이다. **죽기 직전의 로그**를 봐야 진짜 원인이 보인다.

### OOMKilled

컨테이너가 설정된 `resources.limits.memory`를 초과해서 커널 OOM Killer에 의해 강제 종료된 상태다.

```
kubectl describe pod <pod명>
```

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

확인/조치:
- Pod spec의 `resources.limits.memory` 값 확인
- 실제 앱이 필요로 하는 메모리와 비교 (모니터링 도구 없다면 limits를 임시로 늘려 재현 여부 확인)
- 메모리 누수 의심 시 애플리케이션 로그/코드 확인 (시험 범위 밖이지만 실무에서는 중요)

### Init 컨테이너 실패

Init 컨테이너는 순서대로 실행되며, 하나라도 실패하면 본 컨테이너가 시작되지 않는다. Pod 상태는 `Init:Error`, `Init:CrashLoopBackOff` 등으로 표시된다.

```
kubectl get pod <pod명>                        # STATUS 열에 Init:0/1 등으로 표시
kubectl describe pod <pod명>                   # 어떤 init 컨테이너에서 멈췄는지 확인
kubectl logs <pod명> -c <init-컨테이너-이름>    # 반드시 -c로 컨테이너 이름 지정
```

흔한 원인: init 컨테이너가 참조하는 ConfigMap/Secret이 없음, 의존 서비스(DB 등)가 아직 준비되지 않아 대기하다 timeout, 스크립트 자체의 오류.

---

## 컨트롤 플레인 컴포넌트 장애 진단

kubeadm으로 구축한 클러스터에서는 API Server, Scheduler, Controller Manager, etcd가 모두 **static pod**로 실행된다. static pod는 `kube-system` 네임스페이스의 일반 Pod처럼 보이지만, 실제로는 kubelet이 `/etc/kubernetes/manifests/` 디렉터리의 YAML 파일을 감시하며 직접 기동/재시작한다.

```
/etc/kubernetes/manifests/
  ├── etcd.yaml
  ├── kube-apiserver.yaml
  ├── kube-controller-manager.yaml
  └── kube-scheduler.yaml
        |
        v  (kubelet이 이 디렉터리를 주기적으로 감시)
     kubelet이 static pod로 직접 기동
        |
        v
   kube-system 네임스페이스에 <컴포넌트>-<노드명> 형태로 Pod가 보임
   (예: kube-apiserver-controlplane)
```

### 진단 절차

API Server가 응답하지 않으면 `kubectl` 명령 자체가 먹통이 되므로, `kubectl describe pod`로 진단할 수 없다는 점이 다른 장애와 결정적으로 다르다. 이때는 **노드에 직접 SSH로 접속**해 아래 순서로 확인한다.

```
1. static pod manifest 문법/내용 확인
   cat /etc/kubernetes/manifests/kube-apiserver.yaml
   (YAML 문법 오류, 잘못된 flag, 잘못된 인증서 경로 등)

2. kubelet이 이 manifest를 어떻게 처리했는지 확인
   journalctl -u kubelet -f
   (manifest 파싱 에러, 컨테이너 기동 실패 메시지 확인)

3. 컨테이너 런타임 레벨에서 직접 확인 (kubectl이 안 먹힐 때 핵심 도구)
   crictl ps -a                     # 종료된 컨테이너까지 포함해 목록 확인
   crictl logs <container-id>       # 해당 컨테이너의 로그 확인
   (docker 기반이면 docker ps -a / docker logs <id>)

4. 문제 있는 manifest 수정 후 저장
   -> kubelet이 자동으로 감지해 재기동 (별도 apply 불필요)
```

static pod의 핵심 특징: **manifest 파일을 수정하고 저장하면 kubelet이 자동으로 감지해 해당 Pod를 재생성**한다. `kubectl apply`가 필요 없다. 반대로 말하면, manifest에 오타가 있으면 kubelet이 계속 재시작을 시도하다 실패를 반복하므로 `crictl ps -a`로 방금 죽은 컨테이너를 찾아 로그를 봐야 한다.

### kubeconfig 문제로 인한 API Server 접근 불가

"API Server 자체는 정상인데 `kubectl` 명령이 안 먹는다"는 다른 유형이다. 이 경우는 클라이언트 쪽(kubeconfig) 문제일 가능성이 높다.

```
kubectl cluster-info                       # 연결 자체가 되는지 확인
echo $KUBECONFIG                           # 환경변수 확인
cat ~/.kube/config                         # server 주소, 인증서 경로 확인
kubectl config view                        # 현재 사용 중인 컨텍스트/클러스터 확인
kubectl config current-context
```

흔한 원인: `server:` 주소의 오타나 잘못된 포트, `certificate-authority` 경로 오류, 만료된 클라이언트 인증서, 잘못된 컨텍스트 선택.

---

## 네트워크 장애 진단

"Service를 통해 Pod에 접근이 안 된다"는 문제는 원인 후보가 많아서 **순서대로 좁혀나가는 사고**가 특히 중요하다.

```
사용자/다른 Pod의 요청
        |
        v
[1] Service가 올바른 Pod를 selector로 잡고 있는가?
    kubectl get svc <svc명> -o yaml   # spec.selector 확인
    kubectl get pods --show-labels    # Pod의 label과 selector 일치 여부
        |
        v  (일치한다면 다음 단계)
[2] Service에 Endpoint가 생성되었는가?
    kubectl get endpoints <svc명>
    (비어있다면 selector 불일치 또는 대상 Pod가 Ready 상태가 아님)
        |
        v  (Endpoint가 있다면 다음 단계)
[3] NetworkPolicy가 트래픽을 막고 있지 않은가?
    kubectl get networkpolicy -A
    kubectl describe networkpolicy <이름>
    (ingress/egress 규칙이 의도치 않게 트래픽을 차단하는지 확인)
        |
        v  (NetworkPolicy 문제 없다면 다음 단계)
[4] kube-proxy가 정상 동작하는가?
    kubectl get pods -n kube-system -l k8s-app=kube-proxy
    kubectl logs <kube-proxy-pod> -n kube-system
    iptables-save | grep <svc-cluster-ip>   # 규칙이 생성되었는지 직접 확인
        |
        v  (kube-proxy 정상이라면 다음 단계)
[5] DNS 문제인가? (서비스명으로 접근 안 될 때)
    kubectl exec -it <임시-pod> -- nslookup <svc명>
    kubectl get pods -n kube-system -l k8s-app=kube-dns   # CoreDNS 상태
    kubectl logs <coredns-pod> -n kube-system
```

### 왜 이 순서인가

앞 단계가 원인이면 뒤 단계를 아무리 고쳐도 해결되지 않는다. 예를 들어 selector가 애초에 틀렸다면(1번) Endpoint는 당연히 비어있고(2번), 그 상태에서 kube-proxy나 DNS를 아무리 조사해도 시간 낭비다. **가장 상위(Service 정의)부터 하위(실제 패킷 전달)로** 내려가며 확인해야 낭비 없이 원인을 찾는다.

### 각 원인의 특징적 신호

| 원인 | 특징적 신호 |
|---|---|
| selector 불일치 | `kubectl get endpoints`가 비어있음, Pod label과 Service selector가 다름 |
| 대상 Pod가 Not Ready | Endpoint는 있었는데 사라짐, readinessProbe 실패 확인 필요 |
| NetworkPolicy 차단 | Endpoint는 정상인데 특정 출발지/목적지에서만 연결 실패 |
| kube-proxy 문제 | 모든 Service 접근이 해당 노드에서만 실패, iptables 규칙 누락 |
| DNS 문제 | ClusterIP로는 접속되는데 서비스명(FQDN)으로는 실패 |

Networking 전반의 구조는 [CKA 04 - Services & Networking (개념)](../04주차-Services-Networking/CKA-04-Services-Networking-개념.md)을 참고할 것.

---

## 진단에 사용하는 핵심 명령어 정리

| 명령어 | 용도 |
|---|---|
| `kubectl get events --sort-by=.lastTimestamp` | 클러스터 전체 이벤트를 시간순으로 확인 |
| `kubectl describe pod <pod>` | Pod의 상태, 이벤트, 조건을 한 번에 확인 |
| `kubectl logs <pod> [-c <container>] [--previous]` | 컨테이너 로그 확인 (재시작 전/후) |
| `kubectl get pod -o wide` | Pod가 배치된 노드, IP 확인 |
| `kubectl get endpoints <svc>` | Service가 실제로 라우팅하는 Pod IP 목록 |
| `systemctl status kubelet` / `containerd` | 노드 데몬 상태 확인 |
| `journalctl -u kubelet -f` | kubelet 실시간 로그 |
| `crictl ps -a` / `crictl logs <id>` | 컨테이너 런타임 레벨 진단 (kubectl 불능 시) |
| `kubeadm certs check-expiration` | 인증서 만료 여부 확인 |

---

## 용어 요약

| 용어 | 설명 |
|---|---|
| NotReady | 노드가 kubelet heartbeat를 일정 시간 이상 보내지 못한 상태 |
| CrashLoopBackOff | 컨테이너가 반복적으로 죽어 kubelet이 점증적 대기 후 재시작을 반복하는 상태 |
| ImagePullBackOff | 이미지 다운로드 실패로 인한 대기 상태 |
| OOMKilled | 메모리 limits 초과로 커널이 컨테이너를 강제 종료한 상태 |
| Static Pod | kubelet이 `/etc/kubernetes/manifests/`를 감시하며 직접 관리하는 Pod (컨트롤 플레인 컴포넌트가 이 방식으로 실행됨) |
| crictl | CRI(Container Runtime Interface) 호환 런타임을 직접 다루는 CLI. kubectl이 죽었을 때 최후의 진단 수단 |
| Endpoint | Service의 selector에 매칭되어 실제로 트래픽을 받는 Pod IP:Port 목록 |
| kube-proxy | 각 노드에서 Service ClusterIP를 실제 Pod로 라우팅하는 iptables/ipvs 규칙을 관리하는 컴포넌트 |

---

## 흔한 실수

- **실수 1**: "Pod가 이상하면 무조건 재생성부터 한다"
  - 정정: 원인(리소스 부족, 이미지 오타 등)을 먼저 파악하지 않으면 재생성해도 똑같이 실패한다. 반드시 `describe`/`logs`로 원인을 확인한 뒤 조치한다.

- **실수 2**: "`kubectl logs`만 보고 CrashLoopBackOff 원인을 못 찾겠다며 포기"
  - 정정: 재시작 직후에는 로그가 비어있을 수 있다. `kubectl logs --previous`로 죽기 직전 로그를 반드시 확인해야 한다.

- **실수 3**: "API Server가 죽으면 `kubectl describe pod kube-apiserver-...`로 진단하려 한다"
  - 정정: API Server 자체가 죽으면 kubectl이 아예 응답하지 않는다. 이때는 노드에 직접 SSH로 접속해 `/etc/kubernetes/manifests/`와 `journalctl -u kubelet`, `crictl logs`로 진단해야 한다.

- **실수 4**: "static pod manifest를 고친 뒤 `kubectl apply -f`를 실행한다"
  - 정정: static pod는 kubelet이 파일을 직접 감시하는 방식이라 `kubectl apply`가 불필요하다. 파일을 저장하는 순간 kubelet이 자동으로 재생성한다.

- **실수 5**: "Service 접속이 안 되면 바로 NetworkPolicy부터 의심한다"
  - 정정: NetworkPolicy는 원인 후보 중 하나일 뿐이다. selector/Endpoint부터 순서대로 확인해야 시간 낭비 없이 원인을 좁힐 수 있다.

- **실수 6**: "NotReady 노드를 보면 바로 재부팅한다"
  - 정정: 재부팅은 원인을 파악하지 못했을 때의 최후 수단이다. 먼저 `systemctl status kubelet`, `journalctl -u kubelet`으로 원인을 확인해야 같은 문제가 재발하지 않는다.

---

## 빠른 복습

- Troubleshooting은 "증상 파악 → 로그/이벤트 확인 → 컴포넌트별 상태 확인 → 원인 격리 → 조치"의 5단계 절차를 기계적으로 따르는 것이 핵심이다.
- 노드가 NotReady면 `systemctl status kubelet`과 `journalctl -u kubelet`으로 kubelet/런타임/리소스/인증서 문제를 순서대로 확인한다.
- Pod 장애는 상태 이름(Pending, ImagePullBackOff, CrashLoopBackOff, OOMKilled, Init 실패)마다 원인 후보가 정해져 있다. `kubectl describe`와 `kubectl logs --previous`가 핵심 도구다.
- 컨트롤 플레인 컴포넌트는 static pod로 실행되므로, API Server가 죽으면 `kubectl`이 아니라 노드 직접 접속 + `/etc/kubernetes/manifests/` + `journalctl -u kubelet` + `crictl`로 진단한다.
- 네트워크 장애는 selector → Endpoint → NetworkPolicy → kube-proxy → DNS 순으로 상위 계층부터 좁혀나간다.

---

> 다음 단계: [CKA 06 - Troubleshooting (실습)](CKA-06-Troubleshooting-실습.md) — 직접 장애를 만들고 복구하며 진단 절차를 손에 익힌다.
