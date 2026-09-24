---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 08 - 모의고사 & 최종 점검 (실습)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 개념: [CKA 08 - 모의고사 & 최종 점검 (개념)](CKA-08-모의고사-최종-점검-개념.md) | 이전: [CKA 07 - 통합 실습 & 속도 훈련 (실습)](../07주차-통합-실습-속도-훈련/CKA-07-통합-실습-속도-훈련-실습.md)

> 목표: Killer.sh 모의고사 2회를 실제로 운영하고, 오답을 체계적으로 분석하며, 8주간 쌓은 실력을 최종 점검한 뒤 시험 당일까지의 타임라인을 확정한다.

---

## 0. 사전 준비

- [ ] CKA 시험 등록 완료 (등록 확인 메일에서 Killer.sh 접속 링크/코드 확인)
- [ ] 세션 시작 전 최소 반나절 이상 비는 시간대 확보 (세션당 36시간 접근 제한)
- [ ] kubernetes.io 공식 문서 북마크 정리 (자주 쓰는 페이지 즉시 접근 가능하도록)
- [ ] 실습용 alias/vim 설정을 시험 환경과 동일하게 재현 (`alias k=kubectl`, `.vimrc` 등)

---

## 1. Killer.sh 1회차 — 실전 시뮬레이션

### 진행 방법

1. Killer.sh 세션을 시작하고 **2시간 타이머**를 눈에 보이는 곳에 별도로 켜둔다.
2. 실제 시험처럼 참고 자료는 **kubernetes.io 공식 문서만** 허용한다. 블로그, 이전에 풀었던 노트 참고 금지.
3. 문제를 순서대로 풀되, **5분 이상 진전이 없으면 즉시 flag(마킹)** 하고 다음 문제로 넘어간다.
4. 시간이 남으면 flag한 문제로 돌아가 다시 시도한다.
5. 타이머 종료 후 **점수를 먼저 보지 말고**, 먼저 스스로 "막혔던 문제 목록"을 적어본다. 그다음 채점 결과와 비교한다.

### 진행 중 기록할 것

| 항목 | 기록 방법 |
|---|---|
| 막힌 문제 번호 | 문제 번호와 함께 "왜 막혔는지" 한 줄 메모 (모름/헷갈림/시간초과) |
| 소요 시간이 긴 문제 | 문제당 평균 7분을 크게 초과한 문제 번호 |
| 사용한 명령어가 낯설었던 순간 | `kubectl explain`을 몇 번 찾아봤는지 |
| 체감 난이도 | 문제 유형별로 "쉬움/보통/어려움" 표시 |

> 채점과 모범 답안은 세션 종료 후 Killer.sh에서 확인 가능하다. 정답을 확인하기 전에 반드시 위 기록을 먼저 남길 것 — 정답을 본 뒤에는 "원래 알고 있었다"는 착각이 생기기 쉽다.

---

## 2. 오답노트 작성 템플릿

1회차(그리고 2회차) 채점이 끝나면 아래 표를 그대로 채워 넣는다. 이 표가 이번 주 학습의 핵심 산출물이다.

| 문제 번호 | 도메인 | 틀린 이유 | 정답 명령어/YAML | 재복습 링크 |
|---|---|---|---|---|
| 예: Q4 | Storage | StorageClass의 `volumeBindingMode: WaitForFirstConsumer` 동작을 몰라서 PVC가 계속 Pending인 이유를 파악 못함 | `kubectl get sc <name> -o yaml`로 바인딩 모드 확인 후, Pod 스케줄 이후 바인딩되는 흐름 이해 | [CKA 05 - Storage (개념)](../05주차-Storage/CKA-05-Storage-개념.md) |
| 예: Q9 | Troubleshooting | kubelet 로그 확인 명령을 몰라서 원인 파악에 10분 이상 소요 | `journalctl -u kubelet -f` | [CKA 06 - Troubleshooting (개념)](../06주차-Troubleshooting/CKA-06-Troubleshooting-개념.md) |
| Q | | | | |
| Q | | | | |
| Q | | | | |
| Q | | | | |
| Q | | | | |

### 도메인 열에 사용할 값

`Cluster Architecture` / `Workloads & Scheduling` / `Services & Networking` / `Storage` / `Troubleshooting` 중 하나로 고정해서 적는다. 이렇게 해야 아래처럼 **도메인별 집계**가 가능하다.

| 도메인 | 오답 수 | 재복습 필요 여부 |
|---|---|---|
| Cluster Architecture, Installation & Configuration | | |
| Workloads & Scheduling | | |
| Services & Networking | | |
| Storage | | |
| Troubleshooting | | |

오답 수가 2개 이상 몰린 도메인은 **반드시 재복습 대상**으로 지정한다. 비중이 높은 도메인(Troubleshooting 30%, Architecture 25%)은 오답이 1개만 나와도 재복습을 권장한다.

---

## 3. 취약 도메인 재복습

오답노트에서 재복습이 필요하다고 표시한 도메인은 다음 순서로 다시 훑는다.

1. 해당 주차 **개념 노트**를 처음부터 다시 읽되, 이번에는 오답노트에 적어둔 "정답 명령어/YAML"과 노트 내용을 직접 대조한다.
2. 해당 주차 **실습 노트**의 명령어를 다시 손으로 타이핑해서 재현한다.
3. 재현이 끝나면 유사한 변형 문제를 스스로 만들어 풀어본다 (예: namespace를 바꾸거나, replica 수를 바꾸는 등).

| 재복습 대상 도메인 | 개념 노트 | 실습 노트 | 재복습 완료 |
|---|---|---|---|
| Cluster Architecture & Installation | [CKA 02 - Cluster Architecture & Installation (개념)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-개념.md) | [CKA 02 - Cluster Architecture & Installation (실습)](../02주차-Cluster-Architecture-Installation/CKA-02-Cluster-Architecture-Installation-실습.md) | [ ] |
| Workloads & Scheduling | [CKA 03 - Workloads & Scheduling (개념)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-개념.md) | [CKA 03 - Workloads & Scheduling (실습)](../03주차-Workloads-Scheduling/CKA-03-Workloads-Scheduling-실습.md) | [ ] |
| Services & Networking | [CKA 04 - Services & Networking (개념)](../04주차-Services-Networking/CKA-04-Services-Networking-개념.md) | [CKA 04 - Services & Networking (실습)](../04주차-Services-Networking/CKA-04-Services-Networking-실습.md) | [ ] |
| Storage | [CKA 05 - Storage (개념)](../05주차-Storage/CKA-05-Storage-개념.md) | [CKA 05 - Storage (실습)](../05주차-Storage/CKA-05-Storage-실습.md) | [ ] |
| Troubleshooting | [CKA 06 - Troubleshooting (개념)](../06주차-Troubleshooting/CKA-06-Troubleshooting-개념.md) | [CKA 06 - Troubleshooting (실습)](../06주차-Troubleshooting/CKA-06-Troubleshooting-실습.md) | [ ] |

---

## 4. Killer.sh 2회차 — 검증

재복습이 끝난 뒤에만 2회차 세션을 시작한다. 2회차의 목적은 새로운 것을 배우는 것이 아니라 **1회차에서 찾은 약점이 실제로 메워졌는지 확인**하는 것이다.

### 진행 방법

1. 1회차와 동일하게 2시간 타이머, 공식 문서만 허용, flag 활용 원칙을 그대로 적용한다.
2. 1회차 오답노트에서 다룬 도메인의 문제가 나오면 **막히지 않고 풀리는지**를 특히 주의 깊게 확인한다.
3. 종료 후 동일한 오답노트 템플릿(2절)을 다시 작성한다.
4. 1회차 대비 오답 수가 줄어든 도메인과, 여전히 반복되는 도메인을 비교한다.

| 비교 항목 | 1회차 | 2회차 |
|---|---|---|
| 총 오답 수 | | |
| 가장 취약한 도메인 | | |
| 평균 문제당 소요 시간 | | |
| flag 후 재도전 성공 수 | | |

> 2회차에서도 같은 도메인이 반복해서 발목을 잡는다면, 시험을 며칠 미루더라도 해당 도메인을 완전히 잡고 가는 것이 낫다. 자신 없는 도메인을 안고 시험장에 들어가는 것이 가장 위험한 선택이다.

---

## 5. 최종 점검 체크리스트 — 8개 주차 자가 진단

아래 표를 보며 각 주차 핵심 항목에 대해 "설명 가능" / "실습 가능" 두 기준으로 스스로 점검한다. **설명은 되는데 손이 안 움직이는 경우**, 또는 그 반대인 경우 모두 위험 신호다.

| 주차 | 핵심 점검 항목 | 설명 가능 | 실습 가능 |
|---|---|---|---|
| 1 | Control Plane/Worker Node 컴포넌트 역할, `--dry-run=client -o yaml` 활용 | [ ] | [ ] |
| 2 | etcd 백업/복구, kubeadm 클러스터 구축, RBAC 설정 | [ ] | [ ] |
| 3 | Deployment 롤링 업데이트/롤백, taint/toleration, affinity | [ ] | [ ] |
| 4 | Service(ClusterIP/NodePort) 차이, Ingress, NetworkPolicy | [ ] | [ ] |
| 5 | PV/PVC 바인딩, StorageClass 동적 프로비저닝 | [ ] | [ ] |
| 6 | CrashLoopBackOff/ImagePullBackOff/Pending 진단, kubelet 로그 분석 | [ ] | [ ] |
| 7 | 컨텍스트 전환, 시간 압박 속 문제 풀이 속도 | [ ] | [ ] |
| 8 | 오답 분석, 시험 당일 절차 숙지 | [ ] | [ ] |

체크가 비어 있는 항목은 시험 전 마지막으로 시간을 투자해야 할 우선순위다.

### 커맨드 스피드 드릴 (선택, 시간 여유 있을 때)

아래 명령어들을 **보지 않고 타이핑**해서 10분 이내에 완료할 수 있는지 확인한다.

```bash
# Pod 생성 + dry-run YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Deployment 생성 및 스케일
kubectl create deployment web --image=nginx --replicas=3
kubectl scale deployment web --replicas=5

# 이전 리비전으로 롤백
kubectl rollout undo deployment web

# etcd 스냅샷 백업
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 컨텍스트 전환
kubectl config get-contexts
kubectl config use-context <context-name>
```

---

## 6. 시험 접수부터 응시 당일까지 타임라인 예시

| 시점 | 해야 할 일 |
|---|---|
| D-7 | 7주차(통합 실습) 마무리 확인, Killer.sh 1회차 세션 시작 준비 (반나절 이상 시간 확보) |
| D-6 | Killer.sh 1회차 진행 + 오답노트 작성 |
| D-5 ~ D-4 | 오답노트 기준 취약 도메인 재복습 (해당 개념/실습 노트 재학습) |
| D-3 | Killer.sh 2회차 진행 + 오답노트 재작성, 1회차 대비 개선 여부 확인 |
| D-2 | 최종 점검 체크리스트(5절) 전체 항목 점검, 남은 취약 항목 보완 |
| D-1 | System Check(카메라/마이크/네트워크) 실행, 응시 공간 정리, 신분증 확인, 컨디션 관리(충분한 수면) — 새로운 내용 학습은 지양 |
| D-day | 시험 시작 최소 30분 전 로그인, PSI Secure Browser 실행, Room Scan 준비, 신분증 지참 확인 후 응시 |

> D-1에는 새로운 개념을 억지로 욱여넣지 않는다. 이 시점의 학습은 오히려 자신감을 흔들 수 있다. 이미 8주간 쌓은 것을 믿고 컨디션 관리에 집중할 것.

---

## 7. 커리큘럼을 마치며

여기까지 왔다면 [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md)에서 계획했던 8주 로드맵을 모두 통과한 것이다. 1주차의 Pod/Node 기본 개념부터 시작해 Architecture, Workloads, Networking, Storage, Troubleshooting을 손으로 익히고, 이번 주 모의고사로 실전 감각까지 검증했다.

> CKA는 암기 시험이 아니라 "손이 얼마나 빨리, 정확하게 움직이는가"를 보는 시험이다. 개념은 이미 머릿속에 있으니, 이제 필요한 것은 실전에서 그 개념을 침착하게 꺼내 쓰는 것뿐이다.

응시 결과와 무관하게, 이 노트들은 이후 실무에서 쿠버네티스 클러스터를 운영할 때도 계속 참고할 수 있는 자산이다. [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md)으로 돌아가 전체 로드맵을 다시 훑어보며 마무리하자.

---

## 최종 체크리스트

- [ ] Killer.sh 1회차를 2시간 타이머 걸고 실전처럼 완료했다
- [ ] 1회차 오답을 도메인별 표로 정리했다
- [ ] 오답이 몰린 도메인의 개념/실습 노트를 재복습했다
- [ ] Killer.sh 2회차를 완료하고 1회차 대비 개선을 확인했다
- [ ] 8개 주차 자가 진단 표에서 "설명 가능"과 "실습 가능"이 모두 체크되었다
- [ ] 시험 접수 정보(등록 이름, 시험 언어 설정)를 재확인했다
- [ ] D-1 System Check(카메라/마이크/네트워크)를 완료했다
- [ ] 응시 공간과 신분증을 준비했다
- [ ] 시험 당일 최소 30분 전 로그인 계획을 세웠다
