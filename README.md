# MMM: Multi-cluster Metrics Monitroing

Centralized Monitoring for Multi-cluster Metrics

그라파나 테스트. helm으로 커스텀 대시보드까지 빠르게 이식 가능한 구조 만들기

일단 bitnami/grafana로 시작

## grafana 설치

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-grafana bitnami/grafana

# grafana pod의 3000번에 포트포워딩 후 브라우저 접속가능(kubectl port-forward 커맨드, ingress, k9s 포트포워딩 등 사용)
# Pod 실행 후 5분 정도는 지나야 접속가능 상태가 된다.
kubectl port-forward svc/my-grafana 8080:3000 &

# not found socat 발생시
sudo apt install -y socat
```

## prometheus 설치

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install my-prometheus bitnami/prometheus


```
