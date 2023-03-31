Calico component metrics
===
https://github.com/prometheus-community/helm-charts/

```bash
kubectl patch felixconfiguration default --type merge -p '{"spec":{"prometheusMetricsEnabled": true}}'
```

If you are following the tutorial on Windows use the following command in PowerShell: 
```powershell
kubectl patch felixconfiguration default --type merge -p '{\"spec\":{\"prometheusMetricsEnabled\": true}}'
```

```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/05.monitoring/01.felix-svc.yaml
```

```bash
kubectl patch felixconfiguration default --type merge -p '{"spec":{"prometheusMetricsEnabled": true}}'
```

If you are following the tutorial on Windows use the following command in PowerShell: 
```powershell
kubectl patch installation default --type=merge -p '{\"spec\": {\"typhaMetricsPort\":9093}}'
```

```
kubectl create -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/05.monitoring/02.typha-svc.yaml
```

Kubernetes node metrics
===

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

```
docker pull quay.io/prometheus/node-exporter:v1.5.0 
docker tag quay.io/prometheus/node-exporter:v1.5.0 $REGISTRY/prometheus/node-exporter:v1.5.0
docker push $REGISTRY/prometheus/node-exporter:v1.5.0
```

```
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --set=image.registry=private-repo:5000 --namespace calico-monitoring
```

https://docs.tigera.io/calico/latest/operations/monitor/

```
docker pull prom/prometheus:latest
docker tag prom/prometheus:latest $REGISTRY/prom/prometheus:latest
docker push $REGISTRY/prom/prometheus:latest
```

Monitoring all the metrics with Prometheus
===

```
kubectl apply -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/05.monitoring/03.prometheus-config.yaml
```

```
kubectl apply -f https://raw.githubusercontent.com/frozenprocess/Tigera-Presentations/master/2023-03-30.container-and-Kubernetes-security-policy-design/05.monitoring/04.prometheus-pod.yaml
```

Use the following command and then fire a browser and point it to <a href="http://localhost:9090" target="_blank">http://localhost:9090</a>:
```
kubectl port-forward pod/prometheus-pod 9090:9090 -n calico-monitoring
```
https://www.tigera.io/tutorials/?_sf_s=Calico%20eBPF%20and%20XDP

> **Note:** Click <a href="https://docs.tigera.io/calico/latest/operations/monitor/monitor-component-visual" target="_blank">here</a> if you like to learn how to create dashboards and other sort of visualizations from these metrics.


Congratulations!
