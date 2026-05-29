# CMM: Centralized Monitoring for Multi-clusters

여러 클러스터를 한군데서 모니터링

헬름차트 kube-prometheus-stack의 value를 적절히 수정하여 구현

- node-exporter: 모니터링 대상 클러스터에서 helm install시, daemonset에 의해 개별 노드마다 하나씩 배치됨
- Prometheus: 모니터링 대상 클러스터마다 설치
- grafana: 중앙 노드에 설치되고, 각 클러스터의 prometheus를 datasource로서 참조
- thanos: 클러스터 규모가 클 경우 Prometheus의 부하가 과해질 수 있다. 이 때는 중앙 노드에 Thanos를 추가 설치해야 한다.(현재는 미설정)

## 추가목표

관리 및 설치 편의를 위해 다음을 목표로 한다.

- 가급적 기존 헬름차트의 value 수정 만으로 해결
- 커스텀 차트 생성 X, 헬름 template 편집 X
- 대시보드 등 커스텀 요소 관리방법
  - json을 value파일에 포함하여 프로비저닝
  - 온라인 기반 레포지토리에서 다운로드하여 프로비저닝
  - 프로비저닝 외 커스텀 요소는 Persistent Volume 활용
  - root경로의 파일을 가져오는 extraVolume은 미사용 (설치시 번거로움)
  - value 파일에서 ConfigMap 생성가능하나, tpl함수와 json 문법이 충돌하여 사용이 사실상 힘듦

## 사전 설정

```sh
# Add Repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# (오프라인 환경 등)아카이브 파일 필요시
helm pull prometheus-community/kube-prometheus-stack --version 69.2.4
```

## 중앙 모니터

```sh
# grafana only
helm install monitor prometheus-community/kube-prometheus-stack --version 69.2.4 -n monitor -f central_monitor.yaml

# grafana only (환경변수 사용)
envsubst < central_monitor.yaml | helm install monitor prometheus-community/kube-prometheus-stack --version 69.2.4 -n monitor -f -
```

## 개별 클러스터 설정

- 관리 용이성을 위해, 클러스터 별 설정은 모두 동일한 것을 목표로 함
- 단, `릴리즈명은 개별 고유 이름을 포함`하면 좋음
  - Grafana alert 등 에서 조회시, 클러스터 별 메트릭 Label이 구분되어 보기 편함

```sh
# Install exporter and prometheus 
helm install monitor prometheus-community/kube-prometheus-stack --version 69.2.4 -f each_cluster.yaml -n monitor 

# prometheus nodeSelector 지정 예시
helm upgrade monitor prometheus-community/kube-prometheus-stack --version 69.2.4 -f each_cluster.yaml -n monitor \
--set "prometheus.prometheusSpec.nodeSelector.kr-mum/noderole=kafka" \
--set "prometheusOperator.nodeSelector.kr-mum/noderole=kafka"

# kube metric 수집 활성화 예시
helm upgrade monitor prometheus-community/kube-prometheus-stack --version 69.2.4 -f each_cluster.yaml -n monitor \
--set "kubernetesServiceMonitors.enabled=true" \
--set "kubeStateMetrics.enabled=true"

# WSL에서 실행 시 추가옵션 (WSL, 도커데탑 등 일부 환경에서 node exporter 실행 실패시에만 사용)
helm install wsl-prom prometheus-community/kube-prometheus-stack --version 69.2.4 -f each_cluster.yaml \
--set "prometheus-node-exporter.hostRootFsMount.enabled=false" \
-n monitor
```

## 용어

### [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack?modal=install)

- node-exporter, promethues, promethues-operator, thanos, grafana로 구성된 차트
- 자주 쓰이는 조합이라 비슷한 차트가 이미 공유되고 있으며, 그 중에서도 promethues 커뮤니티에서 나온 kube-promethues-stack이 가장 널리 사용된다.
- 구 차트이름이 prometheus-operator였으나, 현재는 일부 구성 요소만 prometheus-operator라고 부름
- 이러한 차트 대부분은 단일 클러스터 내 사용을 가정하여 만들어짐
- 기존 차트로 멀티클러스터에 대해 Centralized 모니터링을 하려면 수정 필요

### ServiceMonitor

- prometheus-operator에서 제공하는 CRD
- ServiceMonitor는 Prometheus가 exporter의 Service를 자동으로 발견하고, 이 Service로 노출되는 메트릭을 수집할 수 있게 해준다.
- 기존에는 Prometheus 설정파일(`prometheus.yaml(prometheusSpec)`)에 exporter 정보를 매번 수동 반영해야 하는데, ServiceMonitor를 통해 이 과정을 자동화하고 동적으로 관리할 수 있다.
  - e.g. 스케일링이 잦은 클러스터에서 daemonset으로 배포된 exporter => 매번 변경되는 exporter 정보를 Prometheus에 반영하려는 경우
  - e.g. 아주 많은 수의 exporter를 Promethues에 반영하려는 경우
- prometheus-operator가 사실상 업계표준이므로, 모니터링 대상이 될만한 여러 서비스의 헬름차트에서 Prometheus연동을 위한 ServiceMonitor 관련 옵션을 제공한다.
  - 단, ServiceMonitor는 동일 클러스터 내에서 동작하는 것을 목적으로 만들어졌기 때문에, 원격 환경에선 적합하지 않다.
- 다시 읽어보기: [Service Monitor 컨셉, 쓰는 이유, 장점](https://jerryljh.medium.com/prometheus-servicemonitor-98ccca35a13e)

#### ServiceMonitor 동작 방식

- Prometheus, ServiceMonitor 모두 CRD임
- 각 리소스에 설정된 `전용 selector를 통해서 prometheus=>servicemonitor=>service=>exporter(Pod) 순으로 label match`하는 방식임
  - Promtheus CRD의 `serviceMonitorSelector` 
  - ServiceMonitor CRD의 `spec.selector.matchLabels`
  - exporter는 반드시 Service(ClusterIP)로 배포되어 있어야 함
- kube-prometheus-stack에서 배포하는 node-exporter,ServiceMonitor는 자동매칭되도록 기본값이 설정되어있지만, `다른 헬름차트의 ServiceMonitor를 연동하려면 Prometheus의 serviceMonitorSelector에 추가등록이 필요`하다.
- `serviceMonitorSelector={}`이면 클러스터 내 모든 ServiceMonitor를 수집대상으로 하는데, kube-prometheus-stack 헬름차트에선 다음 값을 false로 설정하면 된다.

```yaml
prometheus:
  prometheusSpec:
    # false시 클러스터 내 모든 ServiceMonitor를 수집대상으로 함
    # true(default)시, kube-prometheus-stack에 의해 배포된 ServiceMonitor만 수집대상으로 함
    serviceMonitorSelectorNilUsesHelmValues: false  
```

### Sidecar

- 메인컨테이너와 함께 동작하는 보조컨테이너(helper container)를 의미
- 메인컨테이너 앱의 주 기능 외에 로그처리, 모니터링, 보안, 프록시, 파일 시스템 관리 등의 용도로 사용
- 디커플링, 재사용성, 유지보수성에 유리

## Alert

- 흔히 알려진 AlertManager는 Prometheus의 기능이고, Prometheus 본 앱과 별도로 실행되는 앱이다.
- CMM은 통합 모니터링을 지향하므로, 중앙 grafana에서 alerting을 설정
- kps 차트에서 grafana의 alerting 설정은 grafana의 subchart value or grafana ui를 활용

### grafana alerting 특징

- prometheus alertmanager와도 기능이 겹침. grafana stack(loki) 쓸 땐 좋음
- prometheus 외 다양한 datasource에 대해 알람가능
- 장점
  - 멀티 클러스터 모니터링시 중앙에서 관리가능
  - 프로비저닝 뿐 아니라, UI에서도 간단히 설정가능
  - Alertmanger보다 간소하지만, 메트릭이 '임계값을 초과하고 특정 시간동안 지속'되는 것을 조건으로 알람하는 수준은 충분히 가능.
  - SMTP, MS팀즈 전달도 가능
- 단점
  - Prometheus AlertManager 보다는 부족한 기능
  - 버전마다 기능 급변 중

### promethues alertmanager 특징

- 장점
  - 널리 사용됨, 안정적
  - 여러 메트릭 복합 조건으로 트리거 가능
  - 특정 시간대만 검사해서 Alert 발생 가능
    - => 근데 왜 kps차트는 grafana쪽 설정에 alertmanager가 있지???
    - => 이거는 datasource로 alertmanager를 별도 추가하기 위한 용도다.
    - => Grafana에서 AlertManager를 Datasource로 추가하는 것은, Alertmanager에서 생성된 알림을 Grafana 대시보드에서 시각화하고 관리하기 위함(threshold를 넘어선 횟수, 내역 등 체크)
- 단점
  - UI에서 규칙편집 불가. 그냥 현 상황 조회 정도 가능.
  - 고도화된 모니터링이 필요없다면 관리비용만 늘어남
  - 멀티 클러스터 모니터링시 수집지표에 따라 개별 클러스터마다 네트워크 인가 필요할 수 있음
    - 단, AlertManager의 모든 지표는 Prometheus가 수집가능하기 때문에 꼭 필요하진 않음
  - 클러스터마다 필요한 알람조건, 지표가 다를 수 있어서 관리 불편
  - kps차트에서 alert 관련기능은 모두 prometheus alertmanager의 rule 프로비저닝으로 관리되는 것 같다.

## 기타 관련 이슈

### prometheus를 각 클러스터에 둘 것인가, 중앙모니터링 서버에 둘 것인가?

- 서비스에 두면 메모리 부담, 중앙에 두면 더 복잡한 설정 필요
- 각 클러스터에 두는 것으로 종결
- 고도화시 중앙엔 Thanos 추가

### 다른 Centralized 모니터링 사례

- 보통은 각 서비스, 각 클러스터에 prometheus를 배치하고, 중앙에는 Thanos라는 툴을 둠
- prometheus는 클러스터링을 지원하지않는데, 여러 promethues를 하나의 앱처럼 관리하기 위한 툴이 Thanos임
- 중앙에 단일 prometheus 두고 가벼운 모니터링시스템으로 쓰다가 나중에 필요할 때 업글할까? 아니면 처음부터 체계적으로 구축할까도 고민. 급하지는 않은데...
- 하나의 Prometheus에 여러 grafana가 접근가능할 듯한데, 굳이 Thanos가 필요할까???
- 하나의 exporter에 여러 prometheus가 붙을 수 있나? => 이거 포트가 점유되어있다고 안될거같은데...

### EKS 에서 권한(Role)문제

- 권한 이슈는 환경 제공자마다 다를 수 있다.
- GKE에선 권한 최적화해서 해결하는 방법이 있다는 것 같은데, EKS에선 관리자 권한 전체 부여해야할듯
- [참고](https://github.com/prometheus-operator/prometheus-operator/issues/1189)

```sh
# clusterrole 중에 관리자 권한(cluster-admin)이 기본으로 있고, 이걸 사용하는 user에게 clusterbinding으로 연결해준다.
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=your.email@address.com 
kubectl create clusterrolebinding yunan-cluster-admin-binding --clusterrole=cluster-admin --user=yunan_all
# --set "prometheus.prometheusSpec.nodeSelector.wai-eks/noderole=home" \
```

### grafana.db (UI 자산 저장소)

- UI에서 만든 모든 커스텀 정보(대시보드/알람룰/유저/UI 추가 datasource 등)는 PV 내 단일 SQLite 파일 `/var/lib/grafana/grafana.db`에 저장.
- 일반적인 마이그레이션 대상은 `grafana.db` + `grafana.ini`. 본 프로젝트는 ini를 helm value로 관리하므로 db만 옮기면 됨.
- 버전이 다른 db라도 신버전 grafana가 알아서 신스키마로 변환해 사용한다(자동 마이그레이션). 단 변환은 구→신 한 방향만 가능 — 한번 변환된 db는 다시 구버전에서 못 읽는다.

#### 같이 옮길 필요 없는 것

- 프로비저닝된 자산(차트 values/configmap으로 깐 dashboard·datasource·alerting)
  - db에 사본이 있어도 pod 뜰 때 차트가 매번 덮어쓰므로 db 백업과 무관
  - 진본은 values 파일

#### 주의사항

- **git에 올리지 말 것**: API 키/datasource 비번/유저 해시 등 민감정보 포함
- **호스트에서 PV로 db 직접 넣을 때**: grafana 서브차트 securityContext(default UID/GID 472)와 동일한 소유권으로 맞추기. `sudo chown -R 472:472 <pv 경로>`
- **구버전 grafana.db를 신버전 PV에 심어 이식할 때**: init container가 `png`/`pdf`/`csv` 디렉토리 chown에서 Permission denied로 막힘. 이 세 디렉토리는 신버전 첫 install 때 만들어진 PNG 렌더/PDF·CSV export 임시 캐시라 자동 재생성됨. 삭제해도 안전. `sudo rm -rf <pv 경로>/{png,pdf,csv}`

### grafana 13에서 dashboard provisioner inotify 부족

- 로그: `error watching folder ... too many open files`
- 원인: grafana 13이 11보다 inotify watcher를 훨씬 많이 사용. 같은 호스트에 grafana 11/13 둘 다 떠있거나 sidecar까지 같이 돌면 호스트 default(`max_user_instances=128`, `max_user_watches=29638`)가 부족해짐
- 해결: 호스트에서 inotify 한계 상향

```sh
sudo sysctl -w fs.inotify.max_user_instances=512
sudo sysctl -w fs.inotify.max_user_watches=524288

# 영구 적용
echo 'fs.inotify.max_user_instances=512' | sudo tee -a /etc/sysctl.conf
echo 'fs.inotify.max_user_watches=524288' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

이후 grafana pod 재시작.

### docker 런타임에서 grafana init container chown 매번 실패

- k3s가 docker로 동작하는 환경 등에서 docker default seccomp/apparmor가 chown syscall을 제한
- init container가 매 부팅마다 `Permission denied`로 막힘
- PV는 이미 472:472라 init chown 자체가 불필요하므로 values에 다음 추가해서 끔

```yaml
grafana:
  initChownData:
    enabled: false
```

### PV 삭제 안됨(STATUS=TERMINATING에서 멈춤 현상)

- pv는 namespace가 상관없지만, pvc는 헬름 설치시 선택한 namespace에 있다.
- pvc를 삭제해야 한다.

```sh
kubectl delte pvc {PVC_NAME} -n {NAMESPACE}
```

- 그래도 안되는 경우 다음 명령어를 쓰면 pv 삭제 가능

```sh
kubectl patch pv {PV이름} -p '{"metadata":{"finalizers":null}}'
```
