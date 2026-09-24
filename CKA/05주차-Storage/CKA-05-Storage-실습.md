---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 05 - Storage (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 05 - Storage (개념)](CKA-05-Storage-개념.md) | 이전: [CKA 04 - Services & Networking (실습)](../04주차-Services-Networking/CKA-04-Services-Networking-실습.md) | 다음: [CKA 06 - Troubleshooting (실습)](../06주차-Troubleshooting/CKA-06-Troubleshooting-실습.md)

---

## 실습 환경 준비

이 실습은 kubeadm으로 구축한 1 control-plane + 2 worker 클러스터([CKA 01 - 기초 개념 & 환경 세팅 (실습)](../01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-실습.md) 참고)를 전제로 한다. 동적 프로비저닝 실습을 위해서는 `local-path-provisioner` 같은 간단한 provisioner가 필요하다.

```bash
# 클러스터 확인
kubectl get nodes -o wide

# 실습용 네임스페이스 생성
kubectl create namespace storage-lab
kubectl config set-context --current --namespace=storage-lab
```

---

## 1. emptyDir / hostPath 기본 실습

### 1-1. emptyDir로 컨테이너 간 파일 공유

두 컨테이너가 emptyDir을 공유하는 Pod를 만들어본다. 하나는 로그를 쓰고, 다른 하나는 그 로그를 읽는다.

```yaml
# emptydir-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do date >> /logs/out.log; sleep 5; done"]
      volumeMounts:
        - name: shared-logs
          mountPath: /logs
    - name: reader
      image: busybox
      command: ["sh", "-c", "tail -f /logs/out.log"]
      volumeMounts:
        - name: shared-logs
          mountPath: /logs
  volumes:
    - name: shared-logs
      emptyDir: {}
```

```bash
kubectl apply -f emptydir-pod.yaml
kubectl wait --for=condition=Ready pod/emptydir-demo --timeout=60s

# reader 컨테이너 로그로 writer가 쓴 내용이 보이는지 확인
kubectl logs emptydir-demo -c reader --tail=5
```

**확인 포인트**: `writer`가 쓴 파일을 `reader`가 그대로 읽는다 — 같은 Pod 내 컨테이너들이 emptyDir을 공유한다는 증거.

컨테이너 하나를 강제로 죽여서 emptyDir이 유지되는지 확인:

```bash
kubectl exec emptydir-demo -c writer -- kill -STOP 1 2>/dev/null || true
# writer 컨테이너를 재시작시켜본다 (crash 유도)
kubectl exec emptydir-demo -c writer -- sh -c "kill 1"
sleep 5
kubectl get pod emptydir-demo -o jsonpath='{.status.containerStatuses[?(@.name=="writer")].restartCount}'
echo
kubectl logs emptydir-demo -c reader --tail=5
# 컨테이너는 재시작됐지만 로그 파일 내용은 그대로 이어짐 (emptyDir은 Pod 단위로 유지)
```

정리:

```bash
kubectl delete pod emptydir-demo
```

### 1-2. hostPath 실습 및 노드 종속성 확인

```yaml
# hostpath-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  nodeName: <워커노드1_이름>   # 특정 노드에 고정 스케줄링
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo hello-from-hostpath > /data/hello.txt; sleep 3600"]
      volumeMounts:
        - name: host-vol
          mountPath: /data
  volumes:
    - name: host-vol
      hostPath:
        path: /tmp/cka-storage-lab
        type: DirectoryOrCreate
```

```bash
# 워커 노드 이름 확인 후 nodeName 값 치환
kubectl get nodes
sed -i.bak 's/<워커노드1_이름>/<실제노드명>/' hostpath-pod.yaml
kubectl apply -f hostpath-pod.yaml
kubectl wait --for=condition=Ready pod/hostpath-demo --timeout=60s

# 파드 내부 파일 확인
kubectl exec hostpath-demo -- cat /data/hello.txt

# 해당 워커 노드에 직접 ssh 하거나 (또는 노드가 로컬 VM이라면 접근) 확인
# ssh <워커노드1> "cat /tmp/cka-storage-lab/hello.txt"
```

파드를 삭제해도 노드의 파일은 남아있는지, 그리고 **다른 노드로 스케줄링하면 데이터가 보이지 않는지** 확인:

```bash
kubectl delete pod hostpath-demo

# nodeName을 다른 워커 노드로 바꿔서 재생성
sed -i.bak 's/<실제노드명>/<다른워커노드명>/' hostpath-pod.yaml
kubectl apply -f hostpath-pod.yaml
kubectl wait --for=condition=Ready pod/hostpath-demo --timeout=60s
kubectl exec hostpath-demo -- cat /data/hello.txt
# 파일이 없거나 새로 빈 상태 -> hostPath는 노드에 종속됨을 확인
```

```bash
kubectl delete pod hostpath-demo
```

---

## 2. 정적 프로비저닝: PV 수동 생성 + PVC 바인딩

### 2-1. PV 생성 (hostPath 기반, 실습 편의상)

실제 운영에서는 NFS/iSCSI/클라우드 디스크를 쓰지만, 로컬 실습에서는 hostPath 기반 PV로 개념을 익힌다.

```bash
# 노드에 디렉터리 미리 생성 (control-plane에서 hostPath PV 쓸 경우 해당 노드에)
sudo mkdir -p /mnt/data/pv-manual
```

```yaml
# pv-manual.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-manual
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/pv-manual
```

```bash
kubectl apply -f pv-manual.yaml
kubectl get pv pv-manual
# STATUS = Available 이어야 정상
```

### 2-2. PVC 생성 및 바인딩 확인

```yaml
# pvc-manual.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-manual
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl apply -f pvc-manual.yaml
kubectl get pvc pvc-manual
kubectl get pv pv-manual
# 둘 다 STATUS = Bound 로 바뀌는지 확인
```

**용량/AccessMode 매칭 확인 실험** — PVC가 PV보다 큰 용량을 요청하면 바인딩되지 않는 것을 확인:

```bash
kubectl delete pvc pvc-manual

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-too-big
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: manual
  resources:
    requests:
      storage: 5Gi
EOF

kubectl get pvc pvc-too-big
# STATUS = Pending (PV 용량 1Gi < 요청 5Gi 이므로 바인딩 실패)
kubectl describe pvc pvc-too-big | tail -10

kubectl delete pvc pvc-too-big
```

원래 PVC를 다시 생성해서 Bound 상태로 되돌려놓는다.

```bash
kubectl apply -f pvc-manual.yaml
kubectl get pvc pvc-manual
```

---

## 3. StorageClass를 이용한 동적 프로비저닝

### 3-1. 사용 가능한 StorageClass / Provisioner 확인

```bash
kubectl get storageclass
kubectl get sc -o wide
```

- 로컬 kubeadm 클러스터에는 기본 StorageClass가 없는 경우가 많다. `local-path-provisioner`(Rancher)를 설치해 실습한다.

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

kubectl get pods -n local-path-storage
kubectl get sc
# local-path 라는 StorageClass가 생성됨
```

### 3-2. StorageClass 직접 정의 (개념 확인용)

```yaml
# sc-fast.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-local
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

```bash
kubectl apply -f sc-fast.yaml
kubectl get sc
```

### 3-3. PVC 생성 → 동적 프로비저닝 관찰

```yaml
# pvc-dynamic.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-dynamic
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-local
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f pvc-dynamic.yaml
kubectl get pvc pvc-dynamic
```

`volumeBindingMode: WaitForFirstConsumer`이므로 PVC를 소비하는 Pod가 없으면 **PVC는 Pending 상태**로 남는다.

```bash
kubectl get pvc pvc-dynamic
# STATUS = Pending, VOLUME 컬럼 비어있음 (아직 PV 생성 안 됨)
```

Pod를 만들어 PVC를 실제로 사용하게 하면 그제서야 PV가 자동 생성된다:

```yaml
# pod-dynamic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamic
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: pvc-dynamic
```

```bash
kubectl apply -f pod-dynamic.yaml
kubectl wait --for=condition=Ready pod/pod-dynamic --timeout=60s

kubectl get pvc pvc-dynamic
# STATUS = Bound 로 바뀌고 VOLUME 컬럼에 자동 생성된 PV 이름이 채워짐

kubectl get pv
# pvc-dynamic에 대응하는 PV가 자동으로 생성되어 있음 (관리자가 만든 적 없음)

kubectl describe pv $(kubectl get pvc pvc-dynamic -o jsonpath='{.spec.volumeName}')
# ReclaimPolicy가 Delete, StorageClass가 fast-local로 표시됨
```

**확인 포인트**: 정적 프로비저닝과 달리 관리자가 PV를 미리 만들지 않았는데도, PVC + StorageClass 조합만으로 PV가 자동 생성되었다.

---

## 4. Pod에 PVC 마운트 후 데이터 유지 검증

### 4-1. 데이터 쓰기

```bash
kubectl exec pod-dynamic -- sh -c 'echo "persist-me-$(date +%s)" > /data/message.txt'
kubectl exec pod-dynamic -- cat /data/message.txt
```

### 4-2. Pod 삭제 후 재생성 → 데이터 유지 확인

```bash
kubectl delete pod pod-dynamic
# PVC는 그대로 유지되는지 확인
kubectl get pvc pvc-dynamic
```

같은 PVC를 참조하는 Pod를 다시 생성한다 (매니페스트 동일):

```bash
kubectl apply -f pod-dynamic.yaml
kubectl wait --for=condition=Ready pod/pod-dynamic --timeout=60s

kubectl exec pod-dynamic -- cat /data/message.txt
# Pod가 재생성됐지만 이전에 쓴 메시지가 그대로 남아있어야 함
```

**확인 포인트**: emptyDir과 달리, PVC 기반 볼륨은 Pod가 완전히 삭제되고 새로 생성돼도 **PVC/PV가 살아있는 한 데이터가 유지**된다.

### 4-3. Deployment로 재확인 (스케줄링 변화 포함)

```yaml
# deploy-dynamic.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dynamic-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dynamic-app
  template:
    metadata:
      labels:
        app: dynamic-app
    spec:
      containers:
        - name: app
          image: busybox
          command: ["sh", "-c", "sleep 3600"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: pvc-dynamic
```

```bash
kubectl delete pod pod-dynamic
kubectl apply -f deploy-dynamic.yaml
kubectl wait --for=condition=Available deployment/dynamic-app --timeout=60s

POD=$(kubectl get pod -l app=dynamic-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- cat /data/message.txt

# 파드 강제 삭제 -> Deployment가 새 파드 생성
kubectl delete pod $POD
sleep 5
NEW_POD=$(kubectl get pod -l app=dynamic-app -o jsonpath='{.items[0].metadata.name}')
kubectl exec $NEW_POD -- cat /data/message.txt
# 새 파드에서도 동일한 메시지가 보이면 성공
```

---

## 5. Reclaim Policy 변경 및 PVC 삭제 시 동작 관찰

### 5-1. Delete 정책 확인 (동적 PV, 기본값)

```bash
PV_NAME=$(kubectl get pvc pvc-dynamic -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV_NAME -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo
# Delete 로 나와야 함 (StorageClass 기본 정책)
```

Deployment와 PVC를 정리하며 PV가 실제로 삭제되는지 확인:

```bash
kubectl delete deployment dynamic-app
kubectl delete pvc pvc-dynamic

sleep 3
kubectl get pv
# 방금 전까지 있던 $PV_NAME 이 목록에서 사라져 있어야 함 (Delete 정책 -> PV 자동 삭제)
```

### 5-2. Retain 정책으로 변경 후 동작 비교

정적으로 만든 `pv-manual`(정책이 이미 Retain)로 실습한다.

```bash
kubectl get pv pv-manual -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo
# Retain

# PVC 삭제
kubectl delete pvc pvc-manual

# PV 상태 확인
kubectl get pv pv-manual
# STATUS = Released (PV 자체는 삭제되지 않고 남아있음, claimRef는 남아있어 재바인딩 불가)
kubectl describe pv pv-manual | grep -A2 "Claim"
```

**Released 상태에서 새 PVC로 재사용하려면?** `claimRef`를 수동으로 비워야 다시 `Available`로 전환된다 (실무에서 자주 나오는 트러블슈팅 시나리오).

```bash
kubectl patch pv pv-manual -p '{"spec":{"claimRef": null}}'
kubectl get pv pv-manual
# STATUS = Available 로 복귀 확인
```

### 5-3. 이미 존재하는 PV의 정책을 즉석에서 변경

동적 프로비저닝으로 만들어진 PV를 실수로 지우지 않도록, 기존 PV의 정책을 Delete → Retain으로 바꾸는 실습(실무에서 매우 흔함):

```bash
# 다시 동적 PVC를 하나 만들어서 실습
kubectl apply -f pvc-dynamic.yaml
kubectl apply -f pod-dynamic.yaml
kubectl wait --for=condition=Ready pod/pod-dynamic --timeout=60s

PV_NAME=$(kubectl get pvc pvc-dynamic -o jsonpath='{.spec.volumeName}')
kubectl get pv $PV_NAME -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo

kubectl patch pv $PV_NAME -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl get pv $PV_NAME -o jsonpath='{.spec.persistentVolumeReclaimPolicy}'
echo
# Retain 으로 바뀐 것을 확인

kubectl delete pod pod-dynamic
kubectl delete pvc pvc-dynamic
kubectl get pv $PV_NAME
# STATUS = Released, PV는 삭제되지 않고 남아있음 -> Delete였다면 사라졌을 것
```

---

## 정리 (Cleanup)

```bash
kubectl delete pv pv-manual $PV_NAME --ignore-not-found
kubectl delete sc fast-local --ignore-not-found
kubectl delete namespace storage-lab
kubectl config set-context --current --namespace=default
```

---

## 체크리스트

- [ ] emptyDir을 이용해 같은 Pod 내 두 컨테이너가 파일을 공유하도록 구성했다
- [ ] 컨테이너가 재시작돼도 emptyDir 데이터가 유지되는 것을 직접 확인했다
- [ ] hostPath 볼륨이 노드에 종속되어, 다른 노드로 재스케줄링하면 데이터가 보이지 않는 것을 확인했다
- [ ] PV를 수동으로 생성하고, capacity/accessMode 조건에 맞는 PVC로 바인딩(Bound)시켰다
- [ ] PVC 요청 용량이 PV보다 크면 바인딩되지 않고 Pending 상태로 남는 것을 확인했다
- [ ] StorageClass를 생성하고 `volumeBindingMode: WaitForFirstConsumer`가 Pod 생성 시점까지 PV 생성을 지연시키는 것을 확인했다
- [ ] PVC만 만들었을 때는 PV가 자동 생성되지 않고, Pod가 실제로 소비할 때 동적 프로비저닝이 일어나는 과정을 관찰했다
- [ ] PVC를 Pod(및 Deployment)에 마운트해 데이터를 쓰고, Pod를 삭제 후 재생성해도 데이터가 유지되는 것을 검증했다
- [ ] Reclaim Policy가 Delete인 PV는 PVC 삭제 시 함께 삭제되는 것을 확인했다
- [ ] Reclaim Policy가 Retain인 PV는 PVC 삭제 후 Released 상태로 남고, `claimRef`를 지워야 재사용 가능한 것을 확인했다
- [ ] 기존 PV의 Reclaim Policy를 `kubectl patch`로 Delete → Retain 즉석 변경할 수 있다

---

> 다음 단계: [CKA 06 - Troubleshooting (개념)](../06주차-Troubleshooting/CKA-06-Troubleshooting-개념.md) — 시험 비중이 가장 높은(30%) 영역으로, Pod/노드/네트워크 장애를 진단하는 법을 배운다.
