+++
title = "使用GKE管理Deployment"
date = 2020-11-13T00:00:00+08:00
tags = ["gke"]
categories = [""]
draft = false
+++

在本节中介绍了一些 Dev Ops 在管理部署阶段中使用的一些手段，例如：“持续部署”，“蓝绿部署”和“金丝雀部署”等等。

Heterogeneous deployments（异构部署）通常包含两个或多个特定的基础设施或地区以解决特殊的技术需求，根据不同的部署细节，Heterogeneous deployments 也被称作 "hybrid",  "multi-cloud" 或 "public-private"，包括在跨区域的单一云，多公有云（multi-cloud），私有云和公有云的混合（"hybrid"， "public-private"）。

在单一环境或地区上部署通常会遇到一些局限：

- 资源耗尽：在单一环境，尤其是私有云的环境，你可能没有生产环境所需要的计算、网络和存储资源。
- 地理区域的限制：只部署在一个环境会使地理区域遥远的流量在访问时需要跨越大半个世界的区域。
- 限制的高可用性
- 供应商锁定
- 不灵活的资源

Heterogeneous deployments 可以用来解决这些难题，三个常见的场景是：多云部署、导向内部数据、持续集成和持续部署（CI/CD）。

实验内容演示了如何用 GKE 来实践 Heterogeneous deployments。

## 创建 Deployment

首先获取实验需要的样例代码：

```sh
$ gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
cd orchestrate-with-kubernetes/kubernetes
```

创建一个5个节点的集群：

```sh
$ gcloud container clusters create bootcamp --num-nodes 5 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

更新`deployments/auth.yaml`文件，把 Deployment 镜像的版本修改为 1.0.0：

```yaml
...
containers:
- name: auth
  image: kelseyhightower/auth:1.0.0
...
```

使用`kubectl create`命令创建`auth`服务的 Deployment 对象：

```sh
$ kubectl create -f deployments/auth.yaml
```

验证 Deployment 对象的创建状态：

```sh
$ kubectl get deployments
```

当 Deployment 创建完后，会为之创建一个 ReplicaSet 对象，它会用来创建 Pod 对象，通过下面命令查看 Pod 状态：

```sh
$ kubectl get pods
```

接下来创建`auth`服务的 Service：

```sh
$ kubectl create -f services/auth.yaml
```

现在继续创建`hello`服务的 Deployment 和 Service：

```sh
$ kubectl create -f deployments/hello.yaml
$ kubectl create -f services/hello.yaml
```

同样地，创建`frontend`服务的 Deployment 和 Service：

```sh
$ kubectl create secret generic tls-certs --from-file tls/
$ kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
$ kubectl create -f deployments/frontend.yaml
$ kubectl create -f services/frontend.yaml
```

当服务创建好之后，可以通过`kubectl get services frontend`获取到 frontend Service 的 external IP，通过 curl 命令来对服务进行访问：

```sh
$ curl -ks https://<EXTERNAL-IP>
```

或者可以直接通过模板语法来对服务进行访问，而避免手动获取 ip 地址：

```sh
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`
```

通过`kubectl scale`命令可以更新 Deployment 对象 Pod 副本集数量，下面将 hello Deployment 的 Pod 数量提升为5个：

```sh
$ kubectl scale deployment hello --replicas=5
```

现在可以看到 5 个 Pod 都处于运行状态了：

```sh
$ kubectl get pods | grep hello-
```

还是通过`kubectl scale`命令，这次把实例数更新回 3 个：

```sh
$ kubectl scale deployment hello --replicas=3
```

运行中的 Pod 变回了 3 个：

```sh
$ kubectl get pods | grep hello-
```

## 滚动更新

通过滚动更新机制我们可以把一个运行的 Deployment 逐渐升级到新版本，当 Deployment 更新到新版本时，会创建一个新的 ReplicaSet，在增加新的 ReplicaSet 的副本数的同时降低旧的 ReplicaSet 的副本数。

![8d107e36763fd5c1.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/uc6D9jQ5Blkv8wf_ccEcT35LyfKDHz7kFpsI4oHUmb0%3D.png)

要更新 Deployment 对象，可以使用`kubectl edit`命令：

```sh
$ kubectl edit deployment hello
```

把 Deployment 的镜像版本修改为 2.0.0：

```yaml
...
containers:
  image: kelseyhightower/hello:2.0.0
...
```

保存退出后会触发 hello Deployment 进行滚动更新，查看 Deployment 创建的新的 ReplicaSet：

```sh
$ kubectl get replicaset
```

查看滚动更新历史的新版本对应的记录：

```sh
$ kubectl rollout history deployment/hello
```

如果在滚动更新 Deployment 时发现问题，可以执行`kubectl rollout pause`暂停 Deployment 的滚动更新：

```sh
$ kubectl rollout pause deployment/hello
```

查看当前状态：

```sh
$ kubectl rollout status deployment/hello
```

或者直接验证 Pod 的状态：

```sh
$ kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

通过`kubectl rollout resume`我们可以继续被暂停的滚动更新：

```sh
$ kubectl rollout resume deployment/hello
```

滚动更新完成后，执行`kubectl rollout status deployment/hello`会得到以下输出，代表 Deployment 的滚动更新已经完毕：

```text
deployment "hello" successfully rolled out
```

如果我们发现新版的 Deployment 存在问题，想要回滚到旧的版本，可以通过`kubectl rollout undo`把 Deployment 回退到旧的版本：

```sh
$ kubectl rollout undo deployment/hello
```

验证版本历史：

```sh
$ kubectl rollout history deployment/hello
```

## 金丝雀部署

如果你想要你的一部分用户体验新的 Deployment，可以使用金丝雀部署的方式，它允许你的变更部署只关联到一部分用户子集，从而降低风险。

一个金丝雀部署包含新旧版本的多个 Deployment，以及导向 Deployment 的共同 Service。

![48190cf58fdf2eeb.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/qSrgIP5FyWKEbwOk3PMPAALJtQoJoEpgJMVwauZaZow%3D.png)

首先，创建下面这个新版本的 hello Deployemnt：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
        track: canary
        # Use ver 1.0.0 so it matches version on service selector
        version: 1.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
...
```

通过`kubectl create`来创建一个 Deployment 对象：

```sh
$ kubectl create -f deployments/hello-canary.yaml
```

通过 curl 我们对`hello`服务的 version 进行验证：

```sh
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

因为此前我们已经部署了具有 3 个副本的旧版本的 Deployment，因此我们执行上面这条命令时，流量只有 1/4 的比例会访问到新版本，执行多次会发现输出的版本可能是 1.0.0（75%）或 2.0.0（25%）。

但如果我们只想要让一部分用户“固定”在 Canary Deployment，而不影响所有的用户时，这时我们可以创建一个 session affinity 的 Service，在 Service 的定义中添加`sessionAffinity: ClientIP`后，具有相同 IP 的用户每次请求都会路由到相同版本的 Deployment 上：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

## 蓝绿部署

滚动更新的好处是可以让我们以最小的影响和开销来缓慢地部署新版本的应用，使用蓝绿部署的方式，我们可以在当新版本的应用在完全部署后再修改 Load Blancer 把流量引向新版本。

在 Kubernetes 蓝绿部署下，分为 blue 和 green 两个版本的 Deployment，对应旧版本和新版本，它们都包含完整的实例数，当 green 版本部署运行后，可以通过 Service，作为 Deployment 的 router，把流量指向对应的新版本。

因为蓝绿版本的部署特性，你至少需要 Deployment 需要的资源数 X2 作为部署所需的资源，所以要保证你有足够的资源来进行部署。

![9e624196fdaf4534.png](https://image-1301539196.cos.ap-guangzhou.myqcloud.com/POW8Q247ZKNY%2ByHIartCsoEu8MAih7k4u1twusCx6pw%3D.png)

在这种方式下，我们会有一个 blue 版本的 Service，对应着版本 1.0.0 的 Deployment：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  selector:
    app: "hello"
    version: 1.0.0
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

```sh
$ kubectl create -f deployments/hello-blue.yaml
```

然后我们有一个准备部署的版本为 2.0.0 的 Deployment：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
```

再来创建一个 green 版本的 Service，对应着版本 2.0.0 的 Deployment：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: hello
spec:
  selector:
    app: hello
    version: 2.0.0
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

通过 curl 我们会看到当前使用的是 1.0.0 版本：

```sh
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

接下来创建 green 版本对应的 Service：

```sh
$ kubectl create -f deployments/hello-green.yaml
```

现在再执行 curl 命令会发现新的版本已被使用：

```sh
$ curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

当需要的时候，我们要把版本回退到旧版本，这时候只需要重新应用旧版本的 Service即可：

```sh
$ kubectl apply -f services/hello-blue.yaml
```

