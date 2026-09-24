---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 04 - Services & Networking (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 04 - Services & Networking (개념)](CKA-04-Services-Networking-개념.md) | 이전: [CKA 03 - Workloads & Scheduling (실습)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-실습.md) | 다음: [CKA 05 - Storage (실습)](../05주차-Storage/CKA-05-Storage-실습.md)

---

## 사전 준비

이 실습은 kubeadm으로 구축한 멀티노드 클러스터(control-plane 1 + worker 2) 기준이며, CNI로 NetworkPolicy를 지원하는 Calico 또는 Cilium이 설치되어 있다고 가정한다. (Flannel만 단독 설치된 경우 NetworkPolicy 실습은 동작하지 않으니 주의)

```bash
kubectl get pods -n kube-system | grep -i calico     # CNI가 NetworkPolicy를 지원하는지 확인
kubectl get pods -n ingress-nginx                    # Ingress Controller 설치 여부 확인

# 없다면 설치
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
kubectl get pods -n ingress-nginx -w

# 실습용 네임스페이스
kubectl create namespace net-lab
kubectl config set-context --current --namespace=net-lab
```

---

## 실습 1. ClusterIP Service 생성 및 접속 테스트

### 1-1. 백엔드 Deployment + ClusterIP Service 생성

```bash
kubectl create deployment backend --image=nginx --replicas=3 -n net-lab
kubectl expose deployment backend --port=80 --target-port=80 --name=backend-svc -n net-lab
kubectl get svc backend-svc -n net-lab
```

출력 예시:

```
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
backend-svc   ClusterIP   10.96.123.45   <none>        80/TCP    5s
```

### 1-2. Endpoints 확인 및 접속 테스트

```bash
kubectl get endpoints backend-svc -n net-lab   # Pod 3개의 IP:80이 모두 나열되어야 정상

# ClusterIP는 클러스터 외부에서 직접 접근 불가 — 임시 디버깅 Pod로 내부에서 접속
kubectl run tmp-curl --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- \
  wget -qO- backend-svc.net-lab.svc.cluster.local
```

nginx 기본 페이지 HTML이 출력되면 성공이다.

---

## 실습 2. NodePort Service 생성 및 외부 접속 테스트

### 2-1. NodePort Service 생성 (declarative)

```bash
cat <<'EOF' > nodeport-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  namespace: net-lab
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
EOF

kubectl apply -f nodeport-svc.yaml
kubectl get svc backend-nodeport -n net-lab
```

출력 예시:

```
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
backend-nodeport   NodePort   10.96.55.10    <none>        80:30080/TCP   3s
```

### 2-2. 노드 IP로 접속 테스트

```bash
kubectl get nodes -o wide   # 워커 노드 IP 목록 확인

# 어느 노드 IP로 접속해도 동일하게 응답해야 한다
curl http://<worker1-IP>:30080
curl http://<worker2-IP>:30080
curl http://<control-plane-IP>:30080   # Pod가 없는 노드도 포함해 확인
```

> 확인 포인트: NodePort는 Pod가 없는 노드로 요청을 보내도 kube-proxy가 iptables/IPVS 규칙으로 다른 노드의 Pod까지 라우팅해준다. 이것이 "어느 노드로 가도 동작한다"는 말의 실체다. (심화: 워커 노드에 ssh 후 `sudo iptables -t nat -L KUBE-NODEPORTS -n`으로 실제 규칙을 확인할 수 있다.)

---

## 실습 3. Ingress 리소스 작성 및 배포 (경로 기반 라우팅)

### 3-1. 두 번째 백엔드(frontend) 준비

```bash
kubectl create deployment frontend --image=httpd --replicas=2 -n net-lab
kubectl expose deployment frontend --port=80 --target-port=80 --name=frontend-svc -n net-lab
```

이제 `backend-svc`(nginx)와 `frontend-svc`(httpd) 두 Service가 있다.

### 3-2. 경로 기반 Ingress 작성

```bash
cat <<'EOF' > path-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: net-lab-ingress
  namespace: net-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 80
          - path: /frontend
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
EOF

kubectl apply -f path-ingress.yaml
kubectl get ingress -n net-lab
kubectl describe ingress net-lab-ingress -n net-lab
```

### 3-3. Ingress Controller의 NodePort로 접속 테스트

```bash
# ingress-nginx가 어떤 포트로 노출되어 있는지 확인
kubectl get svc -n ingress-nginx

# 예: ingress-nginx-controller가 NodePort 30443(https)/31080(http)라고 가정
curl http://<임의_노드_IP>:31080/backend
curl http://<임의_노드_IP>:31080/frontend
```

`/backend`는 nginx 기본 페이지, `/frontend`는 httpd 기본 페이지("It works!")가 나와야 한다.

### 3-4. 호스트 기반 라우팅 추가 (선택, `path-ingress.yaml`에 rule 추가)

`spec.rules`에 `host: backend.local` 항목을 추가한 뒤 재적용하고, 실제 DNS 등록 없이 Host 헤더로 검증한다.

```bash
kubectl apply -f path-ingress.yaml
curl -H "Host: backend.local" http://<임의_노드_IP>:31080/
```

---

## 실습 4. NetworkPolicy — Deny-all에서 단계적으로 허용하기

### 4-1. 시나리오 준비 — 두 네임스페이스와 라벨링

```bash
kubectl create namespace secure-zone
kubectl create namespace attacker-zone

kubectl run db --image=nginx --labels=app=db -n secure-zone
kubectl expose pod db --port=80 --name=db-svc -n secure-zone

# secure-zone의 정당한 클라이언트
kubectl run allowed-client --image=busybox:1.36 --labels=role=trusted -n secure-zone --command -- sleep 3600
# attacker-zone의 외부 침입자 역할
kubectl run intruder --image=busybox:1.36 -n attacker-zone --command -- sleep 3600
```

### 4-2. 정책 적용 전 — 기본 상태는 전부 허용됨을 확인

```bash
kubectl exec -it allowed-client -n secure-zone -- wget -qO- --timeout=3 db-svc
kubectl exec -it intruder -n attacker-zone -- wget -qO- --timeout=3 db-svc.secure-zone.svc.cluster.local
```

둘 다 응답이 와야 한다. 이것이 "Kubernetes는 기본적으로 모두 허용"이라는 원리의 실증이다.

### 4-3. 1단계 — secure-zone에 Deny-all Ingress 정책 적용

```bash
cat <<'EOF' > deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: secure-zone
spec:
  podSelector: {}
  policyTypes:
    - Ingress
EOF

kubectl apply -f deny-all.yaml
```

재검증 — 이제 둘 다 실패(타임아웃)해야 한다. `podSelector: {}`는 네임스페이스 내 **모든** Pod를 선택하므로, 신뢰된 Pod까지 함께 차단된다. 이것이 "화이트리스트 전환"의 의미다.

```bash
kubectl exec -it allowed-client -n secure-zone -- wget -qO- --timeout=3 db-svc   # timed out
kubectl exec -it intruder -n attacker-zone -- wget -qO- --timeout=3 db-svc.secure-zone.svc.cluster.local   # timed out
```

### 4-4. 2단계 — 특정 라벨(`role=trusted`)을 가진 Pod만 허용

```bash
cat <<'EOF' > allow-trusted.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-trusted-to-db
  namespace: secure-zone
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: trusted
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl apply -f allow-trusted.yaml
```

재검증: 신뢰된 Pod는 이제 성공, 다른 네임스페이스 침입자는 여전히 실패해야 한다.

```bash
kubectl exec -it allowed-client -n secure-zone -- wget -qO- --timeout=3 db-svc                                    # 성공
kubectl exec -it intruder -n attacker-zone -- wget -qO- --timeout=3 db-svc.secure-zone.svc.cluster.local          # 실패
```

> 주의: 같은 Pod를 선택하는 여러 NetworkPolicy는 **합집합(OR)**으로 동작한다. `allow-trusted-to-db`가 `role: trusted` 트래픽을 허용하므로 그 트래픽만 통과하고, 나머지는 여전히 `deny-all-ingress`의 영향으로 차단된다.

### 4-5. 3단계 — namespaceSelector로 특정 네임스페이스 전체 허용 (심화)

```bash
kubectl label namespace attacker-zone env=quarantine --overwrite

cat <<'EOF' > allow-namespace.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-trusted-ns
  namespace: secure-zone
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: trusted-partner   # attacker-zone은 quarantine이므로 매칭 안 됨
      ports:
        - protocol: TCP
          port: 80
EOF

kubectl apply -f allow-namespace.yaml

# attacker-zone은 env=quarantine이라 매칭되지 않으므로 여전히 차단됨
kubectl exec -it intruder -n attacker-zone -- wget -qO- --timeout=3 db-svc.secure-zone.svc.cluster.local
```

### 4-6. 정리

```bash
kubectl delete namespace secure-zone attacker-zone
```

---

## 실습 5. CoreDNS 디버깅

### 5-1. 임시 디버깅 Pod로 DNS 확인

```bash
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- \
  nslookup backend-svc.net-lab.svc.cluster.local
```

정상 출력 예시(`dig`이 필요하면 `--image=nicolaka/netshoot`로 대체):

```
Server:    10.96.0.10
Name:      backend-svc.net-lab.svc.cluster.local
Address:   10.96.123.45
```

### 5-2. 짧은 이름 vs FQDN 테스트

```bash
# 같은 네임스페이스 - 짧은 이름으로 충분
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- nslookup backend-svc

# 다른 네임스페이스에서 짧은 이름 - 실패
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -n default -- nslookup backend-svc

# 다른 네임스페이스에서 <service>.<namespace> - 성공
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -n default -- nslookup backend-svc.net-lab
```

### 5-3. resolv.conf, CoreDNS 상태/로그 확인

```bash
# Pod의 nameserver가 kube-dns ClusterIP와 일치하는지
kubectl run dns-test --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- cat /etc/resolv.conf
kubectl get svc kube-dns -n kube-system

# CoreDNS 파드 상태/로그/설정
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=50
kubectl get configmap coredns -n kube-system -o yaml
```

CoreDNS 로그에 `SERVFAIL`이나 반복적인 에러가 있다면 Corefile 설정 오류나 upstream DNS 문제일 가능성이 크다.

### 5-4. DNS 자체가 안 될 때 흔한 원인 체크리스트

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns      # 1) CoreDNS Pod가 Running인지
kubectl get endpoints kube-dns -n kube-system             # 2) kube-dns Endpoints가 비어있지 않은지
kubectl get networkpolicy -A                              # 3) 정책이 DNS(53번 포트)를 막고 있지 않은지
```

> 흔한 함정: NetworkPolicy로 Egress를 제한하면서 kube-dns(53/UDP·TCP)로 나가는 트래픽을 깜빡하고 막으면, 애플리케이션은 정상인데 DNS 조회만 실패한다. Egress 정책을 걸 때는 반드시 DNS 허용 규칙을 함께 넣어야 한다.

---

## 실습 6. 의도적 장애 — 잘못된 Selector 디버깅

### 6-1. 문제 상황 재현

```bash
kubectl create deployment broken-app --image=nginx --replicas=2 -n net-lab
kubectl get pods -n net-lab -l app=broken-app --show-labels
# 실제 라벨 확인: app=broken-app (Deployment가 자동으로 붙인 라벨)

# 일부러 틀린 selector로 Service 생성
cat <<'EOF' > broken-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
  namespace: net-lab
spec:
  selector:
    app: broken-app-typo   # 오타! 실제 라벨과 다름
  ports:
    - port: 80
      targetPort: 80
EOF

kubectl apply -f broken-svc.yaml
```

### 6-2. 증상 확인

```bash
kubectl run tmp-curl --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- \
  wget -qO- --timeout=3 broken-svc
# wget: download timed out
```

### 6-3. 단계별 디버깅 절차

```bash
kubectl get svc broken-svc -n net-lab                                          # 1) Service 자체는 존재하는가?
kubectl get endpoints broken-svc -n net-lab                                    # 2) Endpoints가 채워져 있는가? ← 핵심 진단 포인트
# 출력: broken-svc   <none>   Xs   ← 비어있음! 바로 여기가 원인
kubectl get svc broken-svc -n net-lab -o jsonpath='{.spec.selector}{"\n"}'      # 3) Service의 selector 확인
# {"app":"broken-app-typo"}
kubectl get pods -n net-lab --show-labels | grep broken-app                    # 4) 실제 Pod 라벨과 비교
# app=broken-app  ← selector와 불일치 확인
```

### 6-4. 수정 및 재검증

```bash
kubectl patch svc broken-svc -n net-lab -p '{"spec":{"selector":{"app":"broken-app"}}}'
kubectl get endpoints broken-svc -n net-lab   # Endpoints가 채워지는지 확인
kubectl run tmp-curl --image=busybox:1.36 -it --rm --restart=Never -n net-lab -- wget -qO- --timeout=3 broken-svc
```

### 6-5. 추가로 자주 겪는 연결 실패 패턴

| 증상 | 확인 명령 | 원인 |
|---|---|---|
| Endpoints가 비어있음 | `kubectl get endpoints <svc>` | selector 오타, 네임스페이스 혼동 |
| Endpoints는 있지만 응답 없음 | `kubectl get pods -o wide` + Readiness 확인 | targetPort가 컨테이너 실제 포트와 불일치 |
| 특정 Pod만 응답 없음 | `kubectl describe pod <pod>` | Readiness Probe 실패로 Endpoints에서 제외됨 |
| ClusterIP는 되는데 NodePort 실패 | `kubectl get svc`, 노드 방화벽 확인 | 클라우드/방화벽에서 NodePort 범위 미개방 |
| DNS로는 안 되는데 IP로는 됨 | `nslookup`, `cat /etc/resolv.conf` | CoreDNS 장애 또는 NetworkPolicy가 53번 포트 차단 |

---

## 정리 (실습 리소스 삭제)

```bash
kubectl delete namespace net-lab
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml --ignore-not-found
```

---

## 체크리스트

- [ ] ClusterIP Service를 생성하고 클러스터 내부 Pod에서 DNS 이름으로 접속했다
- [ ] NodePort Service를 생성하고 여러 노드의 IP:포트로 동일하게 접속되는 것을 확인했다
- [ ] 경로 기반 Ingress를 작성해 `/backend`, `/frontend` 두 경로가 서로 다른 Service로 라우팅되는 것을 확인했다
- [ ] 호스트 기반 라우팅을 `Host` 헤더로 테스트했다
- [ ] Deny-all NetworkPolicy를 적용해 신뢰된 Pod까지 차단되는 것을 직접 확인했다
- [ ] podSelector로 특정 라벨만 허용하는 정책을 추가해 단계적으로 접근을 열었다
- [ ] namespaceSelector로 네임스페이스 단위 허용을 구성했다
- [ ] `nslookup`/`dig`으로 Service의 짧은 이름과 FQDN 조회 차이를 확인했다
- [ ] CoreDNS 파드 로그와 kube-dns Endpoints를 점검하는 절차를 손에 익혔다
- [ ] 잘못된 selector로 인한 연결 장애를 `kubectl get endpoints`만으로 30초 안에 진단할 수 있다

---

> 다음 단계: [CKA 05 - Storage (실습)](../05주차-Storage/CKA-05-Storage-실습.md) — PV/PVC와 StorageClass로 영속 스토리지를 다룬다.
