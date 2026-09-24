---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 01 - 기초 개념 & 환경 세팅 (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 01 - 기초 개념 & 환경 세팅 (개념)](CKA-01-기초-개념-환경-세팅-개념.md) | 다음: [CKA 02 - Cluster Architecture & Installation (실습)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md)

---

## 실습 환경 옵션 비교

CKA를 준비할 때 클러스터를 어떻게 띄울지부터 결정해야 한다. 대표적으로 세 가지 방법이 있다.

| 방법 | 구성 | 장점 | 단점 | CKA 적합도 |
|---|---|---|---|---|
| **kubeadm (VM 3대)** | Control Plane 1 + Worker 2, 각각 별도 VM | 실제 시험 환경과 100% 동일한 방식. 노드 추가/장애/부팅 과정을 모두 경험 | 초기 구축 시간이 걸림 (VM 준비, 네트워크 설정) | ★★★★★ |
| **Kind (Kubernetes in Docker)** | Docker 컨테이너로 노드를 흉내냄 | 설치가 매우 빠름 (수 분), 노트북 하나로 충분 | 실제 kubeadm 설치 과정을 경험할 수 없음. 컨테이너 런타임/네트워크 설정 실습 불가 | ★★☆☆☆ |
| **Minikube** | 단일 노드(또는 멀티노드 지원) 로컬 클러스터 | 설치 매우 쉬움, 애드온 풍부 | 멀티노드 기본 미지원, kubeadm 과정 생략됨 | ★★☆☆☆ |

### 왜 CKA 학습에는 kubeadm 방식을 권장하는가

```
실제 CKA 시험 환경
      |
      v
kubeadm으로 구축된 멀티노드 클러스터 (Killer.sh와 실제 시험 모두 동일)
      |
      v
시험 문제 예시:
 - "새 워커 노드를 클러스터에 join 하라"
 - "etcd 스냅샷을 백업/복구하라"
 - "kubeadm으로 클러스터를 업그레이드하라"
 - "kubelet 서비스 장애를 진단하라"
```

- 시험 도메인 중 **Cluster Architecture, Installation & Configuration이 25%**를 차지하며, 이 영역의 문제 대부분이 kubeadm 기반 클러스터를 전제로 한다 ([CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) 참고)
- Kind/Minikube는 이런 저수준 구축 과정을 추상화해버리므로, 정작 시험에 나오는 kubeadm/systemd/containerd 레벨의 트러블슈팅 감각을 기를 수 없다
- 따라서 **1주차부터 kubeadm으로 직접 멀티노드 클러스터를 구축하는 것**을 이 커리큘럼의 표준 실습 환경으로 삼는다

### 이 실습에서 만들 클러스터

```
                 ┌─────────────────────┐
                 │   control-plane      │
                 │  (2 vCPU, 2GB RAM+)  │
                 │  192.168.56.10        │
                 └───────────┬──────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
   ┌──────────▼──────────┐       ┌──────────▼──────────┐
   │   worker-1            │       │   worker-2            │
   │ (2 vCPU, 2GB RAM+)    │       │ (2 vCPU, 2GB RAM+)    │
   │  192.168.56.11        │       │  192.168.56.12        │
   └───────────────────────┘       └───────────────────────┘
```

VM은 VirtualBox + Vagrant, 또는 멀티패스(Multipass), 또는 클라우드의 저사양 VM 3대 어떤 것을 써도 무방하다. OS는 **Ubuntu 22.04 LTS** 기준으로 설명한다. 아래 명령어는 **세 노드 모두에서 공통으로 실행**하는 부분과, **control-plane에서만** 또는 **worker에서만** 실행하는 부분을 구분해서 표시한다.

---

## 1단계: 사전 요구사항 설정 (모든 노드 공통)

### 1-1. 호스트명 설정

```bash
# control-plane 노드에서
sudo hostnamectl set-hostname control-plane

# worker-1 노드에서
sudo hostnamectl set-hostname worker-1

# worker-2 노드에서
sudo hostnamectl set-hostname worker-2
```

### 1-2. /etc/hosts에 모든 노드 등록 (모든 노드에서 동일하게)

```bash
sudo tee -a /etc/hosts <<EOF
192.168.56.10 control-plane
192.168.56.11 worker-1
192.168.56.12 worker-2
EOF
```

### 1-3. swap 비활성화 (필수 — kubelet은 swap이 켜져 있으면 정상 동작하지 않음)

```bash
sudo swapoff -a
# 재부팅 후에도 유지되도록 fstab에서 swap 라인 주석 처리
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

> **왜 swap을 꺼야 하는가?** kubelet은 메모리 자원을 정확히 추적해 Pod의 QoS/eviction을 관리하는데, swap이 활성화되면 이 계산이 왜곡되어 스케줄링 오류가 발생할 수 있다. Kubernetes 1.22+부터 일부 조건에서 swap 지원이 실험적으로 추가되었지만, CKA 시험 환경 및 기본 kubeadm 설정은 여전히 swap 비활성화를 전제로 한다.

### 1-4. 커널 모듈 및 네트워크 파라미터 설정

```bash
# 컨테이너 네트워킹에 필요한 커널 모듈 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# iptables가 브릿지 트래픽을 볼 수 있도록 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

## 2단계: 컨테이너 런타임 설치 — containerd (모든 노드 공통)

```bash
# containerd 설치
sudo apt-get update
sudo apt-get install -y containerd

# 기본 설정 파일 생성
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# systemd cgroup driver 사용하도록 설정 (kubelet과 cgroup driver를 일치시켜야 함)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# containerd 재시작 및 부팅 시 자동 시작 설정
sudo systemctl restart containerd
sudo systemctl enable containerd
```

> **흔한 실수**: kubelet과 containerd의 cgroup driver가 다르면(`cgroupfs` vs `systemd`) 노드가 `NotReady` 상태에 빠지거나 kubelet이 시작조차 되지 않는다. 반드시 둘 다 `systemd`로 통일할 것.

---

## 3단계: kubeadm, kubelet, kubectl 설치 (모든 노드 공통)

```bash
# 필요한 패키지 설치
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# Kubernetes 공식 apt 저장소 GPG 키 등록
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 저장소 등록
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# 버전 고정 (자동 업그레이드로 인한 클러스터 버전 불일치 방지)
sudo apt-mark hold kubelet kubeadm kubectl
```

> 시험 환경의 Kubernetes 버전은 응시 시점 최신-1 버전대인 경우가 많으므로, 실습 시점에는 [공식 문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)에서 최신 안정 버전 번호를 확인하고 위 `v1.30` 부분을 맞춰서 진행할 것.

---

## 4단계: kubeadm init (control-plane 노드에서만)

```bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.10 \
  --pod-network-cidr=192.168.0.0/16
```

- `--apiserver-advertise-address`: control-plane이 다른 노드에 자신을 알릴 IP
- `--pod-network-cidr`: Pod에 할당할 IP 대역. **선택한 CNI 플러그인이 요구하는 대역과 일치해야 한다** (아래에서 사용할 Calico의 기본값은 `192.168.0.0/16`)

성공하면 마지막에 아래와 같은 출력이 나온다. **이 `kubeadm join` 명령어를 반드시 복사해둘 것** (worker 노드에서 사용).

```
kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234...
```

토큰을 복사해두지 못했다면 나중에 아래 명령으로 다시 생성할 수 있다.

```bash
kubeadm token create --print-join-command
```

### kubectl 사용을 위한 kubeconfig 설정 (control-plane에서, 일반 사용자로)

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

이 시점에 `kubectl get nodes`를 실행하면 control-plane 노드가 `NotReady` 상태로 보일 것이다. CNI(Pod 네트워크 플러그인)를 아직 설치하지 않았기 때문이다.

---

## 5단계: CNI 플러그인 설치 — Calico (control-plane에서만)

Pod 간 네트워크 통신을 가능하게 하는 CNI(Container Network Interface) 플러그인을 설치해야 노드가 `Ready` 상태가 된다.

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

설치 후 Calico 관련 Pod가 모두 `Running` 상태가 될 때까지 대기한다.

```bash
kubectl get pods -n kube-system -w
```

`Ctrl+C`로 watch를 종료하고, control-plane 노드가 `Ready`로 바뀌었는지 확인한다.

```bash
kubectl get nodes
```

> **다른 선택지**: Flannel, Cilium 등도 널리 쓰인다. CKA 시험에서는 특정 CNI를 강요하지 않지만, 실습 환경에서는 설정이 비교적 단순한 Calico나 Flannel을 권장한다. CNI 종류에 따라 `--pod-network-cidr` 값을 반드시 해당 CNI 문서에서 요구하는 대역으로 맞춰야 한다.

---

## 6단계: kubeadm join (worker-1, worker-2 각각에서)

4단계에서 복사해둔 join 명령을 worker-1과 worker-2에서 각각 실행한다.

```bash
sudo kubeadm join 192.168.56.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234...
```

성공하면 `This node has joined the cluster` 메시지가 출력된다.

---

## 7단계: 클러스터 검증 (control-plane에서)

### 노드 확인

```bash
kubectl get nodes
```

```
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   10m     v1.30.x
worker-1        Ready    <none>          3m      v1.30.x
worker-2        Ready    <none>          2m      v1.30.x
```

세 노드가 모두 `Ready`로 보이지 않는다면 CNI 설치가 제대로 안 됐거나, kubelet 서비스에 문제가 있는지 확인한다 (`sudo systemctl status kubelet`, `journalctl -u kubelet -f`).

### 클러스터 정보 확인

```bash
kubectl cluster-info
```

```
Kubernetes control plane is running at https://192.168.56.10:6443
CoreDNS is running at https://192.168.56.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### 전체 네임스페이스의 Pod 확인

```bash
kubectl get pods -A
```

`kube-system` 네임스페이스에 API Server, etcd, Scheduler, Controller Manager, CoreDNS, kube-proxy, Calico 관련 Pod들이 모두 `Running` 상태로 보여야 정상이다.

```
NAMESPACE     NAME                                      READY   STATUS    RESTARTS
kube-system   etcd-control-plane                        1/1     Running   0
kube-system   kube-apiserver-control-plane               1/1     Running   0
kube-system   kube-controller-manager-control-plane      1/1     Running   0
kube-system   kube-scheduler-control-plane                1/1     Running   0
kube-system   kube-proxy-xxxxx (x3)                       1/1     Running   0
kube-system   calico-node-xxxxx (x3)                      1/1     Running   0
kube-system   coredns-xxxxx (x2)                          1/1     Running   0
```

> **개념 연결**: 여기서 보이는 `kube-apiserver-control-plane`, `etcd-control-plane` 등은 [CKA 01 - 기초 개념 & 환경 세팅 (개념)](CKA-01-기초-개념-환경-세팅-개념.md)에서 설명한 Control Plane 컴포넌트들이 실제로는 **Static Pod**(kubelet이 매니페스트 파일을 직접 읽어 실행하는 특수한 Pod)로 실행되고 있다는 것을 보여준다. 이 부분은 2주차 Cluster Architecture에서 더 깊게 다룬다.

---

## kubectl 기본기 실습

여기부터는 control-plane 노드(또는 kubeconfig가 설정된 아무 머신)에서 진행한다.

### Pod 생성/조회/삭제 — imperative 방식

```bash
# Pod 생성
kubectl run nginx --image=nginx:1.25

# 조회
kubectl get pods
kubectl get pods -o wide          # Node, IP까지 표시
kubectl describe pod nginx        # 이벤트 로그 포함 상세 정보

# 삭제
kubectl delete pod nginx
```

### Deployment 생성/조회/삭제

```bash
# Deployment 생성 (Pod 3개 복제)
kubectl create deployment web --image=nginx:1.25 --replicas=3

# 조회
kubectl get deployments
kubectl get pods -l app=web       # label selector로 필터링
kubectl get replicasets           # Deployment가 만든 ReplicaSet 확인

# 삭제
kubectl delete deployment web
```

> Deployment를 삭제하면 그 아래 ReplicaSet과 Pod까지 연쇄적으로 삭제된다. 반대로 Pod 하나만 지우면 Deployment가 즉시 새 Pod를 재생성한다 — [CKA 01 - 기초 개념 & 환경 세팅 (개념)](CKA-01-기초-개념-환경-세팅-개념.md)에서 설명한 Reconciliation Loop가 실제로 동작하는 것을 직접 확인할 수 있다.

```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get pods -w &
# 다른 터미널에서 Pod 하나를 강제로 삭제해보면 즉시 새 Pod가 뜨는 것이 보인다
kubectl delete pod <pod-이름>
```

### `--dry-run=client -o yaml` 활용 — 시험 핵심 패턴

```bash
# Pod YAML 뼈대 생성 (실제로 생성되지 않음)
kubectl run nginx --image=nginx:1.25 --dry-run=client -o yaml > pod.yaml

# Deployment YAML 뼈대 생성
kubectl create deployment web --image=nginx:1.25 --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# Service YAML 뼈대 생성
kubectl expose deployment web --port=80 --target-port=80 \
  --dry-run=client -o yaml > service.yaml

# 생성된 YAML을 확인하고 필요한 부분만 vim으로 수정
vim pod.yaml

# 최종적으로 선언형(apply)으로 적용
kubectl apply -f pod.yaml
```

생성되는 `pod.yaml` 예시 (일부):

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx:1.25
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

> **꿀팁**: `--dry-run=client -o yaml` 뒤에 `| kubectl neat` (별도 플러그인) 를 붙이면 `creationTimestamp: null`, `status: {}` 같은 불필요한 필드를 제거해준다. 시험 환경에 설치돼 있다면 적극 활용할 것.

### `kubectl explain`으로 필드 확인

YAML 필드 이름이나 구조가 기억나지 않을 때 문서를 찾아 헤매지 말고 바로 조회한다.

```bash
# Pod 최상위 필드 목록
kubectl explain pod

# spec 하위 필드까지
kubectl explain pod.spec

# 특정 필드의 상세 설명 (예: 재시작 정책)
kubectl explain pod.spec.restartPolicy

# 중첩 필드까지 재귀적으로 전부 출력 (구조 파악에 유용)
kubectl explain pod.spec.containers --recursive
```

```
$ kubectl explain pod.spec.restartPolicy
KIND:     Pod
VERSION:  v1

FIELD:    restartPolicy <string>

DESCRIPTION:
    Restart policy for all containers within the pod. One of Always,
    OnFailure, Never. Default to Always.
```

---

## 자주 쓰는 alias 및 자동완성 설정

시험 환경에도 그대로 적용 가능한 설정이므로 지금부터 익숙해지는 것이 좋다.

### alias 설정

```bash
# ~/.bashrc 에 추가
echo 'alias k=kubectl' >> ~/.bashrc

# --dry-run=client -o yaml 조합도 alias로 줄여두면 타이핑 시간을 크게 절약
echo 'export do="--dry-run=client -o yaml"' >> ~/.bashrc

source ~/.bashrc
```

사용 예:

```bash
k get pods
k run nginx --image=nginx $do > pod.yaml
```

### kubectl 자동완성 설정

```bash
# 현재 세션에 즉시 적용
source <(kubectl completion bash)

# 영구 적용
echo 'source <(kubectl completion bash)' >> ~/.bashrc

# alias(k)에도 자동완성이 동작하도록 설정
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc

source ~/.bashrc
```

> zsh를 사용한다면 `kubectl completion zsh` 및 `.zshrc`에 동일하게 적용한다. 시험장 환경은 bash 기준으로 세팅되어 있는 경우가 많으므로 bash 기준으로 손에 익혀두는 것을 권장한다.

---

## 체크리스트

- [ ] 3개 노드(control-plane, worker-1, worker-2) 모두 swap을 비활성화했다
- [ ] containerd를 설치하고 `SystemdCgroup = true`로 설정했다
- [ ] kubeadm, kubelet, kubectl을 설치하고 버전을 `apt-mark hold`로 고정했다
- [ ] `kubeadm init`으로 control-plane을 초기화하고 kubeconfig를 설정했다
- [ ] Calico(또는 다른 CNI)를 설치해 노드가 `NotReady` → `Ready`로 바뀌는 것을 확인했다
- [ ] `kubeadm join`으로 worker-1, worker-2를 클러스터에 합류시켰다
- [ ] `kubectl get nodes`로 3개 노드 모두 `Ready` 상태를 확인했다
- [ ] `kubectl cluster-info`, `kubectl get pods -A`로 클러스터 상태를 점검했다
- [ ] imperative 명령으로 Pod와 Deployment를 생성/조회/삭제해봤다
- [ ] `--dry-run=client -o yaml` 조합으로 YAML 뼈대를 생성하고 apply까지 해봤다
- [ ] `kubectl explain`으로 최소 3개 이상의 리소스 필드를 조회해봤다
- [ ] `alias k=kubectl`과 kubectl 자동완성을 설정했다

---

> 다음 단계: [CKA 02 - Cluster Architecture & Installation (실습)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md) — Static Pod, etcd 백업/복구, kubeadm 업그레이드를 직접 실습한다.
