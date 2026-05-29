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
```

## 로컬 테스트시

- 알람관련 설정에서 빈값이 들어갈 경우 배포 실패하도록 변경됨
- `source dummy.env.test` 후 배포할 것


### 실제 업그레이드

PV 안 쓰면 그냥 조지면 된다.

```sh
envsubst < central_monitor.yaml | helm upgrade monitor prometheus-community/kube-prometheus-stack --version 86.0.0 -n monitor -f -
```

PV 쓰면 grafana 13이 SQLite DB를 신스키마로 자동 마이그레이션(단방향)하므로, DB 백업만 떠두고 조진다.
(grafana.db = UI에서 만든 대시보드/알람룰/유저/datasource 등 모든 커스텀 자산. README의 "grafana.db" 참고)

```sh
# "컨테이너 내 PV경로"의 grafana.db를 "컨테이너 내 /tmp"로 복사(운영중 안정적인 스냅샷을 위해 컨테이너 내부에서 복제)
kubectl exec -n monitor <grafana-pod> -- sqlite3 /var/lib/grafana/grafana.db ".backup /tmp/grafana.db"

# 컨테이너 내 /tmp에서 호스트 로컬로 복사
kubectl cp monitor/<grafana-pod>:/tmp/grafana.db ./grafana-$(date +%F).db
```

망하면 복사해둔 파일로 DB 갈아끼우고 `helm rollback monitor -n monitor`.
