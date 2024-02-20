# [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack?modal=install)

- node-exporter, promethues, promethues-operator, grafana로 구성된 차트다.
- 사람들이 많이 쓰는 조합이기 때문에 이미 통합 차트로 만들어진 사례가 있으며, 그 중에서도 promethues 커뮤니티에서 나온 kube-promethues-stack이 종종 사용된다.
  - 근데 설정이 좀 과하긴하다. value파일 4천라인쯤 됨
- grafana에서 출시한 k8s-monitoring 차트도 있는데, 천 라인쯤 됨
  - 모든 기능을 커스텀할 게 아니라면 차라리 따로따로 되있는걸 커스텀으로 합치는게 더 편할 것 같기도..
  - 일단 좀 봐야 알겠다.

- 구 차트이름이 prometheus-opeator였으나, 변경됨. 현재는 일부 구성 요소를 prometheus-operator라고 부름

## 단일 클러스터 내 모니터링

```sh
# Add Repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm update

# (오프라인 환경 등)아카이브 파일 필요시
helm pull prometheus-community/kube-prometheus-stack --version 55.8.3

# 기본 설치
helm install prostack prometheus-community/kube-prometheus-stack --version 55.8.3
# value 포함 후 monitor ns에설치
helm install prostack prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f single_cluster.yaml
# 업글
helm upgrade prostack prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f single_cluster.yaml
```

## 중앙 모니터

```sh
# grafana only
helm install monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f central_monitor.yaml
helm upgrade monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f central_monitor.yaml

# grafana only (환경변수 사용)
envsubst < central_monitor.yaml | helm install monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f -
envsubst < central_monitor.yaml | helm upgrade monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f -
```

## 개별 클러스터 설정

```sh
# each cluster  exporter and prometheus 
helm install monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f each_cluster.yaml 

# prometheus nodeSelector 지정 예시
helm upgrade monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n monitor -f each_cluster.yaml \
--set "prometheus.prometheusSpec.nodeSelector.kr-mum/noderole=kafka" \
--set "prometheusOperator.nodeSelector.kr-mum/noderole=kafka"

# kube metric 켜기 (EKS)
helm upgrade monitor prometheus-community/kube-prometheus-stack --version 55.8.3 -n platform -f 0_each_cluster.yaml \
--set "kubernetesServiceMonitors.enabled=true" \
--set "kubeStateMetrics.enabled=true"
```

### jmx, kafka exporter 테스트용 앱

```sh
helm install test https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/skafka-2.0.3.tgz \
-f https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/kraft-multi.yaml \
--set "kafka.externalAccess.autoDiscovery.enabled=false" \
--set "kafka.externalAccess.controller.service.nodePorts={30003,30004,30005}" \
--set "kafka.metrics.kafka.enabled=true" \
--set "kafka.metrics.jmx.enabled=true"

helm install test https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/skafka-2.0.3.tgz \
-f https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/kraft-multi.yaml \
--set "kafka.externalAccess.autoDiscovery.enabled=false" \
--set "kafka.externalAccess.controller.service.nodePorts={30003,30004,30005}" \
--set "kafka.metrics.kafka.enabled=true" \
--set "kafka.metrics.jmx.enabled=true" \
--set "kafka.metrics.serviceMonitor.enabled=true" \
--set "kafka.metrics.serviceMonitor.namespace=monitor"

```

## 메모

- export-promethues-grafana를 하나의 stack으로 묶어 배포되는 헬름차트들이 여럿 있음
- 대부분, 단일 클러스터 내에서 사용하는 것을 가정하여 만들어짐
- 멀티클러스터에 대해 Centralized 모니터링을 하려면 수정이 필요
- prometheus를 각 서비스 클러스터에 둘 것인가, 중앙모니터링 서버에 둘 것인가.
  - 서비스에 두면 메모리 부담, 중앙에 두면 더 복잡한 설정 필요
- 다른 Centralized 모니터링 예시들이 있음
  - 보통은 각 서비스, 각 클러스터에 prometheus를 배치시키고, 중앙에는 Thanos라는 툴을 둠
  - prometheus는 클러스터링을 지원하지않는데, 여러 promethues를 하나의 앱처럼 관리하기 위한 툴이 Thanos임
- 중앙에 단일 prometheus 두고 가벼운 모니터링시스템으로 쓰다가 나중에 필요할 때 업글할까? 아니면 처음부터 체계적으로 구축할까도 고민. 급하지는 않은데...
- 하나의 Prometheus에 여러 grafana가 접근가능할 듯한데, 굳이 Thanos가 필요할까???
- 하나의 exporter에 여러 prometheus가 붙을 수 있나? => 이거 포트가 점유되어있다고 안될거같은데...

## 주의

- WSL에서 configmap을 읽지 못하는 버그가 있음.

- EC2에서는 정상실행되나, 껐다 켰다 반복시 grafana Pod에서 동일한 failed Event 발생

- volume 관련 설정시 비슷한 에러가 발생
  - 이 땐, k3s를 재시작하면 해결됨

```sh
"MountVolume.SetUp failed for volume ...
```

## Service Monitor

- prometheus-opreator에서 제공하는 CRD
- Service Monitor는 Prometheus가 Kubernetes 서비스를 자동으로 발견하고, 이 서비스들이 노출하는 메트릭을 수집할 수 있게 해준다.
- 원래 Prometheus에서 읽을 exporter정보를 `prometheus.yaml`(`prometheusSpec`)에 수동설정해야 하는데 Service Monitor를 통해 이를 자동화해준다.
- Service Monitor는 모니터링 대상 서비스에서 on-off하는 옵션이다.
  - 단, Service Monitor는 단일 K8s 클러스터 내에서 동작하는 것을 목적으로 만들어졌기 때문에, 원격 환경에선 적합하지 않다.
- 이 헬름 차트 외에도 prometheus 관련 기능을 제공하는 차트에서 Service Monitor 옵션이 종종 등장한다.
- 다시 읽어보기: [Service Monitor 컨셉, 쓰는 이유, 장점](https://jerryljh.medium.com/prometheus-servicemonitor-98ccca35a13e)

## Sidecar

- 메인컨테이너와 함께 동작하는 보조컨테이너(helper container)를 의미
- 메인컨테이너 앱의 주 기능 외에 로그처리, 모니터링, 보안, 프록시, 파일 시스템 관리 등의 용도로 사용
- 디커플링, 재사용성, 유지보수성에 유리

## Alert

흔히 알려진 AlertManager는 Prometheus의 기능이고, Prometheus 본 앱과 별도로 실행되는 앱이다.

### grafana alerting 특징

- prometheus alertmanager와도 기능이 겹침. grafana stack(loki) 쓸 땐 좋음
- prometheus 외 다양한 datasource에 대해 알람가능
- 장점
  - 멀티 클러스터 모니터링시 중앙에서 관리가능
  - 프로비전 뿐 아니라, UI에서도 간단히 설정가능하다.
  - Alertmanger보다 간소하다고하지만, 메트릭이 '임계값을 초과하고 특정 시간동안 지속'되는 것을 조건으로 알람하는 수준은 충분히 가능. SMTP, MS팀즈 전달도 가능
- 단점
  - Prometheus AlertManager 보다는 약한 기능
  - 버전마다 기능이 급변하는 중
  - 대시보드 패널에 의존적

### promethues alertmanager 특징

- 장점
  - 널리 사용되어와서, 안정적
  - 여러 메트릭 복합 조건으로 트리거 가능
  - 특정 시간대만 검사해서 Alert 발생 등 가능
  - => 근데 왜 kps차트는 grafana쪽 설정에 alertmanager가 있지???
  - => 이거는 datasource로 prometheus의 alertmanager를 추가하기 위한 용도다.
  - => Grafana에서 AlertManager를 Datasource로 추가하는 것은, Alertmanager에서 생성된 알림을 Grafana 대시보드에서 시각화하고 관리하기 위함(threshold를 넘어선 횟수, 내역 등 체크)
- 단점
  - UI에서 규칙편집이 안됨. 그냥 현 상황 조회하는 정도만 가능.
  - 고도화된 모니터링이 필요없다면 관리비용만 늘어남
  - 멀티 클러스터를 모니터링할 때는 개별 네트워크 인가가 필요할 수 있음
    - 단, AlertManager의 모든 지표는 Prometheus가 수집가능
  - 클러스터마다 설정도 달라질 수 있기 때문에 관리가 불편해질 수 있음
- kps차트에서 alert 관련기능은 모두 prometheus alertmanager의 rule을 설정하는 방식으로 관리되는 것 같다.
- grafana의 alert 기능을 쓰고 싶으면, grafana의 subchart value or grafana ui를 활용

## EKS 에서 권한(Role)문제

- 권한 관련 문제는 환경 제공자마다 다르게 나타나는 것 같다.
- GKE는 권한 최적화해서 해결하는 방법이 있다는 것 같은데, EKS는 그냥 관리자 권한 전체 부여해야할듯
- [참고](https://github.com/prometheus-operator/prometheus-operator/issues/1189)

```sh
# clusterrole 중에 관리자 권한(cluster-admin)이 기본으로 있고, 이걸 사용하는 user에게 clusterbinding으로 연결해준다.
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=your.email@address.com 
kubectl create clusterrolebinding yunan-cluster-admin-binding --clusterrole=cluster-admin --user=yunan_all
# --set "prometheus.prometheusSpec.nodeSelector.wai-eks/noderole=home" \
```

## PV 삭제 안됨(STATUS=TERMINATING에서 멈춤 현상)

- pv는 namespace가 상관없지만, pvc는 헬름 설치시 선택한 namespace에 있다.
- pvc를 삭제해야 한다.

```sh
kubectl delte pvc {PVC_NAME} -n {NAMESPACE}
```

- 그래도 안되는 경우 다음 명령어를 쓰면 pv 삭제 가능

```sh
kubectl patch pv {PV이름} -p '{"metadata":{"finalizers":null}}'
```
