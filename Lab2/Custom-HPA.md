## 基于定制Metrics实现HPA

### 1. 部署Prometheus Adapter

由于OKE默认没有安装Promethes,所以这里全量按照Prometheus相关组件，包括Prometheus、prometheus-adapter、alertmanager、grafana等

```
$ git clone https://github.com/prometheus-operator/kube-prometheus.git

$ cd kube-prometheus/

$ kubectl create -f manifests/setup

$ until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done

$ kubectl create -f manifests/

$ kubectl -n monitoring get pods

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

部署sample-app应用

```
$ kubectl create -f sample-app.deploy.yaml
$ kubectl create -f sample-app.service.yaml
```
测试验证程序正常运行

```
$ kubectl create -f test.yaml
$ kubectl get pods -o wide
$ kubectl exec php-apache-xxxx -i -t -- /bin/sh

sh-4.4# curl http://10.244.1.6:8080/metrics
# HELP http_requests_total The amount of requests served by the server in total
# TYPE http_requests_total counter
http_requests_total 1
```


### 3. 监控sample-app

```
$ kubectl create -f sample-app.monitor.yaml
```

### 4. 更新Prometheus Adapter配置

```
$ kubectl apply -f prom-adapter.config.yaml

// Restart prom-adapter pods
$ kubectl rollout restart deployment prometheus-adapter -n monitoring
```

### 5. 将prometheus-adapter注册为定制指标API服务

```
$ kubectl create -f api-service.yaml
```

验证是否已经成功注册

```
$ kubectl get --raw /apis/custom.metrics.k8s.io/v1beta2 | python -m json.tool | grep "http_request"

...
            "name": "pods/http_requests",
...
```

可以查看当前`sample-app` `pod`的监控指标(`http_requests`)

```
$ kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta2/namespaces/default/pods/*/http_requests?selector=app%3Dsample-app"

{"kind":"MetricValueList","apiVersion":"custom.metrics.k8s.io/v1beta2","metadata":{},"items":[{"describedObject":{"kind":"Pod","namespace":"default","name":"sample-app-7bdd9fb7dc-b6cjx","apiVersion":"/v1"},"metric":{"name":"http_requests","selector":null},"timestamp":"2023-03-02T06:07:54Z","value":"66m"}]}
```


### 6. HPA配置

创建HPA配置

```
$ kubectl create -f sample-app.hpa.yaml
```

提高HTTP请求频率

```
$ kubectl get pods -o wide
$ kubectl exec php-apache-xxxx -i -t -- /bin/sh

sh-4.4# while true; do curl http://10.244.1.6:8080/metrics; sleep 0.5; done
...
# HELP http_requests_total The amount of requests served by the server in total
# TYPE http_requests_total counter
http_requests_total 12
...
```

观察HPA效果

```
$ kc get pods -w
NAME                          READY   STATUS    RESTARTS   AGE
php-apache-5c6bb5b468-5x4vz   1/1     Running   0          3m36s
sample-app-7bdd9fb7dc-b6cjx   1/1     Running   0          11m
sample-app-7bdd9fb7dc-n4zhz   1/1     Running   0          16s
sample-app-7bdd9fb7dc-rfvb4   0/1     Pending   0          0s
sample-app-7bdd9fb7dc-rfvb4   0/1     Pending   0          0s
sample-app-7bdd9fb7dc-k8qj2   0/1     Pending   0          0s
sample-app-7bdd9fb7dc-k8qj2   0/1     Pending   0          0s
sample-app-7bdd9fb7dc-rfvb4   0/1     ContainerCreating   0          0s
sample-app-7bdd9fb7dc-k8qj2   0/1     ContainerCreating   0          0s
sample-app-7bdd9fb7dc-k8qj2   1/1     Running             0          1s
sample-app-7bdd9fb7dc-rfvb4   1/1     Running             0          8s

$ kc get hpa
NAME         REFERENCE               TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
sample-app   Deployment/sample-app   1049m/500m   1         10        4          11m
```

