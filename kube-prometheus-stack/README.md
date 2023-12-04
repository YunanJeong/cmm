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
--set "kafka.metrics.jmx.enabled=true"

```

## 주의

WSL에서 configmap을 읽지 못하는 버그가 있음.

```sh
"MountVolume.SetUp failed for volume ...
```
