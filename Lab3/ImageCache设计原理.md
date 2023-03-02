### 1.问题陈述

当 Pod 成功调度到 k8s 工作节点时，Pod 所需的容器镜像将从镜像注册表中拉取。根据镜像大小和网络延迟，镜像拉取会消耗相当长的时间。最终，这会增加启动容器和 Pod 达到就绪状态所需的时间。

有几个用例希望 Pod 立即启动并运行。拉取容器镜像所花费的大量时间成为实现这一目标的主要瓶颈。在某些用例中，与镜像注册表的连接可能并非始终可用（例如，边缘节点在移动的游轮上运行的物联网/边缘计算）。

如果需要从私有注册表中拉取镜像，并且不能授予每个人从该注册表中拉取镜像的权限，则可以在集群的节点上提供这些镜像。这样，对这些镜像的任何拉取请求都不会进入注册表，而是使用节点中的镜像（如果 imagepullPolicy=IfNotPresent）。

### 2. 现有解决方案

解决 Pod 启动时间减少问题的现有解决方案是在 Cluster 内部运行一个 Registry 镜像。第一次从本地注册表镜像请求镜像时，它会从主镜像注册表中拉取镜像并将其存储在本地，然后再发送给客户端。在后续请求中，本地注册表镜像能够从其自己的存储中提供镜像。参考这个网址

两种广泛使用的解决方案是 i) 集群内自托管注册表 ii) pull-through 缓存。在前一种解决方案中，本地注册表在 k8s 集群中运行，并在容器运行时配置为镜像注册表。任何镜像拉取请求都会定向到集群内注册表。如果失败，请求将被定向到主要注册表。在后一种解决方案中，本地注册表具有缓存功能。第一次拉取镜像时，会缓存在本地registry中。对镜像的后续请求由本地注册表提供服务。

### 3.现有解决方案的缺点

设置和维护本地注册表镜像会消耗大量的计算资源和人力资源。
对于跨越多个区域的庞大集群，我们需要有多个本地注册表镜像。当应用程序实例跨越多个区域时，这会引入不必要的复杂性。您可能需要多个部署清单，每个清单都指向该区域的本地注册表镜像。
这些方法并没有完全解决实现快速启动 Pod 的需求，因为从本地镜像中拉取镜像仍然存在明显的延迟。有几个用例不能容忍这种延迟。
节点可能会失去与本地注册表镜像的网络连接，因此 Pod 将被卡住，直到连接恢复。

### 4. 建议的解决方案 - 集群镜像缓存

建议的解决方案是拥有集群镜像缓存。镜像缓存分布在所有/多个工作节点上，而不是集中在本地存储库镜像中。需要即时 Pod 启动或不能容忍丢失与镜像注册表的连接的应用程序会将容器镜像存储在集群镜像缓存中，并直接在节点中提供。当 Pod 的镜像拉取策略为“从不”或“IfNotPresent”时，将使用节点中镜像缓存中的镜像。这消除了下载镜像时产生的延迟。

### 5. kubelet 镜像垃圾回收

`Kubernetes` 有一个内置的镜像垃圾回收机制。节点中的 `kubelet` 会定期检查磁盘使用率是否达到特定阈值（可通过标志配置）。一旦达到这个阈值，`kubelet` 会自动删除节点中所有未使用的镜像。

在建议的解决方案中需要实现自动和定期刷新机制。如果镜像缓存中的镜像被 kubelet 的 gc 删除，下一个刷新周期会将删除的镜像拉入镜像缓存。这可确保镜像缓存是最新的。

### 6.高层次设计



解决方案中提出的 `Cluster Image Cache` 将作为扩展 `API` 资源实现。这提供了使用 `Kubernetes` 风格的 `API` 管理（即 `CRUD` 操作）镜像缓存的优势。`Kubernetes` 提供了两种实现扩展 `API` 的机制，即扩展 `API` 服务器和自定义资源定义。我们将使用自定义资源定义 (CRD) 来实现扩展 API。

将定义 `ImageCache` 类型的自定义资源。需要编写一个新的控制器来作用于 `ImageCache` 资源。控制器将需要监视 `ImageCache` 资源并做出相应的反应。

### 7.ImageCache动作

#### 7.1. 创建动作

需要创建一个新的镜像缓存。这将导致将 ImageCache 资源中指定的容器镜像拉到指定的节点上。节点将以选择一组节点的节点选择器的形式指定。ImageCache 资源将以允许指定镜像列表和节点选择器的方式定义。为了允许指定不同的节点选择器，需要允许多个这样的列表。

控制器会将指定的镜像拉到指定的节点上。

#### 7.2. 清除动作

需要清除镜像缓存。之前拉取的镜像将被删除，前提是没有容器使用该镜像。如果镜像在未来某个时间点不再使用，控制者将不会删除此类镜像。这些镜像将留在节点中，以便 `Kubelet` 的 `GC` 操作可能会删除它们。如果用户需要完全擦除缓存中的镜像，用户需要在发出 API 操作之前确保没有容器使用镜像。

需要将终结器添加到 `ImageCache` 资源中。当执行清除操作时，控制器将首先采取措施删除节点中拉取的镜像。一旦所有镜像都被删除，控制器就会移除终结器。

#### 7.3. 更新动作

可以添加新镜像或删除现有镜像。节点选择器也有可能被修改。控制器需要采取适当的措施来保持 ImageCache 的当前状态符合 Desired 状态。

#### 7.4. 显示动作

控制器不需要任何操作。

#### 7.5. 定期自动刷新

控制器将有一个单独的刷新工作程序作为 go 例程运行。刷新工作者执行镜像缓存的定期刷新。如果它发现某些镜像丢失（这些镜像被删除或添加了新节点等），它会拉取这些镜像。

应该可以通过标志打开/关闭刷新工作器。也应该可以配置刷新频率。

#### 7.6. 按需刷新

应该可以通过注释 `ImageCache CR` 请求按需刷新镜像缓存。

### 8.自定义资源定义清单

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: imagecaches.kubefledged.io
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: kubefledged.io
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1alpha2
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: imagecaches
    # singular name to be used as an alias on the CLI and for display
    singular: imagecache
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: ImageCache
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ic
```

### 9. ImageCache 资源清单：
```
apiVersion: kubefledged.io/v1alpha2
kind: ImageCache
metadata:
  name: imagecache
  namespace: kube-fledged
spec:
  cacheSpec:
  # Following images are available in a public registry e.g. Docker hub.
  # These images will be cached to all nodes with label zone=asia-south1-a
  - images:
    - nginx:1.15.5
    - redis:4.0.11
    nodeSelector: zone=asia-south1-a
  # Following images are available in an internal private registry e.g. DTR.
  # These images will be cached to all nodes with label zone=asia-south1-b
  # imagePullSecret secret1 is specified at the end
  - images:
    - myprivateregistry/myapp:1.0
    - myprivateregistry/myapp:1.1.1
    nodeSelector: zone=asia-south1-b
  # Following images are available in an external private registry e.g. Docker store.
  # These images will be cached to all nodes in the cluster
  # imagePullSecret secret2 is specified at the end
  - images:
    - extprivateregistry/extapp:1.0
    - extprivateregistry/extapp:1.1.1
  imagePullSecrets:
  - name: secret1
  - name: secret2
status:
  status: Succeeded
  reason: ImagesPulled
  message: All requested images pulled succesfuly to respective nodes
```

### 10. ImagePuller 作业清单

```
apiVersion: batch/v1
kind: Job
metadata:
  name: imagepuller
spec:
  template:
    spec:
      nodeSelector:
      # Image will be pulled to this specific node
        kubernetes.io/hostname: worker1
      initContainers:
      # Get the echo binary from busybox container
      - name: busybox
        image: busybox:1.29.2
        command: ["cp", "/bin/echo", "/tmp/bin"]
        volumeMounts:
        - name: tmp-bin
          mountPath: /tmp/bin
        imagePullPolicy: IfNotPresent
      containers:
      - name: imagepuller
        # This is the image to be pulled
        image: mysql
        command: ["/tmp/bin/echo", "Image pulled successfully!"]
        securityContext:
          privileged: true
        volumeMounts:
        - name: tmp-bin
          mountPath: /tmp/bin
        imagePullPolicy: IfNotPresent
      volumes:
      - name: tmp-bin
        emptyDir: {}
      restartPolicy: Never
      imagePullSecrets:
      - name: secret1
      - name: secret2
 ```
