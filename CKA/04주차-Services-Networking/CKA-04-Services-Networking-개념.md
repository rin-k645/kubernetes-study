---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 04 - Services & Networking (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 04 - Services & Networking (실습)](CKA-04-Services-Networking-실습.md) | 이전: [CKA 03 - Workloads & Scheduling (개념)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-개념.md) | 다음: [CKA 05 - Storage (개념)](../05주차-Storage/CKA-05-Storage-개념.md)

---

## 왜 Service가 필요한가?

Pod는 언제든지 죽었다 다시 뜰 수 있고, 그때마다 새로운 IP를 받는다. 클라이언트가 Pod IP를 직접 하드코딩하면 재배포될 때마다 연결이 끊어진다. Service는 Label Selector로 대상 Pod 집합을 추적하는 **고정된 가상 IP + 로드밸런서**를 제공해 이 문제를 해결한다.

```
[Client] ---> [Service: my-svc (ClusterIP: 10.96.0.10, 고정)]
                        |
              +---------+---------+
              |         |         |
           Pod A     Pod B     Pod C   (재시작돼도 Service IP는 불변)
```

> 한 줄 요약: "Service는 변화무쌍한 Pod들 앞에 놓인 변하지 않는 안정적인 진입점(고정 IP + DNS 이름)이다."

---

## Service 종류별 개념

### 1. ClusterIP (기본값)

클러스터 **내부에서만** 접근 가능한 가상 IP를 할당한다.

```
[Pod (같은 클러스터)] ---> [ClusterIP Service] ---> [대상 Pod들]
```

- 클러스터 내부 서비스 간 통신(예: 백엔드 ↔ DB)에 사용
- 외부에서는 절대 접근 불가
- 별도 지정이 없으면 `type: ClusterIP`가 기본값

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80          # Service 자신의 포트
      targetPort: 8080  # Pod 컨테이너 포트
```

### 2. NodePort

ClusterIP를 포함하면서, **모든 노드의 특정 포트(30000~32767)**를 열어 외부에서 접근 가능하게 한다. 어떤 노드로 요청을 보내도 kube-proxy가 내부적으로 올바른 Pod까지 라우팅해준다. 테스트/온프레미스 환경에서 간단히 외부 노출할 때 쓰지만, 실무에서는 보안/포트 관리 문제로 LoadBalancer나 Ingress를 더 선호한다.

```
[외부 클라이언트] --> [Node1:30080 | Node2:30080 | Node3:30080] --> [ClusterIP] --> [대상 Pod들]
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080   # 생략 시 자동 할당
```

### 3. LoadBalancer

클라우드 제공자(AWS, GCP, Azure 등)의 **외부 로드밸런서를 자동으로 프로비저닝**한다. NodePort + ClusterIP를 모두 포함하는 상위 개념이다 (`인터넷 → 클라우드 LB → NodePort → ClusterIP → Pod`). 온프레미스/베어메탈 클러스터에서는 클라우드 LB가 없어 `EXTERNAL-IP`가 `<pending>`으로 남는 경우가 많다(MetalLB 같은 별도 구현체 필요). CKA 시험 환경은 대부분 베어메탈이므로 LoadBalancer보다 NodePort/Ingress 실습이 더 중요하다.

### 4. ExternalName

Selector나 Endpoints 없이, Service DNS 이름을 **외부 DNS 이름으로 CNAME 리다이렉트**한다. 클러스터 외부의 레거시 DB, 외부 API 등을 클러스터 내부 이름으로 추상화할 때 사용하며, kube-proxy를 거치지 않고 순수 DNS 레벨에서만 동작한다(프록시/로드밸런싱 없음).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

### Service 종류 비교

| 종류 | 접근 범위 | 포트 범위 | 주요 사용 사례 |
|---|---|---|---|
| ClusterIP | 클러스터 내부 | 임의 | 내부 서비스 간 통신 |
| NodePort | 클러스터 외부 (노드 IP 경유) | 30000~32767 | 간단한 외부 노출, 테스트 |
| LoadBalancer | 클러스터 외부 (클라우드 LB) | 임의 (LB가 매핑) | 클라우드 환경의 프로덕션 외부 노출 |
| ExternalName | DNS 레벨 리다이렉트 | 해당 없음 | 외부 리소스를 내부 이름으로 추상화 |

---

## kube-proxy의 동작 원리

kube-proxy는 **모든 노드에서 DaemonSet으로 실행**되며, API Server에서 Service/Endpoints 변경을 watch해 Service의 가상 IP를 실제 Pod IP로 변환하는 규칙을 노드의 네트워크 스택(iptables 또는 IPVS)에 심는다.

**iptables 모드(기본값)**: Service마다 iptables 체인을 생성하고, ClusterIP로 들어온 트래픽을 **확률적으로(random)** 하나의 Pod IP로 DNAT한다. 규칙을 순차 매칭하므로 Service/Pod 수가 많아지면 오버헤드가 커진다(O(n)).

```
ClusterIP:80 요청 → KUBE-SERVICES 체인 → KUBE-SVC-XXXX 체인 → 확률적 분산(KUBE-SEP-AAA/BBB/CCC) → DNAT
```

**IPVS 모드(대규모 클러스터 권장)**: 커널의 IPVS(IP Virtual Server)를 이용해 해시 테이블 기반으로 조회하므로 Service 수가 많아도 일정한 성능(O(1))을 유지하며, Round Robin·Least Connection 등 다양한 알고리즘을 지원한다. 노드에 `ip_vs` 커널 모듈이 필요하다.

| 항목 | iptables | IPVS |
|---|---|---|
| 매칭 방식 | 순차 규칙 매칭 | 해시 테이블 |
| 대규모 클러스터 성능 | Service 많아지면 느려짐 | 일정한 성능 유지 |
| 로드밸런싱 알고리즘 | 랜덤(확률 기반) | RR, LC, DH 등 다양 |
| 기본값 여부 | 기본 | 명시적 설정 필요 |

> CKA 시험에서는 kube-proxy 모드를 직접 바꾸는 문제보다는 "kube-proxy가 없으면 Service가 동작하지 않는다"는 원리를 이해하는 것이 더 중요하다.

---

## Endpoints / EndpointSlice — Service가 Pod를 찾는 방법

Service는 Selector로 대상 Pod를 지정하지만, 실제로 kube-proxy가 참조하는 것은 **Endpoints(또는 EndpointSlice)** 객체다. Endpoints Controller가 Selector에 매칭되는 **Ready 상태의 Pod IP 목록**을 지속적으로 동기화하며, Readiness Probe를 통과하지 못한 Pod는 목록에서 제외된다(트래픽을 받지 않음). EndpointSlice는 대규모 클러스터에서 단일 객체 크기 제한 문제를 피하기 위해 여러 조각으로 분산한 확장판이다. Selector가 없는 Service(ExternalName 등)는 Endpoints를 수동으로 만들어줘야 한다.

```
kubectl get endpoints my-svc
kubectl get endpointslices -l kubernetes.io/service-name=my-svc
```

> 디버깅 핵심: Service에 연결이 안 될 때 가장 먼저 확인할 것은 `kubectl get endpoints <svc>`가 비어있는지 여부다. 비어있다면 Selector가 Pod의 Label과 일치하지 않거나, Pod가 Not Ready 상태다.

---

## CNI와 Pod 네트워크의 원리 (개괄)

Kubernetes 자체는 네트워크를 구현하지 않는다. **CNI(Container Network Interface)** 플러그인(Calico, Flannel, Cilium, Weave 등)이 실제 Pod 네트워킹을 담당하며, 다음 원칙을 만족해야 한다.

1. 모든 Pod는 NAT 없이 서로 통신할 수 있어야 한다 (같은 노드든 다른 노드든)
2. 모든 노드는 NAT 없이 모든 Pod와 통신할 수 있어야 한다
3. Pod가 자신을 바라보는 IP와 다른 개체가 그 Pod를 바라보는 IP가 같아야 한다

### 오버레이 네트워크 개념 (예: Flannel VXLAN)

노드 간 물리 네트워크 위에 **가상의 터널**을 만들어, 서로 다른 노드의 Pod들이 마치 같은 네트워크에 있는 것처럼 통신하게 한다. Pod A → Pod B로 보낸 패킷은 출발 노드의 flannel이 VXLAN으로 캡슐화해 물리 네트워크로 전송하고, 도착 노드의 flannel이 캡슐을 벗겨(decapsulate) Pod B에게 전달한다.

```
Node1(Pod CIDR 10.244.1.0/24) --VXLAN 캡슐화--> 물리망 --탈캡슐화--> Node2(Pod CIDR 10.244.2.0/24)
```

각 노드에는 서로 다른 Pod CIDR 대역이 할당된다.

```
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'
```

> CKA 시험 범위에서 CNI 플러그인의 상세 구현은 다루지 않지만, "왜 노드마다 다른 Pod CIDR을 갖는지", "CNI 플러그인이 없으면 Pod가 Pending/ContainerCreating에서 멈춘다"는 사실은 자주 등장한다.

---

## CoreDNS — 클러스터 내부 DNS

CoreDNS는 `kube-system` 네임스페이스에서 Deployment로 실행되며, `kube-dns`라는 Service로 노출된다. 모든 Pod의 `/etc/resolv.conf`는 기본적으로 이 CoreDNS를 네임서버(`nameserver 10.96.0.10`)로 지정하고, `search default.svc.cluster.local svc.cluster.local cluster.local` 같은 search 도메인을 갖는다.

### DNS 이름 규칙

| 대상 | 형식 | 예시 |
|---|---|---|
| Service (동일 네임스페이스) | `<service>` | `my-svc` |
| Service (다른 네임스페이스) | `<service>.<namespace>` | `my-svc.prod` |
| Service (FQDN) | `<service>.<namespace>.svc.cluster.local` | `my-svc.prod.svc.cluster.local` |
| Pod (FQDN, IP 기반) | `<pod-ip-dash>.<namespace>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| Headless Service의 Pod | `<pod-hostname>.<service>.<namespace>.svc.cluster.local` | `web-0.web-svc.default.svc.cluster.local` |

`my-svc.prod` 요청 → CoreDNS가 `my-svc.prod.svc.cluster.local`로 확장해 조회 → ClusterIP 반환 → kube-proxy 규칙으로 실제 Pod까지 라우팅되는 흐름이다. `cluster.local`은 클러스터의 기본 도메인(kubeadm 설치 시 기본값, 변경 가능)이며, Pod 단위 DNS는 StatefulSet의 Headless Service처럼 명시적으로 필요할 때만 사용한다.

---

## Ingress와 Ingress Controller

### 개념 차이 — 이 둘을 혼동하는 것이 가장 흔한 실수

```
Ingress 리소스 (YAML)          =  "라우팅 규칙 정의서" (선언적 설정)
Ingress Controller (Pod/Deployment) = "그 규칙을 실제로 실행하는 프록시 서버" (nginx, traefik 등)
```

- **Ingress**: 호스트/경로 기반 라우팅 규칙을 선언하는 Kubernetes 리소스. 그 자체로는 아무 동작도 하지 않는다.
- **Ingress Controller**: 클러스터에 별도로 설치해야 하는 실제 리버스 프록시(예: ingress-nginx, Traefik, HAProxy). Ingress 리소스를 감시(watch)하여 자신의 설정(nginx.conf 등)에 반영한다.
- Kubernetes는 기본적으로 Ingress Controller를 내장하지 않는다 — 반드시 별도 설치 필요 (CKA 시험 환경에는 보통 미리 설치되어 있음)

```
[외부 클라이언트]
      |
[Ingress Controller (nginx Pod, Service로 노출)]
      |
      | (Ingress 리소스의 규칙을 읽어서 라우팅 결정)
      |
   +--+-------------------+
   |                      |
경로: /api            경로: /web
   |                      |
[api-svc (ClusterIP)]  [web-svc (ClusterIP)]
```

### 경로 기반 / 호스트 기반 라우팅

경로 기반은 같은 호스트에서 URL 경로(`/api`, `/web`)로 백엔드를 분기하고, 호스트 기반은 `host` 필드(`api.example.com`, `web.example.com`)로 요청 도메인 자체를 분기한다. 하나의 Ingress 리소스 안에서 두 방식을 혼용할 수도 있다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com       # 호스트 기반: 이 도메인으로 온 요청만 매칭
      http:
        paths:
          - path: /api            # 경로 기반: /api 접두사만 매칭
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
```

- `pathType`은 `Exact`(정확히 일치), `Prefix`(경로 접두사 일치), `ImplementationSpecific`(컨트롤러에 위임) 중 선택
- `ingressClassName` 필드로 여러 Ingress Controller가 있을 때 어떤 것을 쓸지 지정 가능

---

## NetworkPolicy — Pod 간 방화벽

### 기본 원칙 — Kubernetes는 기본적으로 "모두 허용"이다

NetworkPolicy가 하나도 없으면 모든 Pod는 클러스터 내 다른 모든 Pod와 자유롭게 통신 가능하다(flat network). CNI 플러그인이 NetworkPolicy를 지원해야만(Calico, Cilium 등 — Flannel 단독으로는 미지원) 정책이 실제로 적용되며, 지원하지 않는 CNI에서는 정책을 만들어도 효과가 없다.

정책이 하나라도 특정 Pod를 선택하는 순간, 그 Pod는 "화이트리스트" 모드로 전환되어 명시된 트래픽만 허용되고 나머지는 거부된다(Default Deny). Ingress 정책만 있으면 Egress는 여전히 전체 허용이며(반대도 마찬가지) 두 방향은 독립적으로 취급된다. 같은 Pod를 선택하는 여러 NetworkPolicy는 규칙이 **합집합(OR)**으로 적용된다.

### 선택자(Selector) 3가지

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy
  namespace: prod
spec:
  podSelector:              # 이 정책이 적용될 대상 Pod
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:            # ① 같은 네임스페이스 내 특정 라벨의 Pod만 허용
            matchLabels:
              app: frontend
        - namespaceSelector:      # ② 특정 라벨의 네임스페이스 전체를 허용
            matchLabels:
              env: trusted
        - ipBlock:                # ③ 특정 CIDR 대역(주로 클러스터 외부 IP)을 허용
            cidr: 10.0.0.0/24
            except:
              - 10.0.0.5/32
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

| 선택자 | 의미 | 주의점 |
|---|---|---|
| `podSelector` | 같은 네임스페이스 내 라벨이 일치하는 Pod | 네임스페이스를 넘지 못함 |
| `namespaceSelector` | 라벨이 일치하는 네임스페이스에 속한 **모든** Pod | 네임스페이스 자체에 라벨이 있어야 함 (`kubectl label ns`) |
| `ipBlock` | CIDR 대역 (클러스터 외부 트래픽 등에 사용) | Pod IP 기반이라 Pod가 재시작되면 IP가 바뀌어 무효화될 수 있음 |
| `podSelector` + `namespaceSelector` 동시 지정 (같은 `from` 항목 안) | AND 조건: 특정 네임스페이스의 특정 라벨 Pod만 | `from` 리스트의 별도 항목이면 OR, 같은 항목의 필드면 AND |

### Deny-all 패턴 (자주 나오는 시험 문제 유형)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: prod
spec:
  podSelector: {}       # 빈 셀렉터 = 네임스페이스 내 모든 Pod
  policyTypes:
    - Ingress
  # ingress 필드 자체가 없으면 = 아무것도 허용하지 않음
```

`podSelector: {}`는 "네임스페이스 내 모든 Pod"를 의미하고, `ingress` 필드 생략(또는 `[]`)은 "명시된 규칙이 없으므로 전부 차단"을 의미한다. 이후 별도 NetworkPolicy로 특정 트래픽만 추가 허용하며 점진적으로 좁혀나간다 (실습 노트에서 단계별로 다룸).

---

## 핵심 용어 요약

| 용어 | 설명 |
|---|---|
| Service | 동적인 Pod 집합에 고정된 가상 IP와 DNS 이름을 제공하는 리소스 |
| ClusterIP | 클러스터 내부 전용 가상 IP (기본 타입) |
| NodePort | 모든 노드의 특정 포트를 열어 외부 노출 |
| LoadBalancer | 클라우드 외부 로드밸런서 자동 프로비저닝 |
| ExternalName | 클러스터 외부 DNS 이름으로 CNAME 리다이렉트 |
| kube-proxy | 각 노드에서 Service ↔ Pod 간 트래픽 라우팅 규칙(iptables/IPVS)을 관리하는 컴포넌트 |
| Endpoints / EndpointSlice | Service Selector에 매칭되는 실제 Pod IP 목록 |
| CNI | Pod 네트워크(노드 간 통신)를 구현하는 플러그인 표준 (Calico, Flannel 등) |
| CoreDNS | 클러스터 내부 DNS 서버, Service/Pod 이름을 IP로 변환 |
| Ingress | 호스트/경로 기반 HTTP(S) 라우팅 규칙을 선언하는 리소스 |
| Ingress Controller | Ingress 규칙을 실제로 실행하는 리버스 프록시 (별도 설치 필요) |
| NetworkPolicy | Pod 간 트래픽을 제어하는 방화벽 규칙 (CNI 지원 필요) |
| podSelector | NetworkPolicy에서 라벨 기반으로 Pod를 지정하는 선택자 |
| namespaceSelector | NetworkPolicy에서 라벨 기반으로 네임스페이스를 지정하는 선택자 |
| ipBlock | NetworkPolicy에서 CIDR 대역을 지정하는 선택자 |

---

## 흔한 실수

- **실수 1**: "Ingress를 만들면 바로 외부에서 접근된다"
  - 정정: Ingress Controller가 클러스터에 설치되어 있지 않으면 Ingress 리소스는 아무 동작도 하지 않는다. `kubectl get pods -n ingress-nginx` 등으로 컨트롤러 존재 여부를 먼저 확인해야 한다.

- **실수 2**: "Service의 selector와 Pod의 label 이름만 비슷하면 연결된다"
  - 정정: Selector는 **정확히 일치**해야 한다. `app: web`과 `app: Web`은 다르며, 오타 하나로 Endpoints가 비어버린다. 항상 `kubectl get endpoints <svc>`로 검증할 것.

- **실수 3**: "NetworkPolicy를 하나 만들면 다른 트래픽은 이전 규칙대로 동작한다"
  - 정정: 특정 Pod를 선택하는 NetworkPolicy가 하나라도 생기면, 그 Pod에 대해 명시되지 않은 모든 트래픽은 즉시 차단된다(Default Deny 전환). "허용 정책 추가"가 아니라 "허용 목록 자체를 정의"하는 개념이다.

- **실수 4**: "NetworkPolicy의 Egress를 막으면 Ingress도 막힌다"
  - 정정: `policyTypes`의 Ingress와 Egress는 완전히 독립적이다. Egress만 지정한 정책은 Ingress 트래픽에 영향을 주지 않는다.

- **실수 5**: "NodePort 포트 번호를 아무 숫자나 지정할 수 있다"
  - 정정: NodePort는 기본적으로 30000~32767 범위 내에서만 지정 가능하다 (API Server의 `--service-node-port-range`로 조정 가능하지만 시험 환경에서는 기본값 사용).

---

## 빠른 복습

- Service는 변화하는 Pod 집합에 고정된 IP/DNS를 제공하며, ClusterIP(내부) / NodePort(노드 경유 외부) / LoadBalancer(클라우드 LB) / ExternalName(DNS 리다이렉트) 4종류가 있다.
- kube-proxy는 모든 노드에서 실행되며 iptables(기본, 확률적 분산) 또는 IPVS(해시 기반, 대규모에 유리) 규칙으로 Service IP를 실제 Pod IP로 변환한다.
- Service는 Selector로 Pod를 찾지 않고, Endpoints/EndpointSlice에 동기화된 Ready 상태 Pod 목록을 참조한다 — 연결 문제의 첫 확인 지점.
- CNI 플러그인이 오버레이 네트워크(예: VXLAN)로 서로 다른 노드의 Pod 간 통신을 가능하게 한다. 노드마다 별도의 Pod CIDR을 갖는다.
- CoreDNS는 `<service>.<namespace>.svc.cluster.local` 규칙으로 Service 이름을 ClusterIP로 변환한다.
- Ingress(라우팅 규칙 선언)와 Ingress Controller(실제 프록시 구현체)는 별개이며, 컨트롤러가 없으면 Ingress는 동작하지 않는다.
- Kubernetes는 기본적으로 모든 Pod 간 통신을 허용하며, NetworkPolicy가 Pod를 선택하는 순간부터 명시된 규칙 외 모든 트래픽이 차단되는 화이트리스트 모드로 전환된다.

---

> 다음 단계: [CKA 04 - Services & Networking (실습)](CKA-04-Services-Networking-실습.md) — Service, Ingress, NetworkPolicy를 직접 만들고 연결 장애를 디버깅해본다.
