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

그냥 조지면 된다.

```sh
envsubst < central_monitor.yaml | helm upgrade monitor prometheus-community/kube-prometheus-stack --version 86.0.0 -n monitor -f -
```

#### PV 쓰는 경우

grafana 13이 구버전 db를 신스키마로 자동 변환해 사용하는데, 한번 변환된 db는 다시 구버전에서 못 읽으므로 백업만 떠두고 조진다.
(grafana.db = UI에서 만든 대시보드/알람룰/유저/datasource 등 모든 커스텀 자산. README의 "grafana.db" 참고)
일반적인 마이그레이션 대상은 grafana.db + grafana.ini. 본 프로젝트는 ini를 helm value로 관리하므로 db만 옮기면 됨.

```sh
# "컨테이너 내 PV경로"의 grafana.db를 "컨테이너 내 /tmp"로 복사(운영중 안정적인 스냅샷을 위해 컨테이너 내부에서 복제)
kubectl exec -n monitor <grafana-pod> -- sqlite3 /var/lib/grafana/grafana.db ".backup /tmp/grafana.db"

# 컨테이너 내 /tmp에서 호스트 로컬로 복사
kubectl cp monitor/<grafana-pod>:/tmp/grafana.db ./grafana-$(date +%F).db
```

망하면 복사해둔 파일로 DB 갈아끼우고 `helm rollback monitor -n monitor`.

> 구버전 grafana.db를 신버전 PV에 심어 이식하는 동작 확인됨 (UI 자산 정상 복구).
> 단 아래 "이슈 대응"의 권한/png·pdf·csv 디렉토리 처리 필요.

### 이슈 대응

#### 호스트에서 PV로 db 직접 주입시 권한 문제

grafana 서브차트 securityContext와 동일 UID:GID로 소유권 맞추기.

```sh
sudo chown -R 472:472 <pv 경로>
```

#### 구버전 grafana.db를 신버전 PV에 심어 이식 시 init container 실패

(docker 기반 쿠버네티스에서 발생, initChownData는 처음에만 쓰고 꺼두는게 나을듯)
신버전 첫 install 때 PV에 만들어진 `png`/`pdf`/`csv` 디렉토리가 남아있는 상태에서 db를 구버전으로 교체하고 재install하면, init container가 해당 디렉토리 chown에서 Permission denied로 막힘.
이 세 디렉토리는 grafana가 PNG 렌더/PDF·CSV export 결과를 쌓는 임시 출력 캐시 → 자동 재생성, UI 자산 무관, 삭제해도 안전.

```sh
sudo rm -rf <pv 경로>/{png,pdf,csv}
```

