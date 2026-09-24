---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 06 - Troubleshooting (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 06 - Troubleshooting (개념)](CKA-06-Troubleshooting-개념.md) | 이전: [CKA 05 - Storage (실습)](../05주차-Storage/CKA-05-Storage-실습.md) | 다음: [CKA 07 - 통합 실습 & 속도 훈련 (실습)](../07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-실습.md)

---

## 실습 목표

이번 실습은 **일부러 장애를 만들고, 진단 절차대로 원인을 찾아 복구하는 것**을 5개 시나리오로 반복한다. CKA 시험은 이미 망가진 환경을 주고 "고쳐라"라고만 요구하므로, 정답을 외우는 것보다 "무엇을 먼저 확인해야 하는가"의 순서를 몸에 익히는 것이 더 중요하다.

각 시나리오는 다음 구조를 따른다.

```
[장애 재현] -> [증상 확인] -> [진단 명령어] -> [원인] -> [복구 명령어]
```

> 실습 환경: kubeadm으로 구축한 1 control-plane + 1(이상) worker 클러스터를 가정한다. worker 노드명은 `node01`로 표기한다. 실제 환경에 맞게 노드명을 바꿔서 진행할 것.

---

## 시나리오 1 — kubelet 다운으로 인한 노드 NotReady

### 장애 재현

worker 노드(`node01`)에 SSH로 접속해 kubelet을 강제로 중지한다.

```bash
ssh node01
sudo systemctl stop kubelet
```

### 증상 확인

control-plane 노드로 돌아와 확인한다. (기본적으로 kubelet이 40초 이상 heartbeat를 보내지 않으면 NotReady로 전환되므로 약간의 시간이 걸린다.)

```bash
kubectl get nodes
```

```
NAME           STATUS     ROLES           AGE   VERSION
controlplane   Ready      control-plane   10d   v1.30.0
node01         NotReady   <none>          10d   v1.30.0
```

### 진단 명령어

```bash
# 1. control-plane에서 노드 상태와 이벤트 확인
kubectl describe node node01
```

`Conditions` 섹션에서 `Ready` 항목이 `Unknown`이고, 마지막 heartbeat 시각이 오래되었는지 확인한다.

```bash
# 2. 해당 노드에 직접 접속해 kubelet 프로세스 상태 확인
ssh node01
sudo systemctl status kubelet
```

```
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled)
     Active: inactive (dead) since ...
```

```bash
# 3. kubelet 로그로 중지 직전 상황 확인 (평소 습관을 들이는 목적)
sudo journalctl -u kubelet --since "5 min ago"
```

### 원인

kubelet 프로세스가 관리자에 의해 중지됨(`inactive (dead)`). kubelet이 없으면 노드는 API Server에 heartbeat를 보낼 수 없고, 컨테이너도 관리되지 않으므로 클러스터는 해당 노드를 NotReady로 표시한다.

### 복구 명령어

```bash
sudo systemctl start kubelet
sudo systemctl status kubelet     # active (running) 확인
sudo systemctl enable kubelet     # 재부팅 후에도 자동 시작되도록 보장 (원래 활성화되어 있어야 정상)
```

control-plane에서 재확인한다.

```bash
kubectl get nodes -w
```

```
NAME           STATUS   ROLES           AGE   VERSION
node01         Ready    <none>          10d   v1.30.0
```

`Ready`로 돌아오면 복구 완료. `-w`로 실시간 관찰하면 `NotReady -> Ready` 전환을 직접 볼 수 있어 실습 효과가 좋다.

---

## 시나리오 2 — 잘못된 이미지 이름으로 ImagePullBackOff 재현

### 장애 재현

존재하지 않는 이미지 태그로 Pod를 배포한다.

```bash
kubectl run broken-image --image=nginx:this-tag-does-not-exist
```

### 증상 확인

```bash
kubectl get pods
```

```
NAME            READY   STATUS             RESTARTS   AGE
broken-image    0/1     ImagePullBackOff   0          30s
```

(처음 몇 초는 `ErrImagePull`로 보이다가 `ImagePullBackOff`로 바뀐다. 둘 다 같은 원인 계열이다.)

### 진단 명령어

```bash
kubectl describe pod broken-image
```

`Events` 섹션 확인:

```
Events:
  Normal   Pulling    ...   Pulling image "nginx:this-tag-does-not-exist"
  Warning  Failed     ...   Failed to pull image "nginx:this-tag-does-not-exist":
                             rpc error: code = NotFound desc = ... manifest unknown
  Warning  Failed     ...   Error: ErrImagePull
  Normal   BackOff    ...   Back-off pulling image "nginx:this-tag-does-not-exist"
  Warning  Failed     ...   Error: ImagePullBackOff
```

### 원인

이미지 태그가 존재하지 않아 레지스트리에서 manifest를 찾지 못함(`manifest unknown`). 실무에서는 오타, 잘못된 버전 태그, 또는 private 레지스트리 인증 누락이 대표적 원인이다.

### 복구 명령어

올바른 이미지로 수정한다. `kubectl set image`로 즉시 교체하거나, YAML을 고쳐 재적용한다.

```bash
kubectl set image pod/broken-image broken-image=nginx:latest
```

> 참고: `kubectl run`으로 만든 단일 Pod는 `set image`로 직접 수정 가능하지만, 실무/시험에서는 대부분 Deployment로 배포되어 있으므로 `kubectl set image deployment/<이름> <컨테이너명>=<올바른-이미지>` 형태로 고치는 경우가 더 많다.

```bash
kubectl get pods -w
```

```
NAME            READY   STATUS    RESTARTS   AGE
broken-image    1/1     Running   0          10s
```

정리:

```bash
kubectl delete pod broken-image
```

---

## 시나리오 3 — 앱 내부 오류로 CrashLoopBackOff 재현

### 장애 재현

즉시 종료되는 커맨드를 가진 Pod를 배포해 반복 재시작을 유발한다.

```bash
kubectl run crash-app --image=busybox --restart=Always -- /bin/sh -c "echo 'fatal error: config file missing' && exit 1"
```

### 증상 확인

```bash
kubectl get pods
```

```
NAME         READY   STATUS             RESTARTS      AGE
crash-app    0/1     CrashLoopBackOff   4 (30s ago)   2m
```

`RESTARTS` 값이 계속 증가하면서 재시작 간격이 점점 길어지는 것을 관찰할 수 있다(exponential backoff).

### 진단 명령어

```bash
# 1. 직전에 죽은 컨테이너의 로그 확인 (핵심!)
kubectl logs crash-app --previous
```

```
fatal error: config file missing
```

```bash
# 2. Last State, Exit Code 확인
kubectl describe pod crash-app
```

```
State:          Waiting
  Reason:       CrashLoopBackOff
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      ...
  Finished:     ...
```

Exit Code 1은 애플리케이션이 정상적으로 실행되다가 의도적/비정상적으로 오류를 반환하며 종료했음을 의미한다(관련 코드 표는 [CKA 06 - Troubleshooting (개념)](CKA-06-Troubleshooting-개념.md) 참고).

### 원인

컨테이너의 커맨드가 에러 메시지를 출력하고 `exit 1`로 종료하도록 작성됨. 실제 상황이라면 애플리케이션 설정 파일 누락, 환경변수 미설정, 의존 서비스(DB 등) 연결 실패가 흔한 원인이다.

### 복구 명령어

원인이 애플리케이션 로직/설정이므로, 컨테이너가 정상 종료하지 않고 계속 실행되도록 커맨드를 수정한다.

```bash
kubectl delete pod crash-app
kubectl run crash-app --image=busybox --restart=Always -- /bin/sh -c "sleep 3600"
```

```bash
kubectl get pods
```

```
NAME         READY   STATUS    RESTARTS   AGE
crash-app    1/1     Running   0          10s
```

정리:

```bash
kubectl delete pod crash-app
```

> 실전 팁: 시험에서는 이미 배포된 Deployment/Pod의 YAML에서 `command`, `args`, `env`, `volumeMounts` 중 어디가 잘못됐는지 `kubectl logs --previous`와 `describe`로 찾아 `kubectl edit` 또는 YAML 수정 후 재적용하는 형태로 나온다.

---

## 시나리오 4 — API Server 장애: static pod manifest 오류로 복구하기

이 시나리오는 컨트롤 플레인 노드에서 진행한다. **실습 전 반드시 백업을 먼저 만든다.**

### 사전 백업

```bash
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak
```

### 장애 재현

API Server manifest에 존재하지 않는 잘못된 플래그를 추가해 기동이 실패하도록 만든다.

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

`spec.containers[0].command` 배열 마지막에 다음 줄을 추가한다.

```yaml
    - --this-flag-does-not-exist=true
```

저장하면 kubelet이 파일 변경을 감지하고 즉시 재기동을 시도한다(`kubectl apply` 불필요).

### 증상 확인

잠시 후 `kubectl`이 응답하지 않기 시작한다.

```bash
kubectl get nodes
```

```
The connection to the server <서버주소>:6443 was refused - did you specify the right host or port?
```

**여기서 kubectl로는 더 이상 진단할 수 없다.** 이 지점이 이 시나리오의 핵심 포인트다.

### 진단 명령어

컨트롤 플레인 노드에 직접 접속(이미 로그인 상태라면 그대로 진행)해 아래 순서로 확인한다.

```bash
# 1. kubelet 로그에서 static pod 기동 실패 확인
sudo journalctl -u kubelet -f --since "2 min ago"
```

```
... kubelet[...]: Error: unknown flag: --this-flag-does-not-exist
... kubelet[...]: E... "Failed to start container" pod="kube-system/kube-apiserver-controlplane"
```

```bash
# 2. 컨테이너 런타임 레벨에서 직접 확인 (kubectl이 죽어있으므로 crictl 사용)
sudo crictl ps -a | grep apiserver
```

```
CONTAINER    IMAGE                  STATE       NAME             ATTEMPT
a1b2c3d4e5   registry.k8s.io/...    Exited      kube-apiserver   7
```

```bash
# 3. 종료된 컨테이너의 로그 확인
sudo crictl logs a1b2c3d4e5
```

```
Error: unknown flag: --this-flag-does-not-exist
```

### 원인

`kube-apiserver.yaml`에 존재하지 않는 플래그를 추가해 API Server 프로세스가 시작 즉시 종료(exit)됨. kubelet은 static pod manifest에 정의된 대로 계속 재시작을 시도하지만 매번 같은 이유로 실패(`Exited`, ATTEMPT 값 계속 증가).

### 복구 명령어

manifest에서 잘못된 플래그를 제거한다.

```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# 방금 추가한 --this-flag-does-not-exist=true 줄 삭제
```

또는 백업本으로 되돌린다.

```bash
sudo cp /tmp/kube-apiserver.yaml.bak /etc/kubernetes/manifests/kube-apiserver.yaml
```

저장 후 kubelet이 자동으로 정상 기동을 시도한다. 확인:

```bash
sudo crictl ps | grep apiserver     # STATE가 Running으로 바뀌는지 확인
kubectl get nodes                   # kubectl이 다시 정상 응답하는지 확인
```

```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   10d   v1.30.0
node01         Ready    <none>          10d   v1.30.0
```

### 참고 — kubeconfig 오류로 인한 접근 불가 (변형 시나리오)

API Server 자체는 멀쩡한데 `kubectl`만 안 되는 경우도 있다. 아래처럼 kubeconfig의 서버 주소를 일부러 깨뜨려본다.

```bash
cp ~/.kube/config /tmp/config.bak
sed -i 's/:6443/:9999/' ~/.kube/config   # 잘못된 포트로 변경
kubectl get nodes
```

```
Unable to connect to the server: dial tcp ...:9999: connect: connection refused
```

진단: `cat ~/.kube/config`로 `server:` 필드 확인 → 포트가 실제 API Server 포트(기본 6443)와 다름을 발견.

복구:

```bash
cp /tmp/config.bak ~/.kube/config
kubectl get nodes   # 정상 복구 확인
```

이 두 가지(① static pod manifest 오류, ② kubeconfig 오류)는 증상이 비슷해 보이지만(둘 다 kubectl이 안 됨) 원인이 다르므로 반드시 구분해서 진단해야 한다. ①은 서버 쪽 문제(다른 사람도 접속 안 됨), ②는 클라이언트 쪽 문제(나만 접속 안 됨)다.

---

## 시나리오 5 — 리소스 requests/limits로 Pending과 OOMKilled 재현

### 5-1. Pending 재현 — 과도한 리소스 요청

#### 장애 재현

클러스터가 절대 제공할 수 없는 수준의 CPU를 요청하는 Pod를 배포한다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-greedy
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100"
        memory: "10Gi"
EOF
```

#### 증상 확인

```bash
kubectl get pods
```

```
NAME         READY   STATUS    RESTARTS   AGE
too-greedy   0/1     Pending   0          20s
```

#### 진단 명령어

```bash
kubectl describe pod too-greedy
```

```
Events:
  Warning  FailedScheduling  0/2 nodes are available:
           2 Insufficient cpu, 2 Insufficient memory.
```

노드별 실제 여유 리소스도 확인해본다.

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

#### 원인

`requests.cpu: 100`(코어 100개)은 일반적인 실습/시험 환경의 노드 스펙을 훨씬 초과한다. 스케줄러는 요청을 만족하는 노드가 없으면 Pod를 어떤 노드에도 배치하지 않고 `Pending` 상태로 둔다.

#### 복구 명령어

현실적인 값으로 낮춰 재배포한다.

```bash
kubectl delete pod too-greedy
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-greedy
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
EOF
```

```bash
kubectl get pods
```

```
NAME         READY   STATUS    RESTARTS   AGE
too-greedy   1/1     Running   0          5s
```

정리:

```bash
kubectl delete pod too-greedy
```

### 5-2. OOMKilled 재현 — 과도하게 낮은 메모리 limit

#### 장애 재현

메모리를 의도적으로 많이 소비하는 컨테이너에 아주 낮은 memory limit을 건다.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: hog
    image: polinux/stress
    resources:
      limits:
        memory: "20Mi"
      requests:
        memory: "10Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF
```

#### 증상 확인

```bash
kubectl get pods
```

```
NAME       READY   STATUS      RESTARTS      AGE
oom-test   0/1     OOMKilled   3 (20s ago)   1m
```

#### 진단 명령어

```bash
kubectl describe pod oom-test
```

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
  Started:      ...
  Finished:     ...
```

Exit Code 137(=128+9, SIGKILL)과 `Reason: OOMKilled`가 함께 나오면 메모리 초과가 확정적이다.

#### 원인

컨테이너가 150MB를 할당하려 시도했지만 `limits.memory`가 20Mi로 설정되어 있어 커널 OOM Killer가 즉시 프로세스를 강제 종료함.

#### 복구 명령어

실제 필요한 메모리보다 여유 있게 limit을 올린다.

```bash
kubectl delete pod oom-test
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-test
spec:
  containers:
  - name: hog
    image: polinux/stress
    resources:
      limits:
        memory: "256Mi"
      requests:
        memory: "128Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
EOF
```

```bash
kubectl get pods
```

```
NAME       READY   STATUS    RESTARTS   AGE
oom-test   1/1     Running   0          5s
```

정리:

```bash
kubectl delete pod oom-test
```

---

## 전체 정리 — 시나리오별 요약표

| 시나리오 | 재현 방법 | 핵심 진단 명령어 | 복구 방법 |
|---|---|---|---|
| 1. 노드 NotReady | `systemctl stop kubelet` | `describe node`, `systemctl status kubelet`, `journalctl -u kubelet` | `systemctl start kubelet` |
| 2. ImagePullBackOff | 존재하지 않는 이미지 태그 | `describe pod` (Events) | 올바른 이미지로 `set image` |
| 3. CrashLoopBackOff | `exit 1`로 종료하는 커맨드 | `logs --previous`, `describe pod` (Exit Code) | 커맨드/설정 수정 후 재배포 |
| 4. API Server 장애 | manifest에 잘못된 플래그 추가 | `journalctl -u kubelet`, `crictl ps -a`, `crictl logs` | manifest 원복 (kubectl apply 불필요) |
| 5-1. Pending | 과도한 CPU/메모리 requests | `describe pod` (FailedScheduling) | requests 값을 현실적으로 낮춤 |
| 5-2. OOMKilled | 낮은 memory limit + 과다 메모리 사용 | `describe pod` (Exit Code 137, OOMKilled) | memory limit 상향 |

---

## 체크리스트

- [ ] worker 노드의 kubelet을 강제로 내리고, `describe node`와 `journalctl -u kubelet`만으로 원인을 특정한 뒤 복구했다
- [ ] 잘못된 이미지 태그로 `ImagePullBackOff`를 재현하고, Events 메시지에서 `manifest unknown` 류의 문구를 직접 확인했다
- [ ] `kubectl logs --previous`와 `kubectl logs`(재시작 후)의 차이를 직접 비교해봤다
- [ ] `kubectl describe pod`의 Exit Code(1, 137, 139, 143)만 보고 원인 계열을 즉시 떠올릴 수 있다
- [ ] API Server가 완전히 죽은 상태에서 `kubectl` 없이 `journalctl -u kubelet` + `crictl`만으로 진단하고 복구했다
- [ ] static pod manifest를 수정하면 `kubectl apply` 없이 kubelet이 자동으로 재기동한다는 것을 직접 관찰했다
- [ ] kubeconfig 오류(클라이언트 문제)와 API Server 장애(서버 문제)를 증상만으로 구분할 수 있다
- [ ] 과도한 requests로 인한 `Pending`을 `describe pod`의 `FailedScheduling` 메시지로 진단하고 값을 조정해 해결했다
- [ ] `stress` 컨테이너로 `OOMKilled`를 재현하고 memory limit을 조정해 복구했다
- [ ] 5개 시나리오 모두 "장애 재현 → 증상 확인 → 진단 명령어 → 원인 → 복구"를 시간 제한(각 5~10분) 안에 끝내는 연습을 했다

---

> 다음 단계: [CKA 07 - 통합 실습 & 속도 훈련 (개념)](../07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-개념.md) — 지금까지 배운 5개 도메인을 통합해 실전과 같은 시간 압박 속에서 연습한다.
