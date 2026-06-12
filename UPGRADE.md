# kube-prometheus-stack 업그레이드

## central_monitor (grafana)

### value 파일 추가

- 이 외 변경필요 사항 없음

```yaml
# 86.0.0부터 차트가 prometheus-operator CRD를 기본 ON으로 깔지만, 중앙은 operator 미사용이라 불필요
crds:
  enabled: false

# default true. CRD 없는 환경에서는 PrometheusRule 매니페스트 생성 시도 → install 실패. operator 미사용이라 룰셋 무용
defaultRules:
  create: false

grafana:
  # docker 런타임 환경(k3s가 docker로 동작 등)에서 default seccomp/apparmor가 chown syscall 제한 → init container가 매 부팅마다 막힘
  # PV가 이미 472:472라 init chown 자체가 불필요하므로 끔
  initChownData:
    enabled: false
```

### 로컬 테스트시

- 알람관련 설정에서 빈값이 들어갈 경우 배포 실패하도록 변경됨
- `source dummy.env.test` 후 배포할 것

### 실제 업그레이드

#### PV 안 쓰는 경우

- 그냥 조지면 된다.

```sh
envsubst < central_monitor.yaml | helm upgrade monitor prometheus-community/kube-prometheus-stack --version 86.2.2 -n monitor -f -
```

#### PV 쓰는 경우

- grafana 13이 구버전 db를 신스키마로 자동 변환해 사용하는데, 한번 변환된 db는 다시 구버전에서 못 읽으므로 백업만 떠두고 조진다
- grafana.db = UI에서 만든 대시보드/알람룰/유저/datasource 등 모든 커스텀 자산 (README의 "grafana.db" 참고)
- 일반적인 마이그레이션 대상은 grafana.db + grafana.ini. 본 프로젝트는 ini를 helm value로 관리하므로 db만 옮기면 됨

```sh
# "컨테이너 내 PV경로"의 grafana.db를 "컨테이너 내 /tmp"로 복사(운영중 안정적인 스냅샷을 위해 컨테이너 내부에서 복제)
kubectl exec -n monitor <grafana-pod> -- sqlite3 /var/lib/grafana/grafana.db ".backup /tmp/grafana.db"

# 컨테이너 내 /tmp에서 호스트 로컬로 복사
kubectl cp monitor/<grafana-pod>:/tmp/grafana.db ./grafana-$(date +%F).db
```

- 망하면 복사해둔 파일로 DB 갈아끼우고 `helm rollback monitor -n monitor`
- 구버전 grafana.db를 신버전 PV에 심어 이식하는 동작 확인됨 (UI 자산 정상 복구)
  - 단 아래 "이슈 대응"의 권한/png·pdf·csv 디렉토리 처리 필요

### 이슈 대응

#### 호스트에서 PV로 db 직접 주입시 권한 문제

- grafana 서브차트 securityContext와 동일 UID:GID로 소유권 맞추기

```sh
sudo chown -R 472:472 <pv 경로>
```

#### 구버전 grafana.db를 신버전 PV에 심어 이식 시 init container 실패

- docker 기반 쿠버네티스에서 발생. initChownData는 처음에만 쓰고 꺼두는게 나을듯
- 신버전 첫 install 때 PV에 만들어진 `png`/`pdf`/`csv` 디렉토리가 남아있는 상태에서 db를 구버전으로 교체하고 재install하면, init container가 해당 디렉토리 chown에서 Permission denied로 막힘
- 이 세 디렉토리는 grafana가 PNG 렌더/PDF·CSV export 결과를 쌓는 임시 출력 캐시 → 자동 재생성, UI 자산 무관, 삭제해도 안전

```sh
sudo rm -rf <pv 경로>/{png,pdf,csv}
```

#### grafana 13에서 dashboard provisioner가 `too many open files`

- grafana 13이 11보다 inotify watcher를 많이 사용. 호스트 default 한계(`max_user_instances=128`)가 부족해 dashboard provisioner가 폴더 watch 실패
- 호스트에서 sysctl 값 상향:

```sh
sudo sysctl -w fs.inotify.max_user_instances=512
sudo sysctl -w fs.inotify.max_user_watches=524288

# 영구 적용
echo 'fs.inotify.max_user_instances=512' | sudo tee -a /etc/sysctl.conf
echo 'fs.inotify.max_user_watches=524288' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

## each_monitor (node-exporter, prometheus)

- 각 클러스터 value(`each_cluster.yaml` 및 타 저장소의 동등 파일)를 69.2.4 → 86.2.2로 올릴 때 손봐야 할 항목
- grafana는 each cluster에서 꺼두므로(`grafana.enabled: false`) grafana 서브차트 변화(8.9→12.4)는 무관
- 아래 외의 사용 중인 key(alertmanager, nodeExporter, prometheus.prometheusSpec.\*, serviceMonitorSelectorNilUsesHelmValues, kube\*ServiceMonitors, kubeStateMetrics, prometheusOperator, crds.enabled 등)는 key·default 변동 없이 호환 유지되므로 그대로 두면 됨

### 변경 항목 요약

- `prometheus-node-exporter.rbac.pspEnabled` 제거 (필수 권장)
  - 86의 node-exporter 서브차트(4.55)에서 키 자체가 제거됨 (k8s 1.25 PSP 삭제 영향)
  - 남겨도 helm이 무시해 install이 깨지진 않지만 의미 없는 잔재이므로 삭제
- CRD (monitoring.coreos.com 10종) 갱신 (필수)
  - helm은 upgrade 시 CRD를 자동 갱신하지 않음
  - 수동 apply 또는 `crds.upgradeJob.enabled: true` (아래 "CRD 처리" 참고)
- `prometheus-node-exporter.extraArgs` 갱신 (권장)
  - 86 기본 필터 정규식에 `run/containerd/.+`, `erofs`가 추가됨
  - 우리 value의 `--collector.filesystem.*` 정규식을 신버전 기준으로 갱신

- 신규 optional 필드(`prometheus.service.enabled`, histogram 관련 `convert/scrape*Histograms`, `prometheusOperator.podDisruptionBudget` 등)는 안전한 default라 건드릴 필요 없음
- operator의 webhook patch 이미지 출처가 `registry.k8s.io/.../kube-webhook-certgen` → `ghcr.io/jkroepke/kube-webhook-certgen`로 바뀐 건 차트 내부값. 오프라인/사설 레지스트리 환경이면 이미지 pull 경로만 참고

### pspEnabled 제거

```yaml
prometheus-node-exporter:
  rbac:
    # pspEnabled: false   # 86에서 키 제거됨 → 삭제
    create: true
```

### extraArgs 갱신 (권장)

- 86 기본값에 맞춰 filesystem 필터 정규식 갱신
- `run/containerd/.+`(containerd 런타임), `erofs` 추가

```yaml
prometheus-node-exporter:
  extraArgs:
    - --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run/containerd/.+|var/lib/docker/.+|var/lib/kubelet/.+)($|/)
    - --collector.filesystem.fs-types-exclude=^(autofs|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|iso9660|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs|erofs)$
```

### CRD 처리

- prometheus-operator 사용 환경이므로 monitoring.coreos.com CRD 10종이 필요
  - alertmanagerconfigs, alertmanagers, podmonitors, probes, prometheusagents, prometheuses, prometheusrules, scrapeconfigs, servicemonitors, thanosrulers

#### 첫 설치는 자동 (별도 작업 불필요)

- CRD 10종이 crds 서브차트(`charts/crds/crds/*.yaml`)에 helm native `crds/` 방식으로 포함됨
- `crds.enabled: true`(기본값)면 install 시 자동 적용 → 신규 클러스터 첫 설치는 추가 작업 없음

#### 업그레이드 때만 수동 갱신 필요 (필수)

- helm은 install 때만 native `crds/`를 적용. upgrade/rollback 시에는 CRD를 건드리지 않음
- 차트 출처(repo, 로컬 tgz, 디렉토리) 무관하게 동일
- `helm uninstall` 후 재install도 마찬가지 — uninstall은 CRD를 안 지우고 install은 기존 CRD를 skip
  - 기존 클러스터: 버전 올려도 CRD는 구버전 그대로 남음 → 수동 갱신 필요
  - CRD 없는 클린 클러스터: install이 자동으로 깖
- 반영 순서: CRD 먼저 apply → 그다음 helm 실행 (새 operator가 새 CRD 스키마를 기대하므로)
  - `crds.upgradeJob.enabled: true` 방식은 hook이 알아서 이 순서로 처리

갱신 방법 (택1):

- 차트 옵션으로 CRD 자동 갱신 (preview 기능)
  - helm hook으로 busybox(docker.io), kubectl(registry.k8s.io) 이미지 Job을 띄워 차트 내장 CRD를 apply
  ```yaml
  crds:
    upgradeJob:
      enabled: true
  ```
- helm upgrade 전에 수동 apply
  - `--server-side` 필수 (CRD 스키마가 커서 client-side는 어노테이션 256KB 한도 초과로 실패)
  - 업그레이드 시 기존 CRD를 다른 매니저(helm 등)가 소유 → 필드 충돌 경고가 뜨므로 `--force-conflicts`로 소유권을 가져옴
  ```sh
  # 86.2.2의 operator 버전에 맞춰 CRD 10종 적용
  for crd in alertmanagerconfigs alertmanagers podmonitors probes prometheusagents prometheuses prometheusrules scrapeconfigs servicemonitors thanosrulers; do
    kubectl apply --server-side --force-conflicts -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.91.0/example/prometheus-operator-crd/monitoring.coreos.com_${crd}.yaml
  done
  ```

#### 폐쇄망에서 CRD 갱신

- 수동 apply는 `raw.githubusercontent.com`에서 받아오므로 폐쇄망에선 불가
- 외부 다운로드 없이 차트 tgz 내장 CRD를 그대로 사용 (버전도 차트와 정확히 일치)
  ```sh
  # 차트 아카이브에서 CRD 10종 추출
  tar xzf kube-prometheus-stack-86.2.2.tgz kube-prometheus-stack/charts/crds/crds
  # 추출된 yaml 적용 (for문 불필요, 디렉토리째 apply)
  kubectl apply --server-side --force-conflicts -f kube-prometheus-stack/charts/crds/crds/
  ```
- CRD 경로가 이미 로컬에 있으면 추출 없이 그 디렉토리만 가리키면 됨
  ```sh
  # --server-side: CRD 스키마가 커서 client-side는 256KB 한도 초과로 실패
  # --force-conflicts: 기존 CRD 소유자(helm 등)와의 필드 충돌 무시하고 덮어씀
  kubectl apply --server-side --force-conflicts -f <crds 디렉토리 경로>/
  ```
- upgradeJob 방식을 폐쇄망에서 쓰려면: busybox/kubectl 이미지를 사내 레지스트리에 미러링하고 `crds.upgradeJob.image.busybox.registry`, `crds.upgradeJob.image.kubectl.registry`를 사내 레지스트리로 덮어쓰기

