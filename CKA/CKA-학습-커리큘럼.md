---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 학습 커리큘럼

> Certified Kubernetes Administrator(CKA) 취득을 목표로 하는 8주 완전 학습 로드맵. CKA는 100% 실기(터미널 기반) 시험이므로, 개념 노트와 실습 노트를 분리하여 "이해 → 손으로 익히기"를 반복한다.

## 학습 목표

이 커리큘럼을 완료하면 다음을 할 수 있습니다.

- Kubernetes 클러스터의 아키텍처와 핵심 컴포넌트의 역할을 설명할 수 있다
- kubeadm으로 클러스터를 직접 구축, 업그레이드, 백업/복구할 수 있다
- Deployment, Service, Ingress, NetworkPolicy 등을 조합해 애플리케이션을 배포할 수 있다
- PV/PVC/StorageClass를 이용한 스토리지 구성을 할 수 있다
- 장애 상황(노드/파드/네트워크)을 빠르게 진단하고 복구할 수 있다
- 실전 시간 제한(문제당 5~10분) 안에 kubectl 명령으로 문제를 해결할 수 있다

---

## 전제 조건 (Prerequisites)

- Linux CLI 기본 사용법 (vim, systemctl, journalctl)
- 컨테이너/Docker 기초 개념
- YAML 문법에 대한 기본 이해
- 네트워크 기초 (IP, 포트, DNS)

---

## 시험 도메인 비중

| 영역 | 비중 | 대응 주차 |
|---|---|---|
| Troubleshooting | 30% | 6주차 |
| Cluster Architecture, Installation & Configuration | 25% | 2주차 |
| Services & Networking | 20% | 4주차 |
| Workloads & Scheduling | 15% | 3주차 |
| Storage | 10% | 5주차 |

→ 학습 시간 배분도 이 비중을 따른다. Troubleshooting과 Cluster Architecture에 가장 많은 시간을 투자할 것.

---

## 커리큘럼 로드맵

```
[기초]                [핵심 도메인]                      [마무리]
  |                        |                                |
1주차                 2~6주차                          7~8주차
개념/환경세팅         Architecture, Workloads,        통합 실습
                      Networking, Storage,            모의고사
                      Troubleshooting
```

각 주차는 **개념(概念) 노트**와 **실습(實習) 노트**로 분리되어 있다. 반드시 개념 노트를 먼저 읽고, 같은 날 실습 노트를 따라 손으로 실행할 것.

---

## 1단계: 기초 다지기 — 예상 시간: 1주

### 목표
컨테이너/쿠버네티스 핵심 개념을 이해하고, 실습용 멀티노드 클러스터를 직접 구축한다.

| 주차 | 주제 | 개념 노트 | 실습 노트 | 예상 시간 |
|---|---|---|---|---|
| 1 | 기초 개념 & 환경 세팅 | [CKA 01 - 기초 개념 & 환경 세팅 (개념)](01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-개념.md) | [CKA 01 - 기초 개념 & 환경 세팅 (실습)](01주차-기초-개념-환경-세팅/CKA-01-기초-개념-환경-세팅-실습.md) | 8시간 |

### 체크리스트
- [ ] Pod, Node, Cluster, Control Plane의 역할을 설명할 수 있다
- [ ] API Server, etcd, Scheduler, Controller Manager, kubelet, kube-proxy의 관계를 그림으로 그릴 수 있다
- [ ] kubeadm으로 1 control-plane + 2 worker 클러스터를 직접 구축했다
- [ ] `kubectl --dry-run=client -o yaml`로 리소스 YAML을 빠르게 생성할 수 있다

---

## 2단계: 시험 핵심 도메인 — 예상 시간: 5주

### 목표
시험 5대 도메인을 순서대로 정복한다. 비중이 높은 영역(Architecture, Troubleshooting)에 더 많은 시간을 배분한다.

| 주차 | 도메인 | 비중 | 개념 노트 | 실습 노트 | 예상 시간 |
|---|---|---|---|---|---|
| 2 | Cluster Architecture, Installation & Configuration | 25% | [CKA 02 - Cluster Architecture & Installation (개념)](02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-개념.md) | [CKA 02 - Cluster Architecture & Installation (실습)](02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md) | 10시간 |
| 3 | Workloads & Scheduling | 15% | [CKA 03 - Workloads & Scheduling (개념)](03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-개념.md) | [CKA 03 - Workloads & Scheduling (실습)](03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-실습.md) | 8시간 |
| 4 | Services & Networking | 20% | [CKA 04 - Services & Networking (개념)](04주차-Services-Networking/CKA-04-Services-Networking-개념.md) | [CKA 04 - Services & Networking (실습)](04주차-Services-Networking/CKA-04-Services-Networking-실습.md) | 9시간 |
| 5 | Storage | 10% | [CKA 05 - Storage (개념)](05주차-Storage/CKA-05-Storage-개념.md) | [CKA 05 - Storage (실습)](05주차-Storage/CKA-05-Storage-실습.md) | 6시간 |
| 6 | Troubleshooting | 30% | [CKA 06 - Troubleshooting (개념)](06주차-Troubleshooting/CKA-06-Troubleshooting-개념.md) | [CKA 06 - Troubleshooting (실습)](06주차-Troubleshooting/CKA-06-Troubleshooting-실습.md) | 12시간 |

### 체크리스트
- [ ] etcd 스냅샷 백업/복구를 5분 안에 수행할 수 있다
- [ ] RBAC(Role/ClusterRole/RoleBinding)를 이용해 접근 권한을 제한할 수 있다
- [ ] taint/toleration, affinity로 파드 배치를 제어할 수 있다
- [ ] Service(ClusterIP/NodePort), Ingress, NetworkPolicy를 조합할 수 있다
- [ ] PVC를 동적 프로비저닝으로 생성하고 파드에 마운트할 수 있다
- [ ] CrashLoopBackOff, ImagePullBackOff, Pending 등 장애를 원인별로 진단할 수 있다
- [ ] kubelet/API Server 장애를 로그로 추적하고 복구할 수 있다

---

## 3단계: 실전 감각 익히기 — 예상 시간: 2주

### 목표
지금까지 배운 내용을 통합하고, 시험과 동일한 시간 압박 속에서 문제를 푸는 감각을 기른다.

| 주차 | 주제 | 개념 노트 | 실습 노트 | 예상 시간 |
|---|---|---|---|---|
| 7 | 통합 실습 & 속도 훈련 | [CKA 07 - 통합 실습 & 속도 훈련 (개념)](07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-개념.md) | [CKA 07 - 통합 실습 & 속도 훈련 (실습)](07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-실습.md) | 8시간 |
| 8 | 모의고사 & 최종 점검 | [CKA 08 - 모의고사 & 최종 점검 (개념)](08주차-모의고사-최종-점검/CKA-08-모의고사-최종-점검-개념.md) | [CKA 08 - 모의고사 & 최종 점검 (실습)](08주차-모의고사-최종-점검/CKA-08-모의고사-최종-점검-실습.md) | 8시간 |

### 체크리스트
- [ ] kubectl alias(`k`), vim 설정 등 나만의 시험 환경 세팅을 준비했다
- [ ] kubernetes.io 공식 문서에서 필요한 예제를 3분 안에 찾을 수 있다
- [ ] Killer.sh 모의고사를 2회 모두 풀고 오답을 분석했다
- [ ] 17문제/2시간 페이스(문제당 평균 7분)에 익숙하다

---

## 추천 학습 자료

### 공식 문서 (시험 중 유일하게 열람 가능 — 반드시 익숙해질 것)
- [Kubernetes 공식 문서](https://kubernetes.io/docs/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

### 실습 플랫폼
- Killercoda (무료 온라인 쿠버네티스 실습 환경)
- Killer.sh (CKA 등록 시 무료 모의고사 2회 제공, 실제 시험과 가장 유사)
- KodeKloud CKA 코스 (실습 랩 포함)

### 커뮤니티 / 공식 정보
- [CKA 시험 커리큘럼(CNCF 공식 GitHub)](https://github.com/cncf/curriculum)
- Stack Overflow `kubernetes` 태그

---

## 학습 팁

1. **명령어를 외우지 말고 손에 익혀라**: 같은 리소스를 생성/삭제/수정하는 연습을 반복해 반사적으로 나오게 만들 것.
2. **`kubectl explain`을 활용하라**: YAML 필드가 기억나지 않을 때 `kubectl explain <resource>.<field>`로 즉시 확인 가능하다.
3. **imperative command + `--dry-run=client -o yaml`**: 시험에서 YAML을 처음부터 타이핑하지 말고 이 조합으로 뼈대를 생성한 뒤 수정하라.
4. **네임스페이스를 항상 확인하라**: 문제에서 지정한 네임스페이스를 빠뜨리는 실수가 가장 흔하다.
5. **컨텍스트 전환에 익숙해져라**: 시험은 여러 클러스터/컨텍스트를 오가며 진행된다. `kubectl config use-context`를 몸에 익힐 것.
6. **막히면 넘어가라**: 시험 중 한 문제에 매몰되지 말고, 확실한 문제부터 풀고 마킹 후 되돌아오라.
