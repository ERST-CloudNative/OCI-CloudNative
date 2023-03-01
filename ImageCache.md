镜像缓存实践


1. 下载项目代码仓库

```
[root@osstest2 ~]# git clone https://github.com/senthilrch/kube-fledged.git $HOME/kube-fledged
Cloning into '/root/kube-fledged'...
remote: Enumerating objects: 10497, done.
remote: Counting objects: 100% (1385/1385), done.
remote: Compressing objects: 100% (617/617), done.
remote: Total 10497 (delta 777), reused 1233 (delta 666), pack-reused 9112
Receiving objects: 100% (10497/10497), 34.56 MiB | 16.13 MiB/s, done.
Resolving deltas: 100% (4363/4363), done.
```

2. 开始部署
```
[root@osstest2 ~]# cd kube-fledged/

[root@osstest2 kube-fledged]# make deploy-using-yaml

kubectl apply -f deploy/kubefledged-namespace.yaml
namespace/kube-fledged created
kubectl apply -f deploy/kubefledged-crd.yaml
customresourcedefinition.apiextensions.k8s.io/imagecaches.kubefledged.io created
kubectl apply -f deploy/kubefledged-serviceaccount-controller.yaml
serviceaccount/kubefledged-controller created
kubectl apply -f deploy/kubefledged-clusterrole-controller.yaml
clusterrole.rbac.authorization.k8s.io/kubefledged-controller created
kubectl apply -f deploy/kubefledged-clusterrolebinding-controller.yaml
clusterrolebinding.rbac.authorization.k8s.io/kubefledged-controller created
kubectl apply -f deploy/kubefledged-deployment-controller.yaml
deployment.apps/kubefledged-controller created
kubectl rollout status deployment kubefledged-controller -n kube-fledged --watch
Waiting for deployment "kubefledged-controller" rollout to finish: 0 of 1 updated replicas are available...
deployment "kubefledged-controller" successfully rolled out
kubectl delete validatingwebhookconfigurations -l app=kubefledged
No resources found
kubectl apply -f deploy/kubefledged-validatingwebhook.yaml
validatingwebhookconfiguration.admissionregistration.k8s.io/kubefledged-webhook-server created
kubectl delete deploy -l app=kubefledged,kubefledged=kubefledged-webhook-server
No resources found
kubectl apply -f deploy/kubefledged-serviceaccount-webhook-server.yaml
serviceaccount/kubefledged-webhook-server created
kubectl apply -f deploy/kubefledged-clusterrole-webhook-server.yaml
clusterrole.rbac.authorization.k8s.io/kubefledged-webhook-server created
kubectl apply -f deploy/kubefledged-clusterrolebinding-webhook-server.yaml
clusterrolebinding.rbac.authorization.k8s.io/kubefledged-webhook-server created
kubectl apply -f deploy/kubefledged-deployment-webhook-server.yaml
deployment.apps/kubefledged-webhook-server created
kubectl apply -f deploy/kubefledged-service-webhook-server.yaml
service/kubefledged-webhook-server created
kubectl rollout status deployment kubefledged-webhook-server -n kube-fledged --watch
Waiting for deployment "kubefledged-webhook-server" rollout to finish: 0 of 1 updated replicas are available...
deployment "kubefledged-webhook-server" successfully rolled out
kubectl get pods -n kube-fledged
NAME                                          READY   STATUS    RESTARTS   AGE
kubefledged-controller-55f848cc67-2pnbz       1/1     Running   0          61s
kubefledged-webhook-server-597dbf4ff5-grf98   1/1     Running   0          22s
```

3. 验证是否部署成功

```
[root@osstest2 kube-fledged]# kubectl get imagecaches -n kube-fledged
No resources found in kube-fledged namespace.
```

4. 创建imagecache

```
[root@osstest2 kube-fledged]# cat deploy/kubefledged-imagecache.yaml
---
apiVersion: kubefledged.io/v1alpha2
kind: ImageCache
metadata:
  # Name of the image cache. A cluster can have multiple image cache objects
  name: imagecache1
  namespace: kube-fledged
  # The kubernetes namespace to be used for this image cache. You can choose a different namepace as per your preference
  labels:
    app: kubefledged
    kubefledged: imagecache
spec:
  # The "cacheSpec" field allows a user to define a list of images and onto which worker nodes those images should be cached (i.e. pre-pulled).
  cacheSpec:
  # Specifies a list of images (nginx:1.23.1) with no node selector, hence these images will be cached in all the nodes in the cluster
  - images:
    - ghcr.io/jitesoft/nginx:1.23.1
  # Specifies a list of images (cassandra:v7 and etcd:3.5.4-0) with a node selector, hence these images will be cached only on the nodes selected by the node selector
  - images:
    - us.gcr.io/k8s-artifacts-prod/cassandra:v7
    - us.gcr.io/k8s-artifacts-prod/etcd:3.5.4-0
    nodeSelector:
      tier: backend
  # Specifies a list of image pull secrets to pull images from private repositories into the cache
  imagePullSecrets:
  - name: myregistrykey
  
```
基于以上配置，镜像缓存情况如下
`ghcr.io/jitesoft/nginx:1.23.1`，默认在没有`tier: backend`标签的节点都会缓存

```
us.gcr.io/k8s-artifacts-prod/cassandra:v7
us.gcr.io/k8s-artifacts-prod/etcd:3.5.4-0
```
以上两个镜像仅在有`tier: backend`标签的节点上缓存

```
  imagePullSecrets:
  - name: myregistrykey
```
这里配置的是私有镜像仓库的`secret`


接下来创建一个示例`ImageCache`

```
[root@osstest2 kube-fledged]# kubectl create -f deploy/kubefledged-imagecache.yaml
imagecache.kubefledged.io/imagecache1 created
```

5. 查看`ImageCache`

```
[root@osstest2 kube-fledged]# kubectl get imagecaches -n kube-fledged
NAME          AGE
imagecache1   17s
[root@osstest2 kube-fledged]# kubectl get imagecaches imagecache1 -n kube-fledged -o json
{
    "apiVersion": "kubefledged.io/v1alpha2",
    "kind": "ImageCache",
    "metadata": {
        "creationTimestamp": "2023-03-01T08:51:47Z",
        "generation": 3,
        "labels": {
            "app": "kubefledged",
            "kubefledged": "imagecache"
        },
        "name": "imagecache1",
        "namespace": "kube-fledged",
        "resourceVersion": "619584",
        "uid": "e39769d5-2f87-4260-a52c-e5288d272b2d"
    },
    "spec": {
        "cacheSpec": [
            {
                "images": [
                    "ghcr.io/jitesoft/nginx:1.23.1"
                ]
            },
            {
                "images": [
                    "us.gcr.io/k8s-artifacts-prod/cassandra:v7",
                    "us.gcr.io/k8s-artifacts-prod/etcd:3.5.4-0"
                ],
                "nodeSelector": {
                    "tier": "backend"
                }
            }
        ],
        "imagePullSecrets": [
            {
                "name": "myregistrykey"
            }
        ]
    },
    "status": {
        "completionTime": "2023-03-01T08:52:05Z",
        "message": "All requested images pulled succesfully to respective nodes",
        "reason": "ImageCacheCreate",
        "startTime": "2023-03-01T08:51:47Z",
        "status": "Succeeded"
    }
}
```

6. `ImageCache`刷新

```
[root@osstest2 kube-fledged]# kubectl annotate imagecaches imagecache1 -n kube-fledged kubefledged.io/refresh-imagecache=
imagecache.kubefledged.io/imagecache1 annotated
[root@osstest2 kube-fledged]# kubectl get imagecaches imagecache1 -n kube-fledged -o json
{
    "apiVersion": "kubefledged.io/v1alpha2",
    "kind": "ImageCache",
    "metadata": {
        "creationTimestamp": "2023-03-01T08:51:47Z",
        "generation": 5,
        "labels": {
            "app": "kubefledged",
            "kubefledged": "imagecache"
        },
        "name": "imagecache1",
        "namespace": "kube-fledged",
        "resourceVersion": "619918",
        "uid": "e39769d5-2f87-4260-a52c-e5288d272b2d"
    },
    "spec": {
        "cacheSpec": [
            {
                "images": [
                    "ghcr.io/jitesoft/nginx:1.23.1"
                ]
            },
            {
                "images": [
                    "us.gcr.io/k8s-artifacts-prod/cassandra:v7",
                    "us.gcr.io/k8s-artifacts-prod/etcd:3.5.4-0"
                ],
                "nodeSelector": {
                    "tier": "backend"
                }
            }
        ],
        "imagePullSecrets": [
            {
                "name": "myregistrykey"
            }
        ]
    },
    "status": {
        "completionTime": "2023-03-01T08:53:16Z",
        "message": "All requested images pulled succesfully to respective nodes",
        "reason": "ImageCacheRefresh",
        "startTime": "2023-03-01T08:53:15Z",
        "status": "Succeeded"
    }
}
```
7. 删除`ImageCache`

```
[root@osstest2 kube-fledged]# kubectl annotate imagecaches imagecache1 -n kube-fledged kubefledged.io/purge-imagecache=
imagecache.kubefledged.io/imagecache1 annotated
```
