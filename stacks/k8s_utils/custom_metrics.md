# Kubernetes Custom Metrics

```bash
# https://dustinspecker.com/posts/scaling-kubernetes-pods-prometheus-metrics/
# https://github.com/prometheus-operator/kube-prometheus
```

# Installing the Kubernetes Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

helm install prometheus-operator -f prometheus-operator-value.yaml stable/prometheus-operator

## Add Prometheus Helm Repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo update

helm search repo prometheus-community
```

```bash
https://github.com/prometheus-operator/prometheus-operator
```

## Begin Installation

```bash
kubectl create namespace monitoring

git clone https://github.com/prometheus-operator/kube-prometheus kube-prometheus
cd kube-prometheus

# Create the monitoring stack using the config in the manifests directory:
# Create the namespace and CRDs, and then wait for them to be available before creating the remaining resources
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/



# https://github.com/prometheus-operator/kube-prometheus#quickstart
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring

helm install prometheus-adapter prometheus-community/prometheus-adapter --namespace monitoring

# helm install prometheus-adapter prometheus-community/prometheus-adapter --namespace monitoring -f adapter-values.yaml
```

### Customizing Prometheus Installation

```bash
helm install prometheus-adapter prometheus-community/prometheus-adapter \
    --set prometheus.url=http://prometheus-server, prometheus.port=9090 \
    --set rbac.create="true" \
    --namespace monitoring

helm install prometheus-adapter prometheus-community/prometheus-adapter \
    --set prometheus.url="http://prometheus-server.prometheus.svc.cluster.local",prometheus.port="9090" \
    --set rbac.create="true" \
    --namespace monitoring
```

### Check Roll Out Status

```bash
kubectl rollout status deploy/prometheus-adapter --namespace monitoring
```

### Register Custom Metrics API

```bash
kubectl apply -f custom-metrics-apiservice.yaml
```

### Apply Cluster Role RBAC to monitor other namespaces

```bash
kubectl apply -f k8s_cluster_role_rbac_perms.yaml
```

### Add Custom Metrics to Prometheus Adapter configmap

```bash
# kubectl edit cm prometheus-adapter --namespace monitoring
kubectl apply -f custom-metrics-config.yaml
```

#### Recycle Adapter Pods

```bash
# kubectl delete pod --selector app=prometheus-adapter --namespace monitoring
kubectl delete pod --namespace monitoring --selector app.kubernetes.io/name=prometheus-adapter
```

### Verify API

```bash
(.venv) manifests]# kubectl get apiservices v1beta1.metrics.k8s.io
NAME                     SERVICE                      AVAILABLE   AGE
v1beta1.metrics.k8s.io   kube-system/metrics-server   True        41h
(.venv) manifests]# kubectl get apiservices v1beta1.custom.metrics.k8s.io
NAME                            SERVICE                         AVAILABLE   AGE
v1beta1.custom.metrics.k8s.io   monitoring/prometheus-adapter   True        7m56s

kubectl describe apiservice v1beta1.custom.metrics.k8s.io

kubectl api-resources | grep metrics.k8s.io
```

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/\*/http*requests_total" | jq .
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/http_requests_total" | jq .

kubectl get --raw '/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pod/podinfo-64684fdb5b-26lrl/http_requests_per_second' | jq .

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/\*/http_requests_per_second"

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/podinfo/http_requests" | jq .

kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/_/http_requests" | jq .
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/monitoring/pods/_/container_network_receive_bytes_per_second

#### Explore Resource Metrics for nodes and pods

```bash
# Get more metrics about nodes

kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq

# Get more details about pods

kubectl get --raw /apis/metrics.k8s.io/v1beta1/pods | jq
```

### Deploy Shopfront

```bash
kubectl apply -f miztiik-automation-ns.yml

kubectl apply -f shopfront.yml

# Wait for few minutes
# Check custom metrics API
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/" |jq
```

### Access Prometheus GUI

```bash
# kubectl --namespace monitoring port-forward svc/kube-prometheus-stack-prometheus 9090 &
kubectl --namespace monitoring port-forward svc/prometheus-k8s 9090 &
```

### Check the Custom Metrics API for shopfront Metrics

```bash
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/miztiik-automation-ns/pods/*/http_requests_per_second | jq

kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1/namespaces/miztiik-automation-ns/pods/*/http_requests_per_second?labelSelector=app%3Dshopfront | jq
```

### Generate Traffic against the service

```bash
SVC_LB_URL=$(kubectl get svc podinfo -o jsonpath='{.status.loadBalancer.ingress[].hostname}')

SVC_LB_URL=$(kubectl get svc shopfront-svc -n miztiik-automation-ns -o jsonpath='{.status.loadBalancer.ingress[].hostname}')
echo $SVC_LB_URL
kubectl run -i --tty load-generator -n miztiik-automation-ns --rm --image=busybox --restart=Never -- /bin/sh -c "while sleep 0.05; do wget -q -O- http://$SVC_LB_URL; done"
# After HPA reaches MAXPODS kill the load generator, HPA will scale down to a single pod
```

```bash
(.venv) manifests]# kubectl get hpa shopfront-hpa -n miztiik-automation-ns --watch
NAME      REFERENCE            TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
podinfo   Deployment/podinfo   400m/2    1         6         1          55m
podinfo   Deployment/podinfo   417m/2    1         6         1          57m
podinfo   Deployment/podinfo   608m/2    1         6         1          58m
podinfo   Deployment/podinfo   1130m/2   1         6         1          58m
podinfo   Deployment/podinfo   1617m/2   1         6         1          58m
podinfo   Deployment/podinfo   2095m/2   1         6         1          58m
podinfo   Deployment/podinfo   2356m/2   1         6         1          59m
podinfo   Deployment/podinfo   1320m/2   1         6         2          59m
podinfo   Deployment/podinfo   1637m/2   1         6         2          59m
podinfo   Deployment/podinfo   1885m/2   1         6         2          59m
podinfo   Deployment/podinfo   1980m/2   1         6         2          60m
podinfo   Deployment/podinfo   2026m/2   1         6         2          60m
podinfo   Deployment/podinfo   2053m/2   1         6         2          60m
podinfo   Deployment/podinfo   2062m/2   1         6         2          60m
podinfo   Deployment/podinfo   2249m/2   1         6         2          61m
podinfo   Deployment/podinfo   2295m/2   1         6         3          61m
podinfo   Deployment/podinfo   1569m/2   1         6         3          61m
podinfo   Deployment/podinfo   1646m/2   1         6         3          61m
```
