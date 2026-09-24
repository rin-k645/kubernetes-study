---
tags:
  - kubernetes
  - cka
  - study
created: 2026-09-24
---

# CKA 05 - Storage (개념)

> 커리큘럼: [CKA 학습 커리큘럼](../CKA-학습-커리큘럼.md) | 실습: [CKA 05 - Storage (실습)](CKA-05-Storage-실습.md) | 이전: [CKA 04 - Services & Networking (개념)](../04주차-Services-Networking/CKA-04-Services-Networking-개념.md) | 다음: [CKA 06 - Troubleshooting (개념)](../06주차-Troubleshooting/CKA-06-Troubleshooting-개념.md)

---

## Storage가 필요한 이유

### 문제 상황 — 컨테이너는 원래 상태를 저장하지 않는다

컨테이너의 파일시스템은 컨테이너가 죽으면 함께 사라진다. Pod가 재시작되거나 다른 노드로 다시 스케줄링되면 그 안에 쌓아둔 데이터는 전부 날아간다.

```
[Pod 실행] --쓰기--> [컨테이너 파일시스템]
[Pod 재시작/삭제] --> 컨테이너 파일시스템도 함께 삭제됨 (데이터 유실)
```

- DB, 로그, 업로드 파일처럼 **살아남아야 하는 데이터**를 다뤄야 하는 경우가 많음
- 컨테이너 자체 저장소만으로는 상태 저장(stateful) 애플리케이션을 운영할 수 없음

### 해결책 — Volume을 Pod에 연결한다

```
[Pod]
 ├── Container (파일시스템은 휘발성)
 └── Volume (Pod 스펙에 정의, 컨테이너 재시작에도 유지)
```

Kubernetes의 Volume은 Pod 안의 컨테이너들이 공유할 수 있는 디렉터리이며, 어떤 매체를 쓰느냐에 따라 데이터의 생명주기가 달라진다.

---

## 1. Volume과 Pod 생명주기

Volume은 Pod의 스펙(`spec.volumes`)에 정의되고, 컨테이너는 `volumeMounts`로 자신의 파일시스템 경로에 마운트해서 사용한다. **Volume의 생명주기는 볼륨 종류에 따라 다르다.**

```
Pod 생명주기:     [생성] -----------------> [실행 중] -----------------> [삭제]
emptyDir:         [생성] --------유지-------> ...        --------------> [함께 삭제]
hostPath:         (Pod 생성 전부터 노드에 존재) --------------------------> [Pod 삭제돼도 유지]
PVC(PV):          (Pod와 무관하게 독립적으로 존재) ----------------------> [PVC/PV 정책에 따라 결정]
```

### emptyDir

Pod가 생성될 때 **빈 디렉터리**로 만들어지고, 같은 Pod 안의 모든 컨테이너가 공유한다.

```yaml
volumes:
  - name: cache-vol
    emptyDir: {}
```

- 생명주기 = Pod의 생명주기. **Pod가 삭제되면 emptyDir도 함께 사라진다** (컨테이너 재시작만으로는 사라지지 않음)
- 기본적으로 노드의 디스크에 저장됨. `emptyDir.medium: Memory`로 지정하면 tmpfs(메모리)에 저장 가능
- 용도: 같은 Pod 내 컨테이너 간 파일 공유, 임시 캐시, sidecar 패턴

### hostPath

**워커 노드의 파일시스템 경로**를 그대로 Pod에 마운트한다.

```yaml
volumes:
  - name: host-vol
    hostPath:
      path: /data/mydata
      type: DirectoryOrCreate
```

- 생명주기 = 노드의 파일시스템 생명주기. Pod가 삭제돼도 노드에 데이터가 남는다
- 단점: Pod가 **다른 노드로 재스케줄링**되면 그 노드에는 해당 데이터가 없음 → 상태 저장 워크로드에 부적합
- 주로 노드 자체의 로그/모니터링 데이터 수집(DaemonSet), 또는 Docker 소켓 마운트 같은 특수 목적에 사용
- `type` 필드로 대상이 디렉터리인지, 파일인지, 자동 생성할지 등을 검증 가능 (`Directory`, `DirectoryOrCreate`, `File`, `FileOrCreate` 등)

### 용도별 정리

| 볼륨 타입 | 생명주기 | 노드 이동 시 데이터 | 주요 용도 |
|---|---|---|---|
| emptyDir | Pod와 동일 | 유지 안 됨(애초에 Pod 삭제 시 사라짐) | 임시 캐시, 컨테이너 간 공유 |
| hostPath | 노드와 동일 | 유지 안 됨 (다른 노드엔 없음) | 노드 로그 수집, 특수 목적 |
| PV/PVC | 독립적 | 유지됨 (네트워크 스토리지 기반이면) | 실제 영구 데이터, DB 등 |

> emptyDir와 hostPath는 모두 **특정 노드에 종속**된다는 한계가 있다. 진짜 "영구 저장"을 하려면 PV/PVC가 필요하다.

---

## 2. PersistentVolume(PV)과 PersistentVolumeClaim(PVC)

### 왜 PV/PVC로 나누는가

Pod 스펙에 직접 `nfs`, `hostPath` 등 스토리지 상세 정보를 적으면, 개발자가 인프라(어떤 스토리지 서버인지, 어떤 디스크인지)까지 알아야 한다. Kubernetes는 이를 **관리자(PV)와 사용자(PVC)의 역할을 분리**해서 해결한다.

```
[관리자 영역]                          [개발자 영역]
PersistentVolume (PV)                 PersistentVolumeClaim (PVC)
"실제 스토리지 조각"                    "이런 스토리지가 필요해요" (요청서)
  - 용량: 10Gi                           - 용량: 10Gi 이상
  - AccessMode: RWO                      - AccessMode: RWO
  - NFS / iSCSI / CSI 드라이버 등          - StorageClass (선택)
        \                                     /
         \___________ 바인딩(Binding) _______/
                      |
                    [Pod]
              volumes.persistentVolumeClaim으로 참조
```

- **PV**: 클러스터의 스토리지 자원을 표현하는 오브젝트. 네임스페이스에 속하지 않는 **클러스터 스코프** 리소스
- **PVC**: 사용자가 특정 스펙(용량, 접근 모드 등)의 스토리지를 요청하는 오브젝트. **네임스페이스에 속함**
- Pod는 PV를 직접 참조하지 않고, 항상 PVC를 통해서만 스토리지에 접근한다

### 바인딩(Binding) 원리

PVC가 생성되면 컨트롤 플레인이 조건에 맞는 PV를 찾아 **1:1로 바인딩**한다.

```
PVC 요청: capacity >= 5Gi, accessMode = RWO, storageClassName = "manual"
                    |
         [바인딩 컨트롤러가 후보 탐색]
                    |
   PV 목록 중 조건을 만족하는 가장 적합한 PV 선택
   (capacity >= 요청량, accessMode 일치, storageClassName 일치)
                    |
              PV.status.phase = Bound
              PVC.status.phase = Bound
```

바인딩 조건:

1. **StorageClass 이름이 일치**해야 한다 (둘 다 비워두면 기본 StorageClass 또는 빈 값끼리 매칭)
2. PV의 **capacity가 PVC의 요청량 이상**이어야 한다 (초과해도 바인딩 가능하지만 남는 용량은 못 씀)
3. PVC가 요청한 **accessMode를 PV가 지원**해야 한다
4. `selector`(라벨 셀렉터)를 PVC에 지정한 경우 PV의 라벨과도 일치해야 함

일단 바인딩되면 **1:1 전용 관계**가 되어, 다른 PVC가 그 PV를 가로챌 수 없다. PVC를 삭제하기 전까지는 계속 유지된다.

### 상태(phase) 값

| PV Phase | 의미 | PVC Phase | 의미 |
|---|---|---|---|
| Available | 아직 바인딩 안 됨, 사용 가능 | Pending | 조건에 맞는 PV를 찾는 중 |
| Bound | PVC와 바인딩됨 | Bound | PV와 바인딩됨 |
| Released | PVC 삭제됨, 아직 재사용 전 (데이터 남아있을 수 있음) | - | - |
| Failed | 자동 회수 실패 | - | - |

---

## 3. StorageClass와 동적 프로비저닝

### 정적 프로비저닝의 한계

관리자가 PV를 미리 하나하나 수동으로 만들어두는 방식(정적 프로비저닝)은 클라우드 환경에서 비효율적이다. 요청이 있을 때마다 관리자가 개입해야 한다.

### 동적 프로비저닝(Dynamic Provisioning)

**PVC가 생성되는 순간 PV가 자동으로 만들어지는** 방식이다. 이를 가능하게 하는 것이 `StorageClass`다.

```
[StorageClass 정의]                [PVC 생성]
"fast-ssd"                         storageClassName: fast-ssd
provisioner: ebs.csi.aws.com       requests: 10Gi
       |                                  |
       +--------------> [Provisioner가 자동 감지] -----+
                                  |
                    실제 스토리지(EBS 볼륨 등) 생성
                                  |
                        PV 자동 생성 + PVC와 즉시 바인딩
```

- **StorageClass**: "이런 종류의 스토리지를 어떻게 만들지"에 대한 템플릿(클래스) 정의
- **Provisioner**: StorageClass에 지정된, 실제로 스토리지를 프로비저닝하는 플러그인/드라이버 (예: `kubernetes.io/aws-ebs`, `ebs.csi.aws.com`, `rancher.io/local-path`, `nfs.csi.k8s.io`)
- 요즘은 대부분 **CSI(Container Storage Interface)** 드라이버 형태로 provisioner가 제공됨 (In-tree 볼륨 플러그인은 deprecated)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

- `parameters`: provisioner마다 다른 세부 설정 (디스크 타입, 파일시스템 등)
- `volumeBindingMode`:
  - `Immediate` (기본값): PVC 생성 즉시 PV 프로비저닝
  - `WaitForFirstConsumer`: PVC를 사용하는 **Pod가 스케줄링될 때까지 대기** → 해당 노드/존에 맞는 스토리지를 만들 수 있어 클라우드 환경에서 권장
- PVC에 `storageClassName`을 비워두면 클러스터의 **기본(default) StorageClass**가 사용됨 (annotation `storageclass.kubernetes.io/is-default-class: "true"`로 지정)
- `storageClassName: ""`으로 명시하면 동적 프로비저닝을 사용하지 않겠다는 의미 (정적 PV만 바인딩 대상)

---

## 4. 접근 모드(AccessMode)

PV가 **동시에 몇 개의 노드/Pod에서 어떤 방식으로 마운트될 수 있는지**를 정의한다. 스토리지 하드웨어/드라이버가 지원하는 범위 내에서만 설정 가능하다.

| AccessMode | 축약 | 의미 |
|---|---|---|
| ReadWriteOnce | RWO | 한 번에 **하나의 노드**에서 읽기/쓰기 가능 (같은 노드의 여러 Pod는 공유 가능) |
| ReadOnlyMany | ROX | 여러 노드에서 동시에 **읽기 전용**으로 마운트 가능 |
| ReadWriteMany | RWX | 여러 노드에서 동시에 **읽기/쓰기** 가능 |
| ReadWriteOncePod | RWOP | 클러스터 전체에서 **단 하나의 Pod**만 읽기/쓰기 가능 (노드 단위가 아니라 Pod 단위로 제한, K8s 1.22+) |

```
RWO:  [Node A: Pod1, Pod2] ---mount---> [PV]     (Node B는 마운트 불가)

ROX:  [Node A: Pod1] --read--\
      [Node B: Pod2] --read---> [PV]
      [Node C: Pod3] --read--/

RWX:  [Node A: Pod1] --read/write--\
      [Node B: Pod2] --read/write---> [PV]

RWOP: [Node A: Pod1] --read/write--> [PV]   (다른 Pod는 같은 노드여도 마운트 불가)
```

- 클라우드 블록 스토리지(EBS, GCE PD, Azure Disk)는 보통 **RWO만 지원**
- NFS, CephFS, EFS 같은 파일 스토리지는 **RWX 지원 가능**
- RWOP는 "정말로 단일 writer만 허용해야 하는" 워크로드(예: 특정 클러스터형 DB)를 위해 도입됨
- PVC에 여러 AccessMode를 배열로 적을 수 있지만, 실제로는 PV가 지원하는 모드 중 **바인딩 시점에 하나의 방식으로만 마운트**된다

---

## 5. Reclaim Policy

PVC가 삭제되었을 때, 바인딩되어 있던 **PV를 어떻게 처리할지** 결정하는 정책. PV의 `spec.persistentVolumeReclaimPolicy`에 설정한다.

| 정책 | 동작 | 데이터 | 재사용 |
|---|---|---|---|
| Retain | PV를 삭제하지 않고 `Released` 상태로 보존 | 데이터 유지 | 관리자가 수동으로 정리 후 재사용해야 함 |
| Delete | PVC 삭제 시 PV와 **실제 스토리지 자원까지** 자동 삭제 | 데이터 삭제됨 | 자동 정리, 재사용 불가 |
| Recycle | (deprecated) 데이터를 지우고(`rm -rf`) PV를 재사용 가능 상태로 되돌림 | 데이터 삭제됨 | 자동 재사용 — 더 이상 권장 안 됨, 동적 프로비저닝으로 대체 |

```
PVC 삭제
   |
   +-- Retain --> PV: Bound -> Released (데이터 그대로, 관리자 개입 필요)
   |
   +-- Delete --> PV 및 하위 스토리지 자원 자체가 삭제됨
```

- 동적 프로비저닝으로 생성된 PV는 기본적으로 StorageClass의 `reclaimPolicy`를 따르며, 보통 **Delete가 기본값**
- 정적으로 만든 PV는 기본값이 **Retain**
- 중요한 데이터를 다루는 스토리지는 `kubectl patch pv <name> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'`로 정책을 바꿔 실수로 인한 삭제를 방지하는 것이 실무 관행

---

## 6. volumeMounts와 subPath

`volumeMounts`는 컨테이너 스펙에서 **Volume을 파일시스템의 어느 경로에 마운트할지** 지정한다.

```yaml
containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: data-vol
        mountPath: /usr/share/nginx/html
volumes:
  - name: data-vol
    persistentVolumeClaim:
      claimName: my-pvc
```

- 하나의 Volume을 **여러 컨테이너가 서로 다른 mountPath**로 각자 마운트할 수 있다 (같은 Pod 내에서 데이터 공유)
- `mountPath`에 이미 파일이 있던 경우, 마운트 시 **가려짐**(overlay되어 보이지 않게 됨)에 주의

### subPath

하나의 Volume 안에서 **특정 하위 경로만** 컨테이너에 마운트하고 싶을 때 사용한다.

```yaml
volumeMounts:
  - name: shared-vol
    mountPath: /var/log/app
    subPath: app-logs
```

```
Volume (shared-vol) 내부 구조:
  /app-logs/    <-- 이 부분만 컨테이너의 /var/log/app 에 마운트됨
  /app-config/
  /app-cache/
```

- 하나의 PVC를 **여러 컨테이너가 서로 다른 하위 디렉터리**로 나눠 쓸 때 유용 (예: 사이드카 패턴, 멀티 컨테이너가 로그/설정을 분리해서 사용)
- ConfigMap/Secret을 볼륨으로 마운트할 때, 디렉터리 전체가 아니라 **파일 하나만** 특정 경로에 넣고 싶을 때도 자주 사용
- 단점: subPath로 마운트된 파일은 **동적 업데이트가 반영되지 않는** 경우가 있음 (ConfigMap 변경 시 전체 마운트는 자동 갱신되지만 subPath는 갱신 안 됨)

---

## 용어 요약

| 용어 | 설명 |
|---|---|
| Volume | Pod 스펙에 정의되는 저장 공간. 종류에 따라 생명주기가 다름 |
| emptyDir | Pod 생성 시 만들어지는 빈 볼륨. Pod 삭제 시 함께 삭제 |
| hostPath | 노드의 파일시스템 경로를 그대로 마운트 |
| PersistentVolume (PV) | 클러스터 스코프의 실제 스토리지 자원 오브젝트 |
| PersistentVolumeClaim (PVC) | 네임스페이스 스코프의 스토리지 요청 오브젝트 |
| Binding | PVC와 PV가 1:1로 연결되는 과정 |
| StorageClass | 동적 프로비저닝을 위한 스토리지 템플릿 정의 |
| Provisioner | StorageClass가 사용하는, 실제 스토리지를 만드는 드라이버/플러그인 |
| CSI | Container Storage Interface, 표준화된 스토리지 드라이버 인터페이스 |
| AccessMode | PV가 지원하는 동시 마운트 방식(RWO/ROX/RWX/RWOP) |
| Reclaim Policy | PVC 삭제 시 PV 처리 방식(Retain/Delete/Recycle) |
| volumeMounts | 컨테이너에서 Volume을 마운트하는 경로 지정 |
| subPath | Volume 내 특정 하위 경로만 마운트 |
| volumeBindingMode | StorageClass의 PV 생성 타이밍 (Immediate/WaitForFirstConsumer) |

---

## 흔한 실수

- **실수 1**: "emptyDir은 컨테이너가 재시작돼도 사라진다"
  - 정정: emptyDir은 **Pod** 생명주기를 따른다. 같은 Pod 안에서 컨테이너가 재시작(crash 후 재기동)되어도 emptyDir 데이터는 유지된다. Pod 자체가 삭제될 때만 사라진다.

- **실수 2**: "PVC를 삭제하면 데이터는 항상 사라진다"
  - 정정: Reclaim Policy가 `Retain`이면 PVC를 삭제해도 PV와 데이터는 `Released` 상태로 남아있다. `Delete`일 때만 실제 스토리지까지 삭제된다.

- **실수 3**: "StorageClass를 만들면 기존 PVC에도 자동 적용된다"
  - 정정: PVC는 생성 시점에 StorageClass를 참조하며, 이후 PVC의 `storageClassName`은 **불변(immutable)**이다. 기존 PVC를 다른 StorageClass로 옮기려면 새 PVC를 만들고 데이터를 마이그레이션해야 한다.

- **실수 4**: "AccessMode를 RWX로 설정하면 어떤 스토리지든 여러 노드에서 쓸 수 있다"
  - 정정: AccessMode는 PV/스토리지 드라이버가 **실제로 지원하는 범위 내에서만** 유효하다. EBS 같은 블록 스토리지는 RWX를 지원하지 않으므로 PV에 RWX를 적어도 실패한다.

- **실수 5**: "PV의 capacity와 PVC의 요청량이 정확히 같아야 바인딩된다"
  - 정정: PV의 capacity가 PVC 요청량 **이상**이면 바인딩된다. 다만 남는 용량은 활용할 수 없고 낭비된다.

---

## 빠른 복습

- emptyDir은 Pod 생명주기, hostPath는 노드 생명주기를 따르며 둘 다 특정 노드에 종속된다는 한계가 있다.
- PV는 관리자가 정의하는 클러스터 스코프의 실제 스토리지, PVC는 사용자가 요청하는 네임스페이스 스코프의 요청서이며 조건(용량, AccessMode, StorageClass)이 맞으면 1:1로 바인딩된다.
- StorageClass + Provisioner 조합으로 PVC 생성 시 PV가 자동 생성되는 동적 프로비저닝이 가능하다.
- AccessMode는 RWO(단일 노드), ROX(다중 읽기), RWX(다중 읽기/쓰기), RWOP(단일 Pod)로 구분되며 스토리지가 실제로 지원하는 모드만 사용 가능하다.
- Reclaim Policy는 PVC 삭제 후 PV 처리 방식을 결정한다 — Retain(보존), Delete(삭제), Recycle(deprecated).
- volumeMounts는 볼륨을 컨테이너 경로에 연결하고, subPath는 볼륨 내 특정 하위 경로만 선택적으로 마운트할 때 사용한다.

---

> 다음 단계: [CKA 05 - Storage (실습)](CKA-05-Storage-실습.md) — PV/PVC 수동 바인딩과 StorageClass 동적 프로비저닝을 직접 손으로 실습한다.
