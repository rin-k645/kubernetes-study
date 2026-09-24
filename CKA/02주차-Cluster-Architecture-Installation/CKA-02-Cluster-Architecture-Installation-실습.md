---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 02 - Cluster Architecture & Installation (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 02 - Cluster Architecture & Installation (개념)](CKA-02-Cluster-Architecture-Installation-개념.md) | 이전: [CKA 01 - 기초 개념 & 환경 세팅 (실습)](../01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-실습.md) | 다음: [CKA 03 - Workloads & Scheduling (실습)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-실습.md)

---

## 실습 환경 전제

- [CKA 01 - 기초 개념 & 환경 세팅 (실습)](../01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-실습.md)에서 구축한 kubeadm 클러스터 (1 control-plane + 2 worker, Ubuntu 기준)를 그대로 사용한다.
- 모든 명령은 별도 표기가 없으면 **control-plane 노드**에서 실행한다.
- 시험에서는 시간이 절대적으로 부족하므로, 여기 나오는 명령은 **눈으로 읽지 말고 반드시 타이핑하며** 따라할 것.

---

## 1. etcd 스냅샷 백업 & 복구 (시험 최빈출)

### 1-1. 인증서 경로 확인 & etcdctl 준비

인증서 경로를 외우려 하지 말고, 매번 static pod manifest에서 확인하는 습관을 들인다.

```bash
cat /etc/kubernetes/manifests/etcd.yaml | grep -- '-cert\|-key\|-ca\|listen-client\|data-dir'
```

```
--advertise-client-urls=https://192.168.56.10:2379
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
--listen-client-urls=https://127.0.0.1:2379,https://192.168.56.10:2379
--data-dir=/var/lib/etcd
```

이 값들이 아래 백업/복구 명령에 그대로 필요하다. `which etcdctl`로 설치를 확인하고, API 버전을 명시적으로 지정한다 (Kubernetes 1.13+ 기본값이지만 습관화할 것).

```bash
export ETCDCTL_API=3
```

### 1-2. 스냅샷 저장 & 상태 확인

```bash
mkdir -p /opt/backup

ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot-$(date +%Y%m%d%H%M).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

ETCDCTL_API=3 etcdctl --write-out=table snapshot status /opt/backup/etcd-snapshot-*.db
```

```
+----------+----------+------------+------------+
|  HASH    | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| 3a1f2c9d |     1284 |        512 |     2.1 MB |
+----------+----------+------------+------------+
```

시험에서는 문제에 지정된 목적지 경로/파일명을 정확히 사용해야 채점된다.

### 1-3. (연습용) 백업 이후 상태 변화 만들기

```bash
kubectl create namespace backup-test
kubectl run test-pod --image=nginx -n backup-test
```

지금 만든 `backup-test`는 위 1-2 스냅샷 시점 **이후**의 상태다. 아래 복구를 마치면 이 네임스페이스가 사라지는 것으로 "스냅샷 시점으로 되돌아갔다"는 것을 검증한다.

### 1-4. 스냅샷 복구 (snapshot restore)

복구는 기존 etcd를 덮어쓰는 게 아니라 **새로운 데이터 디렉토리**에 데이터를 풀어놓는 방식이다.

```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot-20260924.db \
  --data-dir=/var/lib/etcd-from-backup

ls /var/lib/etcd-from-backup/member/
```

`--data-dir`는 기존 `/var/lib/etcd`와 겹치지 않는 새 경로여야 하며, 이 시점까지는 실제 etcd 프로세스에 아무 영향이 없다.

### 1-5. Static Pod manifest 전환 & 복구 확인

`/etc/kubernetes/manifests/etcd.yaml`의 `volumes` 섹션에서 `etcd-data`의 hostPath를 새 디렉토리로 바꾼다.

```yaml
volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup   # 기존 /var/lib/etcd 에서 변경
      type: DirectoryOrCreate
    name: etcd-data
```

저장 즉시 kubelet이 변경을 감지해 etcd Static Pod를 재기동한다.

```bash
watch crictl ps | grep etcd     # etcd 컨테이너 재시작 확인
kubectl get namespaces          # backup-test가 사라져야 정상 (백업 이전 상태로 복원됨)
```

API Server 응답이 잠시 없을 수 있으니 몇십 초 대기 후 재확인하고, 계속 안 되면 `crictl ps`, `journalctl -u kubelet -f`로 직접 로그를 확인한다.

> 시험 팁: save/restore의 옵션 4종(`--endpoints/--cacert/--cert/--key`)은 반드시 `etcd.yaml`에서 그대로 복사해 쓸 것 — 직접 타이핑하다 오탈자로 실패하는 경우가 가장 흔하다.

---

## 2. kubeadm 클러스터 업그레이드

> 시나리오: control-plane 노드를 1.28.x → 1.29.x로 업그레이드한 뒤, worker 노드도 순서대로 업그레이드한다.

### 2-1. [Control Plane] 버전 확인 → kubeadm 업그레이드 → plan 확인

`apt-mark hold`는 실수로 버전이 올라가는 것을 막는 고정이므로, 올리기 전에는 반드시 `unhold`로 풀어야 한다.

```bash
kubectl get nodes -o wide && kubeadm version

sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
```

```
COMPONENT                 CURRENT   TARGET
kube-apiserver             v1.28.4   v1.29.0
kube-controller-manager    v1.28.4   v1.29.0
kube-scheduler             v1.28.4   v1.29.0
kube-proxy                 v1.28.4   v1.29.0
etcd                       3.5.9-0   3.5.10-0
```

`plan`은 미리보기일 뿐 아무것도 변경하지 않는다.

### 2-2. [Control Plane] drain → upgrade apply → kubelet/kubectl 업그레이드 → uncordon

```bash
kubectl drain <control-plane-node-name> --ignore-daemonsets

sudo kubeadm upgrade apply v1.29.0
# [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.0". Enjoy!

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

kubectl uncordon <control-plane-node-name>
kubectl get nodes   # control-plane만 VERSION이 1.29.0으로 올라갔는지 확인
```

- `--ignore-daemonsets`가 없으면 kube-proxy 같은 DaemonSet Pod 때문에 drain이 실패한다.
- `kubeadm upgrade apply`가 컨트롤 플레인 Static Pod manifest들을 새 버전으로 교체하고 필요 시 인증서도 갱신한다.

### 2-3. [Worker 노드] 한 노드씩 drain → 업그레이드 → uncordon

**control-plane 노드에서** worker를 drain한 뒤, **worker에 SSH 접속**해 아래를 실행한다.

```bash
# (control-plane에서)
kubectl drain <worker-1> --ignore-daemonsets --force

# (worker-1에서)
sudo apt-mark unhold kubeadm
sudo apt-get update && sudo apt-get install -y kubeadm=1.29.0-1.1
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node   # worker는 upgrade apply가 아니라 upgrade node!

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.29.0-1.1 kubectl=1.29.0-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# (다시 control-plane에서)
kubectl uncordon <worker-1>
```

**worker-2도 동일하게 반복**한다 (drain → kubeadm/kubelet/kubectl 업그레이드 → uncordon).

### 2-4. 최종 확인

```bash
kubectl get nodes -o wide
```

모든 노드의 `VERSION`이 `v1.29.0`으로 통일되고 `STATUS`가 전부 `Ready`인지 확인한다.

---

## 3. Static Pod 직접 만들어보기

컨트롤 플레인 컴포넌트가 왜 Static Pod로 뜨는지 이해했다면, 직접 매니페스트를 만들어 kubelet이 자동으로 인식하는 것을 확인한다.

### 3-1. kubelet의 manifest 경로 확인

```bash
ps -ef | grep kubelet | grep -o 'pod-manifest-path=[^ ]*'
# 또는 kubelet config 파일에서 확인
cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

기본값은 `/etc/kubernetes/manifests/`다.

### 3-2. Static Pod 매니페스트 작성

```bash
sudo vi /etc/kubernetes/manifests/static-web.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: static-web
  labels:
    role: static-demo
spec:
  containers:
    - name: web
      image: nginx:alpine
      ports:
        - containerPort: 80
```

파일을 저장하는 즉시(별도 `kubectl apply` 없이) kubelet이 감시하다가 Pod를 생성한다.

### 3-3. Mirror Pod 확인

```bash
kubectl get pods -A | grep static-web
```

```
default   static-web-<node-hostname>   1/1   Running   0   10s
```

Pod 이름 뒤에 노드 이름이 자동으로 붙는 것이 Static Pod의 특징이다.

### 3-4. delete로는 안 지워짐 vs manifest 삭제로 지우기

```bash
kubectl delete pod static-web-<node-hostname>
sleep 5
kubectl get pods -A | grep static-web   # 여전히 존재 (kubelet이 즉시 재생성)

sudo rm /etc/kubernetes/manifests/static-web.yaml
sleep 5
kubectl get pods -A | grep static-web   # 이제 사라짐
```

"manifest 파일 = 원본, kubectl로 보이는 Pod = 그 원본의 반영(mirror)"이라는 관계가 이 실습으로 명확해진다.

---

## 4. RBAC 실습 — ServiceAccount, Role, RoleBinding

### 4-1. 네임스페이스와 ServiceAccount 생성

```bash
kubectl create namespace rbac-demo
kubectl create serviceaccount deploy-viewer -n rbac-demo
```

### 4-2. Role 생성 — Pod 조회만 가능하도록 제한

imperative 명령 + dry-run으로 YAML 뼈대를 만든 뒤 필요하면 수정하는 방식을 연습한다.

```bash
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n rbac-demo \
  --dry-run=client -o yaml > pod-reader-role.yaml

cat pod-reader-role.yaml
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: rbac-demo
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

```bash
kubectl apply -f pod-reader-role.yaml
```

### 4-3. RoleBinding으로 ServiceAccount에 권한 부여

```bash
kubectl create rolebinding deploy-viewer-binding \
  --role=pod-reader \
  --serviceaccount=rbac-demo:deploy-viewer \
  -n rbac-demo
```

### 4-4. `kubectl auth can-i --as`로 권한 검증

ServiceAccount를 `--as`로 지정할 때는 반드시 `system:serviceaccount:<namespace>:<name>` 전체 형식을 써야 한다.

```bash
kubectl auth can-i get pods --as=system:serviceaccount:rbac-demo:deploy-viewer -n rbac-demo      # yes
kubectl auth can-i delete pods --as=system:serviceaccount:rbac-demo:deploy-viewer -n rbac-demo   # no (삭제 권한 없음)
kubectl auth can-i get pods --as=system:serviceaccount:rbac-demo:deploy-viewer -n default        # no (다른 네임스페이스는 Role 범위 밖)
```

### 4-5. ClusterRole + RoleBinding 조합 (클러스터 범위 리소스 접근)

Node 목록 조회처럼 클러스터 범위 리소스는 Role로는 불가능하고 ClusterRole이 필요하다.

```bash
kubectl create clusterrole node-reader --verb=get,list --resource=nodes
kubectl create clusterrolebinding deploy-viewer-node-binding \
  --clusterrole=node-reader --serviceaccount=rbac-demo:deploy-viewer

kubectl auth can-i get nodes --as=system:serviceaccount:rbac-demo:deploy-viewer   # yes
```

### 4-6. 정리(cleanup)

```bash
kubectl delete namespace rbac-demo
kubectl delete clusterrole node-reader
kubectl delete clusterrolebinding deploy-viewer-node-binding
```

---

## 5. kubeconfig 다루기 — 여러 클러스터/컨텍스트 시나리오

### 5-1. 현재 컨텍스트 확인

```bash
kubectl config view
kubectl config get-contexts
kubectl config current-context
```

### 5-2. 두 번째 kubeconfig 준비, context 이름 변경, 병합(flatten)

여러 클러스터를 오가는 시나리오를 재현하기 위해 admin.conf를 복사해 별도 이름의 context를 만들고, `KUBECONFIG`에 콜론(`:`)으로 경로를 나열해 병합한다.

```bash
cp /etc/kubernetes/admin.conf /home/vagrant/second.conf
sudo chown vagrant:vagrant /home/vagrant/second.conf

export KUBECONFIG=/home/vagrant/second.conf
kubectl config rename-context kubernetes-admin@kubernetes second-cluster-context
unset KUBECONFIG

export KUBECONFIG=/etc/kubernetes/admin.conf:/home/vagrant/second.conf
kubectl config view --flatten > /home/vagrant/merged.conf
export KUBECONFIG=/home/vagrant/merged.conf

kubectl config get-contexts
```

```
CURRENT   NAME                          CLUSTER      AUTHINFO             NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
          second-cluster-context        kubernetes   kubernetes-admin
```

### 5-3. context 전환, 네임스페이스 기본값 지정, 일회성 실행

```bash
kubectl config use-context second-cluster-context
kubectl config set-context --current --namespace=rbac-demo
kubectl config view --minify | grep namespace   # 현재 활성 context 정보만 확인

# 전환 없이 --context 플래그로 바로 지정 (시험에서 시간 절약)
kubectl --context=kubernetes-admin@kubernetes get nodes
```

### 5-4. 원래 설정으로 복구

```bash
unset KUBECONFIG
kubectl config current-context   # 기본 위치(~/.kube/config)의 context로 복귀했는지 확인
```

---

## 체크리스트

- [ ] `etcdctl snapshot save`로 인증서 경로를 정확히 지정해 스냅샷을 저장할 수 있다
- [ ] `etcdctl snapshot restore`로 새 데이터 디렉토리를 만들고, `etcd.yaml`의 hostPath를 수정해 복구를 완료할 수 있다
- [ ] 스냅샷 시점 이전/이후 리소스 차이를 `kubectl get`으로 검증할 수 있다
- [ ] `apt-mark unhold/hold`로 kubeadm/kubelet/kubectl 버전을 안전하게 관리할 수 있다
- [ ] `kubeadm upgrade plan` → `apply` → kubelet/kubectl 수동 업그레이드 순서를 control-plane에서 완주했다
- [ ] worker 노드는 `kubeadm upgrade node`를 사용한다는 것을 기억하고 있다
- [ ] 각 노드 업그레이드 전 `drain`, 후 `uncordon`을 빠뜨리지 않는다
- [ ] `/etc/kubernetes/manifests/`에 YAML을 직접 추가해 Static Pod를 생성/삭제해봤다
- [ ] `kubectl delete pod`로 Static Pod가 지워지지 않는 이유를 설명할 수 있다
- [ ] ServiceAccount + Role + RoleBinding을 생성하고 `kubectl auth can-i --as`로 권한을 검증했다
- [ ] ClusterRole + RoleBinding(네임스페이스 한정) vs ClusterRole + ClusterRoleBinding(클러스터 전체)의 차이를 실습으로 구분했다
- [ ] `kubectl config` 명령으로 context를 추가/전환/병합하고 `--minify`로 확인할 수 있다
- [ ] 전체 흐름(백업 → 장애 상황 가정 → 복구 → 검증)을 5분 이내에 수행할 수 있다 (시험 목표 시간)

---

> 다음 단계: [CKA 03 - Workloads & Scheduling (실습)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-실습.md) — Deployment, DaemonSet, 스케줄링 제어를 손으로 익힌다.
