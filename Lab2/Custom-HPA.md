## 基于定制Metrics实现HPA

### 1. 部署Prometheus Adapter

由于OKE默认没有安装Promethes,所以这里全量按照Prometheus相关组件，包括Prometheus、prometheus-adapter、alertmanager、grafana等

```
# git clone https://github.com/prometheus-operator/kube-prometheus.git

# cd kube-prometheus/

# kubectl create -f manifests/setup

# until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done

# kubectl create -f manifests/

# kubectl -n monitoring get pods

NAME                                   READY   STATUS    RESTARTS       AGE
alertmanager-main-0                    2/2     Running   0              100m
alertmanager-main-1                    2/2     Running   1 (100m ago)   100m
alertmanager-main-2                    2/2     Running   1 (100m ago)   100m
blackbox-exporter-6fd586b445-g9fn7     3/3     Running   0              101m
grafana-5656dfff76-jd9pz               1/1     Running   0              101m
kube-state-metrics-787856b689-9mgn7    3/3     Running   0              101m
node-exporter-8m74m                    2/2     Running   0              101m
node-exporter-fg2q2                    2/2     Running   0              101m
node-exporter-xpdm9                    2/2     Running   0              101m
prometheus-adapter-5985b74d7-k4bsp     1/1     Running   0              55m
prometheus-adapter-5985b74d7-rlvx9     1/1     Running   0              55m
prometheus-k8s-0                       2/2     Running   0              100m
prometheus-k8s-1                       2/2     Running   0              100m
prometheus-operator-65ff8b668d-r9q42   2/2     Running   0              101m
```


### 2. 部署应用

```
# kubectl create -f sample-app.deploy.yaml
# kubectl create -f sample-app.service.yaml
```

```
# kubectl exec php-apache-5c6bb5b468-5x4vz -i -t -- /bin/sh
sh-4.4# curl http://10.244.1.6:8080/metrics
# HELP http_requests_total The amount of requests served by the server in total
# TYPE http_requests_total counter
http_requests_total 1
```

