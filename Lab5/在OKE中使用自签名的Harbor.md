## 安装并在OKE中使用HTTP协议的Harbor

### 1. 安装Harbor

##### Step 1. 创建虚拟机

准备一台虚拟机，操作系统选择CentOS 7 , 并调整安全策略（VCN Security List中也需要放行对应的端口）：

```shell
sudo setenforce 0 
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sudo sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sudo firewall-cmd --zone=dmz --add-port=80/tcp
sudo firewall-cmd --zone=public --add-port=80/tcp
sudo firewall-cmd --zone=dmz --add-port=443/tcp
sudo firewall-cmd --zone=public --add-port=443/tcp
```

##### Step 2. 准备证书

请把下面的IP换成你的虚拟机IP或者域名：

```shell
sudo mkdir -p /data/cert
sudo chown -R opc:opc /data
cd /data/cert
openssl genrsa -des3 -passout pass:123456 -out ca.key 2048
openssl rsa -in ca.key -passin pass:123456 -out ca.key
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt -subj "/CN=10.0.10.222"
openssl genrsa -out tls.key 2048
openssl req -new -key tls.key -out tls.csr -subj "/CN=10.0.10.222"
cat > server.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = 10.0.10.222
EOF

openssl x509 -req -in tls.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out tls.crt -days 3650 -extfile server.ext
```

##### Step3. 安装Docker

这次咱们使用虚拟机搭建Harbor，需要使用docker和docker-compose，先来安装docker，:

```shell
sudo yum install -y yum-utils
sudo yum-config-manager \
   --add-repo \
   https://download.docker.com/linux/centos/docker-ce.repo

cat >> /etc/yum.repos.d/docker-ce.repo << "EOF"
[centos-extras]
name=Centos extras aarch64 - $basearch
baseurl=http://mirror.centos.org/altarch/7/extras/aarch64/
enabled=1
gpgcheck=1
gpgkey=https://www.centos.org/keys/RPM-GPG-KEY-CentOS-7-aarch64
EOF

sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl start docker
sudo systemctl enable docker
```

安装docker-compose，2023年3月最新版本是2.16.0版，最新版本请查看 https://github.com/docker/compose/releases

```shell
sudo curl -L https://github.com/docker/compose/releases/download/v2.16.0/docker-compose-`uname -s`-`uname -m` -o /usr/bin/docker-compose
sudo chmod +x /usr/bin/docker-compose
docker-compose -v
```

##### Step 4. 让Docker 客户端允许HTTP(可跳过此步，有特殊要求才执行此步骤)

在客户端上

```shell
sudo vim /etc/docker/daemon.json
```

```json
{
    "insecure-registries":["10.0.10.222:80"]
}
```

```shell
sudo systemctl restart docker
```

##### Step4. 安装Harbor

下载Harbor离线安装包，现在的最新版本是1.17.0，最新版本请查看 https://github.com/goharbor/harbor/releases

```shell
cd ~
wget https://github.com/goharbor/harbor/releases/download/v1.10.17/harbor-offline-installer-v1.10.17.tgz
tar -xzvf harbor-offline-installer-v1.10.17.tgz
cd harbor
vim harbor.yml
```

编辑配置文件harbor.yml, 把里面的IP，证书、密码等信息修改成你的，以下是修改的部分内容（使用默认密码）：

```yaml
#修改域名
hostname: 10.0.10.222

https:
  # 修改证书
  certificate: /data/cert/tls.crt
  private_key: /data/cert/tls.key
  
#使用默认密码
harbor_admin_password: Harbor12345
```

安装

```shell
sudo ./prepare
sudo ./install.sh
```

查看效果

```shell
sudo docker ps
```

```shell
CONTAINER ID   IMAGE                                  COMMAND                  CREATED          STATUS                    PORTS                                                                            NAMES
24409f50c471   goharbor/nginx-photon:v1.10.17         "nginx -g 'daemon of…"   58 seconds ago   Up 52 seconds (healthy)   0.0.0.0:80->8080/tcp, :::80->8080/tcp, 0.0.0.0:443->8443/tcp, :::443->8443/tcp   nginx
8c7bf4e4bbc3   goharbor/harbor-jobservice:v1.10.17    "/harbor/harbor_jobs…"   58 seconds ago   Up 52 seconds (healthy)                                                                                    harbor-jobservice
9419cdc57716   goharbor/harbor-core:v1.10.17          "/harbor/harbor_core"    58 seconds ago   Up 53 seconds (healthy)                                                                                    harbor-core
601bfb304739   goharbor/harbor-db:v1.10.17            "/docker-entrypoint.…"   58 seconds ago   Up 55 seconds (healthy)   5432/tcp                                                                         harbor-db
eec079f2a8e9   goharbor/redis-photon:v1.10.17         "redis-server /etc/r…"   58 seconds ago   Up 54 seconds (healthy)   6379/tcp                                                                         redis
0ba2f536b675   goharbor/registry-photon:v1.10.17      "/home/harbor/entryp…"   58 seconds ago   Up 54 seconds (healthy)   5000/tcp                                                                         registry
2f04724d6df7   goharbor/harbor-registryctl:v1.10.17   "/home/harbor/start.…"   58 seconds ago   Up 54 seconds (healthy)                                                                                    registryctl
a0cd270f75eb   goharbor/harbor-portal:v1.10.17        "nginx -g 'daemon of…"   58 seconds ago   Up 55 seconds (healthy)   8080/tcp                                                                         harbor-portal
05f496667edb   goharbor/harbor-log:v1.10.17           "/bin/sh -c /usr/loc…"   58 seconds ago   Up 57 seconds (healthy)   127.0.0.1:1514->10514/tcp                                                        harbor-log
```



##### Step 5. 上传测试镜像

打开 https://10.0.10.222 （你的VMIP或域名），让浏览器忽略证书错误，  使用admin+默认密码Harbor12345登录

![image-20230308183924880](C:\Users\Wilbur\docs\OCI-CloudNative\Lab5\在OKE中使用自签名的Harbor.assets\image-20230308183924880.png)

看到有个默认的项目library，我们推送一个项目到这个项目中：（注意，如果用https，请去掉“:80”字样）

```shell
sudo docker pull nginx
sudo docker tag nginx 10.0.10.222:80/library/nginx:harbor-ver-latest
sudo docker login 10.0.10.222:80
#用户名 admin
#默认密码 Harbor12345
sudo docker push 10.0.10.222:80/library/nginx:harbor-ver-latest
```

![image-20230308184006741](C:\Users\Wilbur\docs\OCI-CloudNative\Lab5\在OKE中使用自签名的Harbor.assets\image-20230308184006741.png)

## 2. OKE中使用Harbor

OKE能正常使用HTTPS的Harbor，所以不在本文的讨论范围之内。

##### Step 1.  让OKE允许自签名的镜像仓库 

当Harbor使用Http协议时，OKE需要做一些特殊处理。在创建Pool时，在"show advanced options"的"Initialization script"里面输入下面脚本内容。编辑Pool也可以修改初始化脚本，不过修改完后要缩容到0再扩展到所需节点数，否则存量worknode不会变更。

```shell
#!/bin/bash
curl --fail -H "Authorization: Bearer Oracle" -L0 http://169.254.169.254/opc/v2/instance/metadata/oke_init_script | base64 --decode >/var/run/oke-init.sh
bash /var/run/oke-init.sh

sudo dd iflag=direct if=/dev/sda of=/dev/null count=1
echo "1" | sudo tee /sys/class/block/sda/device/rescan
echo "y" | sudo /usr/libexec/oci-growfs

sudo bash -c 'cat << EOF >> /etc/crio/crio.conf
insecure_registries = ["10.0.10.222"]
EOF'
sudo systemctl daemon-reload
sudo systemctl restart crio
```

![image-20230308183759845](C:\Users\Wilbur\docs\OCI-CloudNative\Lab5\在OKE中使用自签名的Harbor.assets\image-20230308183759845.png)

##### Step 2. 部署测试应用

创建密钥

```shell
kubectl create secret docker-registry image-pull-secret \
  --docker-server=10.0.10.222:80 \
  --docker-username=admin \
  --docker-password=Harbor12345 \
  --docker-email=test@oracle.com
```

编辑一个部署文件nginx.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 100 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      imagePullSecrets:
        - name: image-pull-secret
      containers:
      - name: nginx
        image: 10.0.10.222:80/library/nginx:harbor-ver-latest 
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

部署

```shell
date
kubectl apply -f nginx.yaml
date && kubectl describe deployment nginx-deployment |grep Replicas:
```

![image-20230308181826438](assets\Image_20230308183346.png)

