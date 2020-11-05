+++
title = "GKE使用小记"
date = 2020-11-05T00:00:00+08:00
tags = ["gke"]
categories = [""]
draft = false
+++

## 申请

申请官网：https://cloud.google.com/kubernetes-engine

首先到申请官网登录 Google 账号，接着开始进入试用申请流程，在第1步中的国家/地区选择地区为美国，进入第2步，账号类型选择个人，姓名和地址通过美国地址生成器获取：http://www.haoweichi.com/ ，付款方式填上信用卡的信息（我用的是招行的全币卡），完成信息填写后提交，申请成功后信用卡会扣费1美元（会退回），结束后会得到300美元的免费试用金额以及91天的免费试用期。

申请完试用账号后可以创建 Kubernetes 集群，区域对应的位置可以在：https://cloud.google.com/compute/docs/regions-zones 查看，这里选择的是香港的节点（asia-east2-c），节点默认使用的机器配置是 e2-medium（2个vCPU，4GB内存），磁盘大小为100G，完成配置后点击创建按钮开始创建一个有3个节点集群。

![屏幕快照 2020-11-05 18.59.58](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-11-05%2018.59.58.png)

## 安装Gcloud SDK

参考官方文档：https://cloud.google.com/sdk/docs/install

把对应的 GcloudSDK 版本下载下来后解压：

```shell
$ wget https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-315.0.0-darwin-x86_64.tar.gz
```

执行安装脚本：

```shell
$ ./google-cloud-sdk/install.sh
```

按照安装提示输入交互选项即可完成安装，这里需要配置默认的 PROJECT_ID 来访问集群，PROJECT_ID 可以在 gcp 的控制台获取，配置完成后执行`gcloud init`来登录Google账号，执行命令后会弹出一个浏览器页面进行账号登录，在此步骤中也会更新 kubectl 的配置，完成后就可以在本地使用 kubectl 命令来连接 GKE 集群了。

在完成这个步骤后要注意本地用的 kubectl 版本跟服务端使用的版本是否一致，可以通过`kubectl version`命令获取，如果两者版本不一致可能会导致兼容性问题，这时候需要更新 kubectl 的版本：https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/

更新相应的 kubectl 版本:

```shell
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/darwin/amd64/kubectl"
```

现在执行`gcloud compute instances list`，获取 GKE 上的计算节点列表：

```text
NAME                                      ZONE          MACHINE_TYPE  PREEMPTIBLE  INTERNAL_IP  EXTERNAL_IP     STATUS
gke-cluster-1-default-pool-98a27c33-368d  asia-east2-c  e2-medium                  10.170.0.3   34.92.195.10    RUNNING
gke-cluster-1-default-pool-98a27c33-mm1m  asia-east2-c  e2-medium                  10.170.0.4   35.220.224.219  RUNNING
gke-cluster-1-default-pool-98a27c33-pvf6  asia-east2-c  e2-medium                  10.170.0.2   35.220.229.6    RUNNING
```

## 测试GKE集群

为了测试 GKE 集群，在这里需要推送一个本地镜像到 GKE 集群，在推送镜像之前，需要完成`gcloud auth configure-docker` 完成配置 Docker。

完成后执行命令推送镜像到 gcr（其中 PROJECT_ID 可通过`gcloud config get-value project`获取）：

```shell
$ docker push gcr.io/$PROJECT_ID/nginx:v1
```

推送完毕后，使用推送的镜像部署一个 Deployment：

```shell
$ kubectl run web --image=gcr.io/$PROJECT_ID/nginx:v1 --port 80
```

通过 NodePort 方式创建服务：

```shell
$ kubectl expose deployment web --target-port=80 --type=NodePort
```

最后使用 Ingress 暴露服务到 GKE 分配的公网IP，Ingress 定义如下：

```text
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: web
          servicePort: 80
        path: /
```

执行`kubectl apply -f ingress.yml`创建一个 Ingress，创建完成后等待数分钟，可以在 GKE 的控制台上看到 Ingress 状态变为 OK。

![屏幕快照 2020-11-05 19.30.13](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-11-05%2019.30.13.png)

这时候访问 Ingress 端点的IP，即可看到服务运行成功：

![屏幕快照 2020-11-05 19.31.38](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202020-11-05%2019.31.38.png)


