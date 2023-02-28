

### 1. 环境准备

下载istio

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.17.1 TARGET_ARCH=x86_64 sh -
export PATH="$PATH:/root/istio-1.17.1/bin"
```
安装Helm

```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

通过Helm安装Prometheus-Adaptor

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install my-release prometheus-community/prometheus-adapter
kubectl get --raw /apis/custom.metrics.k8s.io/v1beta1 | python -m json.tool
```

部署Nginx Ingress Controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml
```

获取OKE集群的访问配置

```
oci ce cluster create-kubeconfig --cluster-id ocid1.cluster.oc1.ap-tokyo-1.XXXXX --file $HOME/.kube/config --region ap-tokyo-1 --token-version 2.0.0  --kube-endpoint PUBLIC_ENDPOINT
```

下载Kubectl

```
curl -LO https://dl.k8s.io/release/v1.25.4/bin/linux/amd64/kubectl
chmod +x kubectl
cp kubectl /usr/bin/
```

验证集群与环境是否就绪

```
kc get node
istioctl x precheck
```

开始部署istio

```
kc create ns istio-system
istioctl install --set components.cni.enabled=true
kc -n istio-system get pods
```

部署其周边组件

```
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/grafana.yaml
kubectl apply -f samples/addons/jaeger.yaml
kubectl apply -f samples/addons/kiali.yaml
```

使能istio自动注入

```
kubectl label namespace default istio-injection=enabled
```

部署示例应用

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo-ingress.yaml
```

访问应用

```
# 获取istio gateway 公网IP地址
[root@osstest2 istio-1.17.1]# kc -n istio-system get svc istio-ingressgateway
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                      AGE
istio-ingressgateway   LoadBalancer   10.96.126.197   140.83.33.77   15021:30306/TCP,80:32192/TCP,443:31359/TCP   11h

# bookinfo的访问URL为：http://140.83.33.77/productpage
```

![image](https://user-images.githubusercontent.com/4653664/221776438-b34f3e86-dbd2-4adc-ae77-a15bf02a97b7.png)

访问Kiali服务

```
# 修改type: LoadBalancer
[root@osstest2 istio-1.17.1]# kc -n istio-system edit svc kiali

[root@osstest2 istio-1.17.1]# kc -n istio-system get svc kiali
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)                          AGE
kiali   LoadBalancer   10.96.14.110   155.248.179.191   20001:30608/TCP,9090:32163/TCP   11h

# Kiali服务地址：http://155.248.179.191:20001
```

![image](https://user-images.githubusercontent.com/4653664/221778563-6b9bbf45-3864-4d4b-aecf-be847d1ccb00.png)

访问grafana服务

```
# 修改type: LoadBalancer
[root@osstest2 istio-1.17.1]# kc -n istio-system edit svc kiali

[root@osstest2 istio-1.17.1]# kc -n istio-system get svc grafana
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
grafana   LoadBalancer   10.96.140.128   131.186.33.86   3000:31311/TCP   12h

# Grafana访问地址：131.186.33.86：3000
```
![image](https://user-images.githubusercontent.com/4653664/221782812-61002696-b133-4897-8462-f0cd38ef4527.png)


日志系统接入(使用OCI Logging)

创建动态组

```
instance.compartment.id='ocid1.compartment.oc1..aaaaaaaafqgbjumv6djs3xdbkq27gat2nyhtowzdfltiy42w2rthjuvpl46a'
```

创建policy

```
Allow dynamic-group applog to use log-content in compartment k8s
```

创建名为oke-app-logs的`Log Group`

![image](https://user-images.githubusercontent.com/4653664/221799662-3852ed43-d252-4794-b631-e64f39b2fe64.png)

创建名为oke-app-log的定制log

![image](https://user-images.githubusercontent.com/4653664/221799953-3ab57c48-1126-4ce5-985c-882c91355dbd.png)

创建名为oke-app-log的`agent config`

![image](https://user-images.githubusercontent.com/4653664/221800460-9449f78f-0f0f-4f2a-aabf-220a79ba31eb.png)

查看`OCI Logging Search`

![image](https://user-images.githubusercontent.com/4653664/221805362-df9a9381-fb88-4410-807f-d24f548b2559.png)


