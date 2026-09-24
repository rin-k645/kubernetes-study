---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 03 - Workloads & Scheduling (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 03 - Workloads & Scheduling (개념)](CKA-03-Workloads-Scheduling-개념.md) | 이전: [CKA 02 - Cluster Architecture & Installation (실습)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md) | 다음: [CKA 04 - Services & Networking (실습)](../04주차-Services-Networking/CKA-04-Services-Networking-실습.md)

---

## 실습 환경

[CKA 01 - 기초 개념 & 환경 세팅 (실습)](../01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-실습.md)에서 구축한 kubeadm 클러스터(control-plane 1 + worker 2)를 그대로 사용한다.

```bash
k get nodes    # cka-cp, cka-worker1, cka-worker2
alias k=kubectl
export do="--dry-run=client -o yaml"
```

---

## 1. Deployment 생성 / 스케일 / 롤링 업데이트 / 롤백

### 생성과 스케일

```bash
k create deployment web --image=nginx:1.24 --replicas=3
k get rs                                # web-<hash> ReplicaSet 1개 생성됨
k scale deployment web --replicas=5
k rollout history deployment/web        # REVISION 1 그대로 → scale은 리비전을 늘리지 않음
```

`kubectl scale`은 Pod template을 바꾸지 않으므로 새 리비전이 생기지 않는다는 것을 반드시 확인한다.

### 롤링 업데이트

```bash
k set image deployment/web nginx=nginx:1.25
k annotate deployment web kubernetes.io/change-cause="nginx 1.25로 업그레이드" --overwrite
k rollout status deployment/web
k get rs -l app=web    # 구버전 RS는 0/0, 신버전 RS는 5/5
```

`maxSurge`/`maxUnavailable`을 조정해 무중단 배포를 흉내낸다.

```bash
k patch deployment web -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
```

이후 이미지를 다시 바꾸며 `k get pods -w`로 Ready Pod 수가 5 아래로 떨어지지 않는지 관찰한다.

### 롤백

일부러 존재하지 않는 태그로 배포해 실패를 유발한다.

```bash
k set image deployment/web nginx=nginx:broken-tag
k rollout status deployment/web --timeout=20s   # 타임아웃 발생
k get pods -l app=web                           # ImagePullBackOff 확인
k rollout history deployment/web
k rollout undo deployment/web                   # 직전 리비전으로 롤백
k rollout undo deployment/web --to-revision=1   # 특정 리비전 지정도 가능
k rollout status deployment/web
```

---

## 2. DaemonSet 배포

```yaml
# ~/manifests/log-agent-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      containers:
      - name: log-agent
        image: busybox:1.36
        command: ["sh", "-c", "while true; do echo tick; sleep 30; done"]
        resources:
          requests: { cpu: "50m", memory: "32Mi" }
          limits: { cpu: "100m", memory: "64Mi" }
```

> control-plane 노드는 기본 `NoSchedule` taint가 있으므로, 모든 노드(control-plane 포함)에 배치하려면 toleration이 필수다.

```bash
k apply -f ~/manifests/log-agent-daemonset.yaml
k get daemonset log-agent
k get pods -l app=log-agent -o wide   # 노드 수(3)만큼 Pod가 정확히 1개씩 있는지 확인
```

---

## 3. CronJob 배포

```yaml
# ~/manifests/cleanup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: cleanup
            image: busybox:1.36
            command: ["sh", "-c", "echo cleanup started at $(date); sleep 5"]
```

```bash
k apply -f ~/manifests/cleanup-cronjob.yaml
k get cronjob cleanup
```

스케줄을 기다리지 않고 수동으로 Job을 생성해 즉시 테스트할 수 있다.

```bash
k create job --from=cronjob/cleanup cleanup-manual-test
k get jobs
k logs -l job-name=cleanup-manual-test
```

5분 이상 대기 후 `k get jobs`와 `k get cronjob cleanup -o jsonpath='{.status.lastScheduleTime}'`로 자동 생성 여부를 확인한다.

---

## 4. Taint / Toleration 실습

worker2를 특수 워크로드 전용으로 격리한다.

```bash
k taint nodes cka-worker2 dedicated=special:NoSchedule
k describe node cka-worker2 | grep Taints
```

**toleration 없는 Deployment** — worker2에 배치되지 않아야 한다.

```bash
k create deployment plain-app --image=nginx:1.24 --replicas=3
k get pods -l app=plain-app -o wide    # worker2에는 하나도 없어야 정상
```

**toleration 있는 Deployment** — worker2에도 배치 "가능"해진다 (강제는 아님).

```yaml
# ~/manifests/special-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: special-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: special-app }
  template:
    metadata:
      labels: { app: special-app }
    spec:
      tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "special"
        effect: "NoSchedule"
      containers:
      - name: nginx
        image: nginx:1.24
```

```bash
k apply -f ~/manifests/special-app.yaml
k get pods -l app=special-app -o wide
```

toleration만으로는 worker2에 반드시 배치되지 않는다는 점(다른 노드에 여유가 있으면 그쪽으로 갈 수 있음)을 확인한다. 강제 배치는 5절의 nodeAffinity와 조합해야 한다.

```bash
k delete deployment plain-app special-app
k taint nodes cka-worker2 dedicated=special:NoSchedule-   # 끝에 '-'로 taint 제거
```

---

## 5. nodeAffinity 실습

```bash
k label node cka-worker2 disktype=ssd
```

```yaml
# ~/manifests/ssd-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ssd-app
spec:
  replicas: 3
  selector:
    matchLabels: { app: ssd-app }
  template:
    metadata:
      labels: { app: ssd-app }
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
      containers:
      - name: nginx
        image: nginx:1.24
```

```bash
k apply -f ~/manifests/ssd-app.yaml
k get pods -l app=ssd-app -o wide   # 3개 모두 cka-worker2에 배치됨
```

라벨을 제거해 필수 조건이 깨지면 어떻게 되는지 확인한다.

```bash
k label node cka-worker2 disktype-
k delete pod -l app=ssd-app
k get pods -l app=ssd-app -o wide     # Pending 상태로 남음
k describe pod -l app=ssd-app | grep -A3 Events
# Warning  FailedScheduling  0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

```bash
k label node cka-worker2 disktype=ssd     # 원복
k delete deployment ssd-app
```

---

## 6. Resource Limit 실습 (requests/limits, kubectl top, OOMKilled)

metrics-server가 없다면 먼저 설치한다 (`kubectl top` 사용을 위해 필요).

```bash
k get deployment metrics-server -n kube-system || \
  k apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### QoS 클래스 확인 (Burstable vs Guaranteed)

```bash
# requests != limits → Burstable
k run burstable-demo --image=nginx:1.24 \
  --requests='cpu=100m,memory=64Mi' --limits='cpu=200m,memory=128Mi'
k describe pod burstable-demo | grep "QoS Class"    # Burstable

# requests == limits → Guaranteed
k run guaranteed-demo --image=nginx:1.24 \
  --requests='cpu=100m,memory=64Mi' --limits='cpu=100m,memory=64Mi'
k describe pod guaranteed-demo | grep "QoS Class"    # Guaranteed
```

`kubectl top`과 노드 할당량 확인:

```bash
k top pod burstable-demo
k top node
k describe node cka-worker1 | grep -A10 "Allocated resources"
k delete pod burstable-demo guaranteed-demo
```

### OOMKilled 재현

```yaml
# ~/manifests/oom-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: app
    image: polinux/stress
    resources:
      requests: { memory: "50Mi" }
      limits: { memory: "100Mi" }         # limits보다 큰 메모리를 요구하도록 유도
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

```bash
k apply -f ~/manifests/oom-demo.yaml
k get pod oom-demo -w    # STATUS가 OOMKilled → Running → OOMKilled 반복(CrashLoopBackOff)
k describe pod oom-demo
```

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

`Exit Code: 137`과 `Reason: OOMKilled`가 메모리 limits 초과의 핵심 시그널이다. CPU는 limits 초과 시 Kill이 아니라 throttling만 발생한다는 차이도 함께 기억한다.

```bash
k delete pod oom-demo
```

---

## 7. ConfigMap / Secret 주입 실습 (env vs volume)

```bash
k create configmap app-config --from-literal=APP_ENV=production --from-literal=APP_PORT=8080
k create secret generic app-secret --from-literal=DB_PASSWORD=s3cr3t-pass
k get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d   # 평문 확인 (암호화 아님)
```

### env 방식

```yaml
# ~/manifests/env-injection-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-injection-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "env | grep -E 'APP_|DB_' && sleep 3600"]
    env:
    - name: APP_ENV
      valueFrom: { configMapKeyRef: { name: app-config, key: APP_ENV } }
    - name: DB_PASSWORD
      valueFrom: { secretKeyRef: { name: app-secret, key: DB_PASSWORD } }
    envFrom:
    - configMapRef: { name: app-config }
```

```bash
k apply -f ~/manifests/env-injection-pod.yaml
k logs env-injection-demo    # APP_ENV, APP_PORT, DB_PASSWORD 모두 출력됨
```

### volume 방식

```yaml
# ~/manifests/volume-injection-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-injection-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "cat /etc/config/APP_ENV; echo; cat /etc/secret/DB_PASSWORD; echo; sleep 3600"]
    volumeMounts:
    - { name: config-volume, mountPath: /etc/config }
    - { name: secret-volume, mountPath: /etc/secret, readOnly: true }
  volumes:
  - name: config-volume
    configMap: { name: app-config }
  - name: secret-volume
    secret: { secretName: app-secret }
```

```bash
k apply -f ~/manifests/volume-injection-pod.yaml
k exec volume-injection-demo -- ls /etc/config /etc/secret
```

### 갱신 반영 차이 확인 (핵심 실습)

```bash
k edit configmap app-config    # APP_ENV: production -> staging 으로 변경

k exec env-injection-demo -- env | grep APP_ENV        # 여전히 production (재시작 전까지 미반영)
k exec volume-injection-demo -- cat /etc/config/APP_ENV # 잠시 후 staging으로 자동 갱신됨
```

env 방식은 즉시 반영되지 않고, volume 방식만 kubelet sync 주기(최대 1분 내외)에 따라 자동 갱신된다는 것을 직접 확인한다.

```bash
k delete pod env-injection-demo volume-injection-demo
k delete configmap app-config
k delete secret app-secret
```

---

## 8. 전체 정리

```bash
k delete deployment web --ignore-not-found
k delete daemonset log-agent --ignore-not-found
k delete cronjob cleanup --ignore-not-found
k delete job --all --ignore-not-found
k get all
```

---

## 체크리스트

- [ ] `kubectl create deployment`로 생성하고 `kubectl scale`로 replica를 조절했으며, scale은 새 리비전을 만들지 않음을 확인했다
- [ ] `kubectl set image`로 롤링 업데이트를 수행하고 `kubectl rollout status`로 진행 상황을 관찰했다
- [ ] 잘못된 이미지 배포 후 `kubectl rollout undo`(리비전 지정 포함)로 롤백에 성공했다
- [ ] `maxSurge`/`maxUnavailable`을 조정해 배포 속도/안정성 트레이드오프를 확인했다
- [ ] DaemonSet을 배포해 모든 노드(control-plane 포함, toleration 필요)에 Pod가 1개씩 뜨는 것을 확인했다
- [ ] CronJob을 배포하고 `kubectl create job --from=cronjob/...`으로 수동 실행을 트리거했다
- [ ] 노드에 taint를 걸고 toleration 없는 Pod는 배치되지 않음을, toleration 있는 Pod만 배치 가능함을 확인했다
- [ ] nodeAffinity(`requiredDuringScheduling...`)로 특정 라벨 노드에만 Pod를 강제 배치했다
- [ ] 라벨 제거 시 필수 nodeAffinity 조건이 깨져 Pod가 Pending 상태로 남는 것을 확인했다
- [ ] requests/limits 설정에 따라 QoS Class(Guaranteed/Burstable)가 달라지는 것을 `kubectl describe pod`로 확인했다
- [ ] 메모리 limits를 초과시켜 OOMKilled(Exit Code 137) 상황을 직접 재현했다
- [ ] `kubectl top pod`, `kubectl top node`, `kubectl describe node`의 Allocated resources를 읽을 수 있다
- [ ] ConfigMap/Secret을 env와 volume 두 가지 방식으로 각각 주입했다
- [ ] ConfigMap 변경 시 env는 반영 안 되고 volume만 자동 갱신됨을 직접 확인했다

---

> 다음 단계: [CKA 04 - Services & Networking (개념)](../04주차-Services-Networking/CKA-04-Services-Networking-개념.md) — Service, Ingress, NetworkPolicy로 워크로드를 네트워크에 노출하고 제어하는 방법을 학습한다.
