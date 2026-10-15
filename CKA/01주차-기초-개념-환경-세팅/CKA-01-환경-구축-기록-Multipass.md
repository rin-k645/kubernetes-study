---
tags:
  - kubernetes
  - cka
  - study
  - infra
created: 2026-10-13
---

# CKA 01 - 환경 구축 기록 (Multipass on host-server)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습 본문: [CKA 01 - 기초 개념 & 환경 세팅 (실습)](CKA-01-기초-개념-환경-세팅-실습.md)

실습 문서의 "VM은 VirtualBox + Vagrant, 또는 Multipass, 또는 클라우드 VM 3대 어떤 것을 써도 무방하다"는 부분을, 실제 보유 중인 베어메탈/클라우드 서버 2대(host-server, cluster-hypervisor) 환경에 맞춰 어떻게 구체화했는지 기록한다.

> **주의**: 이 문서는 원본(개인 Obsidian 보관용) 기록에서 서버 공인 IP, 내부 IP, kubeadm 부트스트랩 토큰 등 민감 정보를 자리표시자로 치환한 공개용 버전이다.

---

## 왜 이 방식을 선택했는가

### 보유 서버 현황

| 서버 | 주소 | 스펙 | 용도 |
|---|---|---|---|
| host-server | `<HOST-SERVER-PUBLIC-IP>` | 16 vCPU / 125 GiB / 3.8 TB | 미사용 (Stage 4 합류 예정) |
| cluster-hypervisor | `<CLUSTER-HYPERVISOR-PUBLIC-IP>` | 32 vCPU / 251 GiB / 3.8 TB | 하이퍼바이저로 이미 운영 중 (VM 3대로 k8s 클러스터 운영) |

cluster-hypervisor는 이미 다른 용도로 쓰이고 있어 건드릴 수 없다. 따라서 CKA 실습용 멀티노드 클러스터는 **미사용 상태인 host-server 한 대 위에, 중첩 가상화로 VM 3대(control-plane, worker-1, worker-2)를 올리는 방식**으로 구축했다.

### 왜 "중첩 가상화 지원 여부"를 먼저 확인해야 했는가

host-server 자체가 이미 어떤 하이퍼바이저 위에서 동작하는 VM이다 (`systemd-detect-virt` 결과 `kvm`). 이런 VM 안에서 또 다른 VM(Multipass/QEMU 기반)을 띄우려면, 상위 하이퍼바이저가 **중첩 가상화(nested virtualization)** 를 게스트에게 노출해줘야 한다. 이게 꺼져 있으면:

- CPU 가상화 명령어(`vmx`/`svm`)가 게스트에 보이지 않아 `/dev/kvm`이 아예 생성되지 않거나
- 생성되더라도 하드웨어 가속 없이 완전 소프트웨어 에뮬레이션으로 동작해 2vCPU짜리 VM 3대조차 체감상 매우 느려진다 (컨테이너 런타임·kubelet 부팅 지연, apt 설치 타임아웃 등으로 실습 자체가 괴로워짐)

그래서 Multipass를 설치하기 전에 먼저 host-server에서 아래를 확인했다.

```bash
ssh host-server "systemd-detect-virt; egrep -o 'vmx|svm' /proc/cpuinfo | sort -u; lsmod | grep -i kvm; ls -la /dev/kvm; nproc; free -h"
```

**확인 결과:**

```
--- virt-what ---
kvm                                    # host-server 자신이 KVM 위의 게스트
--- cpu flags ---
vmx                                    # Intel VT-x가 게스트까지 노출됨 → 중첩 가상화 가능
--- kvm module ---
kvm_intel             557056  0
kvm                  1531904  1 kvm_intel
irqbypass              12288  1 kvm
--- /dev/kvm ---
crw-rw---- 1 root kvm 10, 232 ... /dev/kvm   # 디바이스 노드 존재 → 하드웨어 가속 사용 가능
--- resources ---
16 vCPU
125Gi 메모리 (여유 122Gi), swap 0B
```

→ 중첩 가상화가 하드웨어 가속까지 정상 지원됨을 확인했고, VM 3대(2vCPU/2GB씩, 총 6vCPU/6GB)를 올려도 리소스 여유가 충분해 **Multipass(QEMU+KVM 기반) 방식으로 진행하기로 결정**했다.

> 만약 이 확인에서 `vmx`/`svm` 플래그가 안 보이거나 `/dev/kvm`이 없었다면, 중첩 가상화가 막혀있는 것이므로 Multipass 대신 클라우드에서 별도의 저사양 VM 3대를 각각 발급받는 방식으로 전환해야 했다.

---

## 1단계: host-server에 Multipass 설치

```bash
ssh host-server
sudo snap install multipass
multipass version
```

**결과:**
```
multipass 1.16.4 from Canonical✓ installed
multipass   1.16.4
multipassd  1.16.4
```

---

## 2단계: VM 3대 생성

실습 문서의 노드 스펙(2 vCPU, 2GB RAM)에 맞춰 Ubuntu 22.04 이미지로 3대를 생성했다.

```bash
multipass launch 22.04 --name control-plane --cpus 2 --memory 2G --disk 10G
multipass launch 22.04 --name worker-1 --cpus 2 --memory 2G --disk 10G
multipass launch 22.04 --name worker-2 --cpus 2 --memory 2G --disk 10G
```

---

## 3단계: 할당된 IP 확인

```bash
multipass list
```

**결과:**

| Name | State | IPv4 | Image |
|---|---|---|---|
| control-plane | Running | `<CONTROL-PLANE-IP>` | Ubuntu 22.04 LTS |
| worker-1 | Running | `<WORKER-1-IP>` | Ubuntu 22.04 LTS |
| worker-2 | Running | `<WORKER-2-IP>` | Ubuntu 22.04 LTS |

> **중요**: 실습 본문([CKA 01 - 기초 개념 & 환경 세팅 (실습)](CKA-01-기초-개념-환경-세팅-실습.md))은 VirtualBox 기준 `192.168.56.10/11/12`를 예시로 들지만, Multipass는 자체 가상 네트워크에서 IP를 할당한다. 이후 모든 단계(`/etc/hosts`, `kubeadm init --apiserver-advertise-address`, join 명령 등)에서 **문서의 `192.168.56.x`를 실제 할당된 IP로 치환**해서 진행해야 한다.

### 실제 환경에 맞춘 치환표

| 역할 | 문서 예시 IP | 실제 IP (host-server Multipass) |
|---|---|---|
| control-plane | 192.168.56.10 | `<CONTROL-PLANE-IP>` |
| worker-1 | 192.168.56.11 | `<WORKER-1-IP>` |
| worker-2 | 192.168.56.12 | `<WORKER-2-IP>` |

---

## VM 접속 방법 메모

```bash
# 대화형 쉘
multipass shell control-plane

# 외부에서 단발성 명령 실행 (스크립팅/반복 작업에 유리)
multipass exec control-plane -- sudo hostnamectl set-hostname control-plane
```

---

## 1단계: 사전 요구사항 설정 (완료)

3개 VM 모두에서 `multipass exec` / `multipass shell`로 아래 작업을 수행했다.

### 1-1. 호스트명 설정

```bash
multipass exec control-plane -- sudo hostnamectl set-hostname control-plane
multipass exec worker-1 -- sudo hostnamectl set-hostname worker-1
multipass exec worker-2 -- sudo hostnamectl set-hostname worker-2
```

### 1-2. /etc/hosts에 모든 노드 등록 (실제 IP 기준)

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- sudo tee -a /etc/hosts <<EOF
<CONTROL-PLANE-IP> control-plane
<WORKER-1-IP> worker-1
<WORKER-2-IP> worker-2
EOF
done
```

> **참고**: Multipass 기본 이미지는 cloud-init의 `manage_etc_hosts: True` 설정 때문에 `/etc/hosts` 상단에 경고 주석이 붙는다 ("변경사항을 영구 반영하려면 `/etc/cloud/cloud.cfg`를 수정하라"는 안내). 지금 이 실습에서는 VM을 재생성하지 않는 한 문제가 되지 않으므로 무시해도 된다. 다만 VM을 재부팅했을 때 추가한 라인이 cloud-init에 의해 덮어써지는 것처럼 보이면, 이 설정 때문이라는 것을 기억할 것.

### 1-3. swap 비활성화

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- sudo swapoff -a
  multipass exec $vm -- sudo sed -i '/ swap / s/^/#/' /etc/fstab
done
```

Multipass 기본 이미지는 애초에 swap이 구성되어 있지 않아(`Swap: 0B`), 이 단계는 형식상 실행만 된 것에 가깝다.

### 1-4. 커널 모듈 및 네트워크 파라미터 설정

세 VM 각각에 `multipass shell`로 들어가 동일하게 실행:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 검증 결과

```bash
for vm in control-plane worker-1 worker-2; do
  echo "=== $vm ==="
  multipass exec $vm -- cat /etc/hosts
  multipass exec $vm -- free -h
done
```

3개 VM 모두 `/etc/hosts`에 세 노드 매핑이 정상 반영되었고, `Swap: 0B`로 비활성화 상태를 확인했다.

| VM | 메모리(total) | Swap |
|---|---|---|
| control-plane | 1.9Gi | 0B |
| worker-1 | 1.9Gi | 0B |
| worker-2 | 1.9Gi | 0B |

**1단계 완료.**

---

## 2단계: 컨테이너 런타임 설치 — containerd (완료)

3개 VM 모두에서 동일하게 실행:

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- bash -c '
    sudo apt-get update &&
    sudo apt-get install -y containerd &&
    sudo mkdir -p /etc/containerd &&
    containerd config default | sudo tee /etc/containerd/config.toml > /dev/null &&
    sudo sed -i "s/SystemdCgroup = false/SystemdCgroup = true/" /etc/containerd/config.toml &&
    sudo systemctl restart containerd &&
    sudo systemctl enable containerd
  '
done
```

### 검증 결과

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- grep SystemdCgroup /etc/containerd/config.toml
  multipass exec $vm -- systemctl is-active containerd
done
```

| VM | SystemdCgroup | containerd 상태 |
|---|---|---|
| control-plane | true | active |
| worker-1 | true | active |
| worker-2 | true | active |

**2단계 완료.**

---

## 3단계: kubeadm, kubelet, kubectl 설치 (완료)

3개 VM 모두에서 동일하게 실행:

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- bash -c '
    sudo apt-get update &&
    sudo apt-get install -y apt-transport-https ca-certificates curl gpg &&
    sudo mkdir -p /etc/apt/keyrings &&
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg &&
    echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list &&
    sudo apt-get update &&
    sudo apt-get install -y kubelet kubeadm kubectl &&
    sudo apt-mark hold kubelet kubeadm kubectl
  '
done
```

> 원본 문서에 없던 `sudo mkdir -p /etc/apt/keyrings`를 추가했다. Ubuntu 22.04 기본 이미지엔 이 디렉터리가 기본으로 없을 수 있어, 없으면 `gpg --dearmor -o` 단계에서 실패할 수 있기 때문이다.

### 검증 결과

```bash
for vm in control-plane worker-1 worker-2; do
  multipass exec $vm -- kubeadm version -o short
  multipass exec $vm -- kubelet --version
done
```

3대 모두 `v1.30.14`로 설치 확인.

| VM | kubeadm | kubelet | kubectl |
|---|---|---|---|
| control-plane | v1.30.14 | v1.30.14 | v1.30.14 |
| worker-1 | v1.30.14 | v1.30.14 | v1.30.14 |
| worker-2 | v1.30.14 | v1.30.14 | v1.30.14 |

**3단계 완료.**

---

## 4단계: kubeadm init (완료)

control-plane에서 실제 IP를 반영해 초기화 실행:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<CONTROL-PLANE-IP> \
  --pod-network-cidr=192.168.0.0/16
```

**발급된 join 명령어** (worker-1, worker-2에서 6단계에 사용 — 기본 24시간 후 토큰 만료):

```
kubeadm join <CONTROL-PLANE-IP>:6443 --token <BOOTSTRAP-TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA-CERT-HASH>
```

> 토큰이 만료되면 control-plane에서 `kubeadm token create --print-join-command`로 재발급 가능.

> `kubeadm init` 로그에 `remote version is much newer: v1.37.1; falling back to: stable-1.30` 경고가 떴다 — 설치한 kubeadm(v1.30.14)보다 훨씬 최신 stable(v1.37.1)이 나와 있다는 안내일 뿐, 설치 자체는 지정한 v1.30.14로 정상 진행된 것이므로 무시해도 된다.

### kubeconfig 설정

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 확인 결과

```
$ kubectl get nodes
NAME            STATUS     ROLES           AGE   VERSION
control-plane   NotReady   control-plane   82s   v1.30.14
```

예상대로 `NotReady` — CNI 미설치 상태라 정상이다.

**4단계 완료.**

---

## 5단계: CNI 플러그인 설치 — Calico (완료)

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
kubectl get pods -n kube-system -w
```

### 확인 결과

```
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-564985c589-l898d   1/1     Running   0          66s
calico-node-vsxj5                          1/1     Running   0          66s
coredns-55cb58b774-gpcf4                   1/1     Running   0          3m51s
coredns-55cb58b774-njwfc                   1/1     Running   0          3m51s
etcd-control-plane                         1/1     Running   0          4m6s
kube-apiserver-control-plane               1/1     Running   0          4m7s
kube-controller-manager-control-plane      1/1     Running   0          4m6s
kube-proxy-85xr7                           1/1     Running   0          3m51s
kube-scheduler-control-plane               1/1     Running   0          4m6s
```

```
$ kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   4m34s   v1.30.14
```

`NotReady` → `Ready` 전환 확인.

**5단계 완료.**

---

## 6단계: kubeadm join (완료)

host-server 호스트에서 `multipass exec`로 각 worker에 join 명령 전달:

```bash
multipass exec worker-1 -- sudo kubeadm join <CONTROL-PLANE-IP>:6443 --token <BOOTSTRAP-TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA-CERT-HASH>

multipass exec worker-2 -- sudo kubeadm join <CONTROL-PLANE-IP>:6443 --token <BOOTSTRAP-TOKEN> \
        --discovery-token-ca-cert-hash sha256:<CA-CERT-HASH>
```

### 확인 결과

```
$ multipass exec control-plane -- kubectl get nodes
NAME            STATUS   ROLES           AGE     VERSION
control-plane   Ready    control-plane   6m34s   v1.30.14
worker-1        Ready    <none>          56s     v1.30.14
worker-2        Ready    <none>          17s     v1.30.14
```

3개 노드 모두 `Ready`. **6단계 완료 — 3노드 kubeadm 클러스터 구축 성공.**

---

## 7단계: 클러스터 검증 (완료)

```bash
kubectl cluster-info
kubectl get pods -A
```

### 확인 결과

```
Kubernetes control plane is running at https://<CONTROL-PLANE-IP>:6443
CoreDNS is running at https://<CONTROL-PLANE-IP>:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-564985c589-l898d   1/1     Running   0          4m37s
kube-system   calico-node-64xgw                          1/1     Running   0          83s
kube-system   calico-node-cjzxt                          1/1     Running   0          2m2s
kube-system   calico-node-vsxj5                          1/1     Running   0          4m37s
kube-system   coredns-55cb58b774-gpcf4                   1/1     Running   0          7m22s
kube-system   coredns-55cb58b774-njwfc                   1/1     Running   0          7m22s
kube-system   etcd-control-plane                         1/1     Running   0          7m37s
kube-system   kube-apiserver-control-plane               1/1     Running   0          7m38s
kube-system   kube-controller-manager-control-plane      1/1     Running   0          7m37s
kube-system   kube-proxy-85xr7                           1/1     Running   0          7m22s
kube-system   kube-proxy-bpkq9                           1/1     Running   0          83s
kube-system   kube-proxy-q94q5                           1/1     Running   0          2m2s
kube-system   kube-scheduler-control-plane               1/1     Running   0          7m37s
```

API Server, etcd, Scheduler, Controller Manager, CoreDNS(x2), kube-proxy(x3), Calico(x4 — kube-controllers 1 + node 3) 전부 `Running`. **7단계 완료.**

---

## 환경 구축 마무리

host-server 한 대(Multipass 중첩 가상화) 위에 kubeadm 기반 3노드 클러스터(control-plane 1 + worker 2) 구축을 완료했다. 1~7단계 전부 완료.

## 다음 단계

[CKA 01 - 기초 개념 & 환경 세팅 (실습)](CKA-01-기초-개념-환경-세팅-실습.md) 후반부 **"kubectl 기본기 실습"** (Pod/Deployment 생성·조회·삭제, `--dry-run=client -o yaml` 패턴, `kubectl explain`, alias/자동완성 설정)으로 진행한다.

이후 커리큘럼: [CKA 02 - Cluster Architecture & Installation (실습)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md) — Static Pod, etcd 백업/복구, kubeadm 업그레이드.
