# [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack?modal=install)

- node-exporter, promethues, promethues-operator, grafana로 구성된 차트다.
- 사람들이 많이 쓰는 조합이기 때문에 이미 통합 차트로 만들어진 사례가 있으며, 그 중에서도 promethues 커뮤니티에서 나온 kube-promethues-stack이 종종 사용된다.
  - 근데 설정이 좀 과하긴하다. value파일 4천라인쯤 됨
- grafana에서 출시한 k8s-monitoring 차트도 있는데, 천 라인쯤 됨
  - 모든 기능을 커스텀할 게 아니라면 차라리 따로따로 되있는걸 커스텀으로 합치는게 더 편할 것 같기도..
  - 일단 좀 봐야 알겠다.

- 구 차트이름이 prometheus-opeator였으나, 변경됨. 현재는 일부 구성 요소를 prometheus-operator라고 부름

```sh
# Add Repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
# 기본 설치
helm install prostack prometheus-community/kube-prometheus-stack --version 54.2.2
# value 포함 후 monitor ns에설치
helm install prostack prometheus-community/kube-prometheus-stack --version 54.2.2 -n monitor -f myvalue.yaml
# 업글
helm upgrade prostack prometheus-community/kube-prometheus-stack --version 54.2.2 -n monitor -f myvalue.yaml

# jmx, kafka exporter 테스트용 앱
helm install test https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/skafka-2.0.3.tgz \
-f https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/kraft-multi.yaml \
--set "kafka.externalAccess.autoDiscovery.enabled=false" \
--set "kafka.externalAccess.controller.service.nodePorts={30003,30004,30005}" \
--set "kafka.metrics.jmx.enabled=true"

helm upgrade test https://github.com/YunanJeong/simple-kafka-deploy/releases/download/v2.0.3/skafka-2.0.3.tgz \
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

## 주의

- WSL에서 configmap을 읽지 못하는 버그가 있음.

- EC2에서는 정상실행되나, 껐다 켰다 반복시 grafana Pod에서 동일한 failed Event 발생

```sh
"MountVolume.SetUp failed for volume ...
```
