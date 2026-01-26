---
title: flux 介绍
author: Zpekii
tags: [GitOps,Flux]
categories: [GitOps, flux]
---

# Flux: 自动化GitOps好帮手

## 说在前面

> 推荐阅读：[GitOps | GitOps is Continuous Deployment for cloud native applications](https://www.gitops.tech/#what-is-gitops)

### 什么是 GitOps

**GitOps** 是一种实现**云原生应用持续部署**的方法。核心是使用我们熟知的 **[Git]([Git](https://git-scm.com/))** 工具，在一个包含了我们应用的基础设施的声明性描述(比如 [k8s deployment.yaml]([Deployments | Kubernetes](https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/)))的 **Git** 仓库中，完成自动化流程部署；在我们需要在集群上部署新应用或更新现有应用时，就只需要在 **Git** 仓库上提交就行了。

### 为什么要搭建 GitOps

1. 让你的部署更快并且还可以让你更频繁地执行部署
	- 采用 **GitOps** 的独到之处就是你不需要来回切换工具来部署应用
2. 更简单更快的错误恢复
	- 错误恢复只需要使用 **Git** 进行回退还原即可
3. 更便捷的证书部署管理
	- 你不需要真正访问到部署环境中才能管理
4. 自文档化部署
	- **GitOps** 规范要求对任何环境的每次更改都必须通过 **Git** 完成，这样你不需要通过 **ssh** 登录到服务器，直接通过查看主分支就能知道服务器都运行了什么
5. 团队知识共享
	- **Git** 仓库中包含了所有应用的基础设施完整描述，团队中的每个人可以随时快捷的了解变化

## Flux 介绍

> 官方地址: [Flux](https://fluxcd.io/)

Flux 是一个用于保持 k8s 集群和配置源(如 Git 仓库)同步的工具，它能够在新代码推送到仓库后自动更新集群上部署的应用

## Flux应用示例

### 前置条件

- 已部署 k8s 或 k3s 集群(我将以 k3s 为例)
- 可访问的 **Git** 仓库(Github, Gitlab 或 Gitea, 下面将以我自己部署的 Gitea 仓库为例，公开仓库:[Zpekii/go-example](https://git.0orz.top/Zpekii/go-example))

### 安装 Flux Cli 工具

通过 Bash 安装(适用于Linux)

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
```

###  检查是否满足所需依赖

```bash
flux check --pre
```

执行后输出形如:

```
► checking prerequisites
✔ Kubernetes 1.34.3+k3s1 >=1.32.0-0
✔ prerequisites checks passed
```

### 将 Flux 安装到集群

```bash
flux bootstrap git \
	--url=$URL \
	--branch=$BRANCH \
	--username=$USER_NAME \
	--token-auth=true \
	--path=./clusters/app
```

其中`$URL`、`$BRANCH`和`$USER_NAME`需要替换成实际的 git 仓库地址、分支(一般是`main`或`master`主分支)以及拥有访问 **git** 仓库的账号名

我的例子:

```bash
flux bootstrap git \
	--url=https://git.0orz.top/Zpekii/go-example.git \
	--branch=main \
	--username=Zpekii \
	--token-auth=true \
	--path=./clusters/app
```

执行后，需要输入 **git** 账号密码，如果通过校验，则会输出形如:

```bash
► connecting to github.com
✔ repository created
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
✔ install completed
► configuring deploy key
✔ deploy key configured
► generating sync manifests
✔ sync manifests pushed
► applying sync manifests
◎ waiting for cluster sync
✔ bootstrap finished
```

这个过程将会：

- (如果你填写的 git 仓库地址还没有创建，那么它会帮你创建)
- 在你的 git 仓库添加 Flux 组件(通过向你的仓库发起一个提交进行变更, 组件将会保存到项目根目录`clusters/app`下，这个由`--path`参数决定)
- 在你的集群上部署 Flux 组件
- 配置 Flux 组件跟踪仓库上的 `clusters/app` 路径（由`--path`参数决定）

### 克隆你的仓库

```bash
git clone $URL
```

请替换`$URL`为你实际的仓库地址，我的例子:

```bash
git clone https://git.0orz.top/Zpekii/go-example.git
```

### 创建 `kustomize `目录来保存你的部署配置文档

```bash
cd $PRJ_PATH && mkdir -p kustomize
```

`$PRJ_PATH`替换为拉取仓库到服务器本地文件系统后的项目路径，我的例子:

```bash
cd go-example && mkdir -p kustomize
```

### 编写集群部署配置文档

使用 vim 创建和编辑文档

```bash
vim kustomize/deployment.yaml
```

按下 Insert 键，根据自己的实际情况编写吧，可以参考我的例子:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: helloapp
---
apiVersion: v1
kind: Service
metadata:
  name: go-example
  namespace: helloapp
spec:
  selector:
    app: go-example
  ports:
    - port: 8800
      targetPort: 8800
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-example
  namespace: helloapp
spec:
  selector:
    matchLabels:
      app: go-example
  replicas: 3
  template:
    metadata:
      labels:
        app: go-example
    spec:
      containers:
        - name: go-example
          image: harbor.0orz.top/go-example/go-example:8b5d44399e523027840a68ce17249d9ecfd5c094
          imagePullPolicy: IfNotPresent
          resources:
            requests:
              memory: "5Mi"
              cpu: "10m"
            limits:
              memory: "50Mi"
              cpu: "100m"
          ports:
            - containerPort: 8800
          volumeMounts:
            - name: config-volume
              mountPath: /config
            - name: helloapp-test-key
              mountPath: /certs
          command: ["/helloapp"]
          args: ["-f", "/config/.linux-config.yaml"]
      volumes:
        - name: config-volume
          configMap:
            name: go-example-config
        - name: helloapp-test-key
          secret:
            secretName: helloapp-test-key # 需要事先创建该 Secret 
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: go-example-config
  namespace: helloapp
data:
  .linux-config.yaml: |
    server:
      port: 8800
    certs:
      testKeyPath: /certs/test.key

```

我都配置了什么：

- 创建了一个`helloapp`命名空间用来单独和管理我部署的应用
- 创建了一个`Service`来暴露我的应用，让外部可以访问
- 声明了我的应用`Deployment`信息，需要哪些资源、使用哪个镜像、创建几个副本，以及使用哪些配置和密钥
- 创建了一个`ConfigMap`来声明我的应用配置

按下 Esc 键退出编辑，然后输入保存并退出 vim 编辑

```bash
:wq
```

### 将仓库源添加到 Flux 中

在 Flux 中创建一个`git source`

```bash
flux create source git $SOURCE_NAME \
  --url=$URL \
  --branch=$BRANCH \
  --interval=1m \
  --export > ./clusters/app/$SOURCE_NAME-source.yaml
```

请根据实际替换相应的变量，我的例子:

```bash
flux create source git go-example \
  --url=https://git.0orz.top/Zpekii/go-example.git \
  --branch=main \
  --interval=1m \
  --export > ./clusters/app/go-example-source.yaml
```

执行后，输出形如：

```bash
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: go-example
  namespace: flux-system
spec:
  interval: 1m
  ref:
    branch: main
  url: https://git.0orz.top/Zpekii/go-example.git
```

### 将部署信息添加到 Flux 中

在 Flux 中创建一个 `Kustomization `，让 Flux 知道从哪个源读取部署配置文档，然后应用并部署到集群中

```bash
flux create kustomization $SOURCE_NAME \
  --target-namespace=$TARGET_NAMESPACE \
  --source=$SOURCE_NAME \
  --path=$CONFIG_PATH \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/app/$SOURCE_NAME-kustomization.yaml
```

请根据实际替换相应的变量（注意`$CONFIG_PATH`是填写前面创建的`kustomize`目录路径），我的例子:

```bash
flux create kustomization go-example \
  --target-namespace=helloapp \
  --source=go-example \
  --path="./kustomize" \
  --prune=true \
  --wait=true \
  --interval=30m \
  --retry-interval=2m \
  --health-check-timeout=3m \
  --export > ./clusters/app/go-example-kustomization.yaml
```

执行后，输出形如:

```bash
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: go-example
  namespace: flux-system
spec:
  interval: 30m0s
  path: ./kustomize
  prune: true
  retryInterval: 2m0s
  sourceRef:
    kind: GitRepository
    name: go-example
  targetNamespace: helloapp
  timeout: 3m0s
  wait: true
```

### 将修改提交到仓库

```bash
git add . && git commit -m "chore: add GitRepository and  Kustomization"
git push
```

### 查看 Flux 应用同步状况

```bash
flux get kustomizations --watch
```

输出形如:

```bash
NAME            REVISION                SUSPENDED       READY   MESSAGE
flux-system     main@sha1:bea43605      False           True    Applied revision: main@sha1:bea43605
go-example      main@sha1:bea43605      False           True    Applied revision: main@sha1:bea43605
```

### 查看应用部署情况

```bash
kubectl get all -n $TARTGET_NAMESPACE
```

`$TARTGET_NAMESPACE`请替换成实际的部署命名空间，我的例子:

```bash
kubectl get all -n helloapp
```

输出形如:

```bash
NAME                             READY   STATUS    RESTARTS   AGE
pod/go-example-6d6f47fd6-5w4xw   1/1     Running   0          23h
pod/go-example-6d6f47fd6-gj249   1/1     Running   0          23h
pod/go-example-6d6f47fd6-kwxg5   1/1     Running   0          23h

NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/go-example   LoadBalancer   10.43.64.67   10.0.0.16     8800:32615/TCP   24h

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/go-example   3/3     3            3           24h

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/go-example-59996666bf   0         0         0       24h
replicaset.apps/go-example-6d6f47fd6    3         3         3       23h
```

## 最后

恭喜你🎉，你现在拥有了一套完全自动化的、基于 Flux 的 GitOps 持续部署流水线！如有疑问或任何想交流的内容，欢迎评论和留言😄