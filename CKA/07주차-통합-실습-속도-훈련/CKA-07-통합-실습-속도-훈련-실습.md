---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 07 - 통합 실습 & 속도 훈련 (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 07 - 통합 실습 & 속도 훈련 (개념)](CKA-07-통합-실습-속도-훈련-개념.md) | 이전: [CKA 06 - Troubleshooting (실습)](../06주차-Troubleshooting/CKA-06-Troubleshooting-실습.md) | 다음: [CKA 08 - 모의고사 & 최종 점검 (실습)](../08주차-모의고사-최종-점검/CKA-08-모의고사-최종-점검-실습.md)

---

## 실습 목표

- 시험 시작 직후 할 vimrc/alias/자동완성 설정을 손에 익힌다
- 여러 컨텍스트를 오가며 문제를 푸는 흐름을 반복 훈련한다
- 2~6주차 지식을 섞은 복합 문제를 시간 제한(문제당 7분) 안에 푸는 연습을 한다
- 공식 문서에서 예제를 3분 안에 찾아 붙여넣는 연습을 한다

---

## 1. 시험 환경 세팅 실습

### 1-1. vimrc 설정

YAML은 스페이스 기반 들여쓰기를 요구한다. tab 키를 눌렀을 때 스페이스로 변환되지 않으면 파싱 에러가 나기 쉽다. 시험 시작 직후 다음을 `~/.vimrc`에 입력한다.

```bash
cat <<EOF > ~/.vimrc
set tabstop=2
set shiftwidth=2
set expandtab
set number
set autoindent
set cursorline
EOF
```

| 옵션 | 의미 |
|---|---|
| `tabstop=2` / `shiftwidth=2` | tab 및 들여쓰기 이동 폭을 2칸으로 |
| `expandtab` | tab 입력 시 실제로는 스페이스로 변환 (YAML 필수) |
| `autoindent` / `cursorline` | 이전 줄 들여쓰기 유지 / 현재 줄 강조 |

**연습**: vimrc를 지운 상태로 YAML을 한 번 편집해 들여쓰기가 깨지는 것을 직접 경험한 뒤, 설정을 적용하고 다시 편집해 차이를 체감한다.

### 1-2. kubectl 자동완성 + alias 스크립트

시험 시작 시 아래 스크립트를 그대로 입력할 수 있도록 손에 익힌다. `~/.bashrc`에 넣거나 매번 터미널에 직접 붙여넣어도 된다.

```bash
# 자동완성 활성화
source <(kubectl completion bash)

# alias 설정
alias k=kubectl
complete -o default -F __start_kubectl k

# 자주 쓰는 dry-run 플래그 단축
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"

# 확인
alias
k get nodes
```

| 단축 | 원래 명령 | 예시 |
|---|---|---|
| `k` | `kubectl` | `k get po` |
| `$do` | `--dry-run=client -o yaml` | `k run nginx --image=nginx $do > pod.yaml` |
| `$now` | `--force --grace-period=0` | `k delete po nginx $now` |

**연습**: alias 없이 `kubectl get pods -A`와 alias로 `k get po -A`를 각각 5번 타이핑해 시간을 비교해본다. 사소해 보이는 차이가 2시간 동안 누적된다.

---

## 2. 컨텍스트 전환 모의 시나리오

시험은 문제마다 다른 클러스터를 지정한다. 아래 시나리오로 "컨텍스트 확인 → 전환 → 작업 → 원래대로 복귀" 흐름을 반복 훈련한다.

### 사전 준비 (연습용 컨텍스트 2개 생성)

로컬에 클러스터가 하나뿐이라면, 네임스페이스를 클러스터처럼 취급해 컨텍스트를 흉내낼 수 있다.

```bash
kubectl create namespace ctx-a
kubectl create namespace ctx-b

C=$(kubectl config view -o jsonpath='{.clusters[0].name}')
U=$(kubectl config view -o jsonpath='{.users[0].name}')
kubectl config set-context context-a --cluster=$C --user=$U --namespace=ctx-a
kubectl config set-context context-b --cluster=$C --user=$U --namespace=ctx-b

kubectl config get-contexts
```

### 시나리오: A에서 Pod 생성 → B에서 Deployment 스케일 → 원래 컨텍스트로 복귀

```bash
# 1. 원래 컨텍스트 기록 (시험에서는 항상 첫 동작)
kubectl config current-context
# 예: original-context

# 2. context-a로 전환 후 Pod 생성
kubectl config use-context context-a
kubectl config current-context   # 반드시 재확인
kubectl run web-a --image=nginx --dry-run=client -o yaml | kubectl create -f -
kubectl get pod web-a

# 3. context-b로 전환 후 Deployment 생성 및 스케일
kubectl config use-context context-b
kubectl config current-context   # 재확인
kubectl create deployment app-b --image=nginx --replicas=1
kubectl scale deployment app-b --replicas=3
kubectl get deployment app-b

# 4. 원래 컨텍스트로 복귀
kubectl config use-context original-context
kubectl config current-context   # 최종 확인
```

**체크 포인트**: 각 단계 전환 직후 `kubectl config current-context`를 확인하지 않고 넘어갔다면, 그 자체가 실전에서의 실수 패턴이다. 이 확인 동작이 반사적으로 나올 때까지 반복한다. 연습이 끝나면 `kubectl config delete-context context-a context-b`와 `kubectl delete namespace ctx-a ctx-b`로 정리한다.

---

## 3. 복합 시나리오 문제 훈련 (문제당 7분 제한)

2~6주차 지식(Architecture, Workloads, Networking, Storage, Troubleshooting)을 섞은 문제를 직접 만들어 풀어본다. 타이머를 켜고, 7분이 지나면 강제로 멈추고 flag 후 다음으로 넘어가는 훈련을 한다.

### 문제 1 — Workloads + Scheduling (배점: 낮음, 목표 3분)

> `web-ns` 네임스페이스에 `nginx:1.25` 이미지를 사용하는 Deployment `web`을 생성하라. replica는 2개이며, 각 Pod는 `disktype=ssd` 레이블이 붙은 노드에만 스케줄되어야 한다.

<details>
<summary>풀이 힌트 (먼저 스스로 풀어본 뒤 확인)</summary>

```bash
kubectl create ns web-ns
kubectl create deployment web --image=nginx:1.25 --replicas=2 -n web-ns $do > web.yaml
# web.yaml의 spec.template.spec에 nodeSelector 추가
#   nodeSelector:
#     disktype: ssd
kubectl apply -f web.yaml
```
</details>

### 문제 2 — Services & Networking (배점: 중간, 목표 6분)

> `web` Deployment(문제 1에서 생성)를 `ClusterIP` Service `web-svc`로 노출하고 포트 80을 사용하라. 그리고 `frontend` 레이블이 붙은 Pod에서만 `web-svc`로 접근 가능하도록 NetworkPolicy를 작성하라.

<details>
<summary>풀이 힌트</summary>

```bash
kubectl expose deployment web -n web-ns --port=80 --name=web-svc

# NetworkPolicy는 공식 문서 예제를 검색해 붙여넣는 것이 빠름
# kubernetes.io/docs/concepts/services-networking/network-policies/ 참고
kubectl create -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend
  namespace: web-ns
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
EOF
```
</details>

### 문제 3 — Storage (배점: 중간, 목표 6분)

> `data-pvc`라는 이름으로 5Gi 크기의 PVC를 `web-ns`에 생성하고, 문제 1의 Deployment `web`에 `/usr/share/nginx/html` 경로로 마운트하라.

<details>
<summary>풀이 힌트</summary>

```bash
kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  namespace: web-ns
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
EOF

kubectl edit deployment web -n web-ns
# volumes / volumeMounts 필드 추가
```
</details>

### 문제 4 — Troubleshooting (배점: 높음, 목표 10분)

> `web-ns`의 Pod 하나가 `CrashLoopBackOff` 상태다. 원인을 찾아 복구하라. (일부러 잘못된 command를 넣은 Pod를 만들어 연습)

<details>
<summary>연습용 장애 상황 만들기 + 풀이 흐름</summary>

```bash
kubectl run broken --image=nginx -n web-ns --command -- /bin/badcommand   # 장애 유발
kubectl describe pod broken -n web-ns   # Events에서 원인 확인
kubectl logs broken -n web-ns
kubectl delete pod broken -n web-ns && kubectl run broken --image=nginx -n web-ns   # 복구
```
</details>

### 문제 5 — Cluster Architecture (배점: 높음, 목표 8분)

> RBAC을 이용해 `web-ns` 네임스페이스에서 Pod 조회만 가능한 ServiceAccount `viewer`를 생성하라.

<details>
<summary>풀이 힌트</summary>

```bash
kubectl create serviceaccount viewer -n web-ns
kubectl create role pod-reader --verb=get,list,watch --resource=pods -n web-ns
kubectl create rolebinding viewer-binding --role=pod-reader \
  --serviceaccount=web-ns:viewer -n web-ns

# 검증
kubectl auth can-i list pods -n web-ns --as=system:serviceaccount:web-ns:viewer
```
</details>

### 훈련 방법

1. 타이머 앱(또는 `date +%s`로 시작/종료 시각 기록)으로 문제당 시간을 재며 순서대로 푼다.
2. 시간 안에 못 풀면 즉시 멈추고 "flag" 표시(메모장에 문제 번호 적어두기)를 하고 다음 문제로 넘어간다.
3. 5문제를 모두 훑은 뒤, flag된 문제로 돌아가 마무리한다.
4. 총 소요 시간과 정답률을 기록해 주 후반에 다시 반복, 속도가 개선되는지 확인한다.

---

## 4. 공식 문서에서 예제 빠르게 찾기 연습

시험 중 허용된 유일한 자료가 kubernetes.io이므로, 이 사이트 안에서 예제를 찾는 속도 자체가 실력이다. 아래 항목들을 각각 **3분 안에** 찾아 터미널에 붙여넣는 연습을 한다.

| 찾을 예제 | 검색 경로 힌트 |
|---|---|
| 기본 Pod spec / multi-container Pod | Concepts → Workloads → Pods (→ Init Containers) |
| NetworkPolicy 예제 (deny-all, allow 특정 포트) | Concepts → Services, Load Balancing, and Networking → Network Policies |
| RBAC Role/RoleBinding 예제 | Reference → Access Authn Authz → RBAC |
| PV/PVC 예제 (hostPath, StorageClass) | Concepts → Storage → Persistent Volumes |
| Static Pod / taint-toleration 예제 | Tasks → Configure Pods and Containers → Static Pods / Concepts → Scheduling |

### 연습 방법

표의 항목을 무작위로 골라 kubernetes.io 검색창/좌측 목차 탐색만으로 3분 안에 예제 페이지를 찾고, 예제 YAML을 복사해 `kubectl create -f - <<EOF ... EOF`로 실제 클러스터에 적용해본다. 3분을 넘기면 어떤 경로에서 헤맸는지 기록하고 다음엔 그 경로를 단축한다. URL을 외우기보다 사이트 내 검색/네비게이션 구조에 익숙해지는 것이 중요하며, 같은 예제를 2~3번 반복하면 경로가 몸에 남는다.

---

## 5. 이번 주 정리 및 체크리스트

- [ ] `~/.vimrc`에 `tabstop=2`, `shiftwidth=2`, `expandtab`을 설정하는 것을 보지 않고 타이핑할 수 있다
- [ ] `alias k=kubectl` + 자동완성 스크립트를 1분 안에 설정할 수 있다
- [ ] `kubectl config current-context` 확인을 컨텍스트 전환마다 습관적으로 실행한다
- [ ] 컨텍스트 A → B → 원래 컨텍스트로 돌아오는 시나리오를 실수 없이 완료했다
- [ ] 복합 시나리오 문제 5개를 문제당 7분 제한으로 풀어봤고, 최소 3개는 시간 안에 완료했다
- [ ] kubernetes.io에서 Pod spec, NetworkPolicy, RBAC 예제를 각각 3분 안에 찾을 수 있다
- [ ] `kubectl explain`, `--dry-run=client -o yaml`, heredoc(`<<EOF`) 세 가지를 실제 손으로 써서 체화했다
- [ ] flag 후 다음 문제로 넘어가는 습관이 몸에 배어, 한 문제에 10분 이상 매몰되지 않는다

---

> 다음 단계: [CKA 08 - 모의고사 & 최종 점검 (실습)](../08주차-모의고사-최종-점검/CKA-08-모의고사-최종-점검-실습.md) — Killer.sh 모의고사로 실전과 동일한 조건에서 최종 점검한다.
