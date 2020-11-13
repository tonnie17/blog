+++
title = "初试Helm"
date = 2020-11-13T01:00:00+08:00
tags = ["gke"]
categories = [""]
draft = false
+++

# Helm

Helm 是一个用来管理 Kubernetes 包（Charts）的工具集，使用 Helm 可以轻松管理 Kubernetes 包的配置和依赖，简化了 Kubernetes 的应用管理和更新。

Helm 分为两个部分：

- 客户端（Helm）和服务端（Tiller）

实验内容演示了如何用 GKE 来使用 Helm 进行包的安装，由于 Helm 3.0 版本以后去除了 Tiller，因此修改了演示命令，不再包含 Tiller 内容。

首先获取 Helm 的安装脚本：

```sh
$ curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
```

执行脚本安装 Helm：

```sh
$ chmod 700 get_helm.sh && ./get_helm.sh
```

在安装 Charts 之前，使用`helm repo update`命令来更新可用的 Charts 列表：

```sh
$ helm repo update
```

更新完毕后，安装 mysql 官方的 chart，release名称为`my-release`：

```sh
$ helm install my-release stable/mysql
```

执行完后，得到以下输出：

```text
NAME: my-release
LAST DEPLOYED: Fri Nov 13 09:04:23 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
my-release-mysql.default.svc.cluster.local

To get your root password run:

    MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace default my-release-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode; echo)

To connect to your database:

1. Run an Ubuntu pod that you can use as a client:

    kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il

2. Install the mysql client:

    $ apt-get update && apt-get install mysql-client -y

3. Connect using the mysql cli, then provide your password:
    $ mysql -h my-release-mysql -p

To connect to your database directly from outside the K8s cluster:
    MYSQL_HOST=127.0.0.1
    MYSQL_PORT=3306

    # Execute the following command to route the connection:
    kubectl port-forward svc/my-release-mysql 3306

    mysql -h ${MYSQL_HOST} -P${MYSQL_PORT} -u root -p${MYSQL_ROOT_PASSWORD}
```

这时 mysql chart 已经被安装并部署了，可以用`helm ls`来验证：

```text
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
my-release	default  	1       	2020-11-13 09:04:23.913907536 +0000 UTC	deployed	mysql-1.6.7	5.7.30
```

根据输出提示，我们可以运行一个系统镜像安装 mysql client 来连接之前部署的 mysql 服务：

```sh
$ kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
```

进入容器后，执行命令更新并安装 mysql client：

```sh
$ apt-get update && apt-get install mysql-client -y
```

Helm 在安装 mysql 后会创建一个 Service，使用 mysql 命令连接服务名来连接到 mysql 服务端：

```sh
$ mysql -h my-release-mysql -p
```

