---
title: cilium 介绍
author: Zpekii
tags: [cilium]
categories: [cilium]
---

# 初识Cilium 

## Cilium能干什么？解决了什么问题？

### 能干什么:

1. **网络连接**

   - 为 Kubernetes Pod 提供 IP 地址和路由，让它们能互相通信
     - 起到和[flannel](https://github.com/flannel-io/flannel)一样的作用

   - 替代 kube-proxy，直接实现 ClusterIP、NodePort、LoadBalancer 等服务访问

2. **安全策略**

   - 支持 **L3/L4（IP、端口）** 到 **L7（应用层协议，如 HTTP、gRPC、Kafka）** 的安全策略

   - 不依赖固定 IP，而是基于 **容器/服务身份** 来定义访问控制，更适合动态微服务环境

3. **可观测性**

   - 内置 **Hubble**，可以实时查看 Pod/Service 之间的通信流量和拓扑关系

   - 提供丰富的可视化和监控能力，帮助排查网络问题

4. **高性能转发**

   - 基于 **eBPF** 技术，在 Linux 内核中直接处理数据包，性能比传统 iptables 更好

   - 减少规则维护复杂度，提升大规模集群的稳定性

5. **可扩展性**

   - 支持多种环境（Kubernetes、容器运行时、裸机）

   - 能与 Service Mesh、API Gateway 等结合，提供更细粒度的流量控制

### 解决了什么问题:

1. **基于 IP 的安全策略不适用**

   - iptables 等传统机制依赖固定 IP/端口来做访问控制
   - 在微服务架构下，Pod 的 IP 经常变化，导致规则维护复杂且容易出错

2. **性能瓶颈**

   - kube-proxy 使用 iptables 维护成千上万条规则
   - 在大规模集群中，规则更新频繁，性能下降明显

3. **缺乏可视化与可观测性**

   - 难以追踪 Pod 与 Service 之间的通信路径

   - 故障排查和安全审计困难

## 怎么部署使用?

> 官方教程: [Cilium Quick Installation — Cilium 1.18.6 documentation](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

### Linux 环境

> **前置条件:**
>
> 主机上需要部署或连接了 [k8s]([安装 kubeadm | Kubernetes](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)) 集群或 [k3s ]([快速入门指南 | K3s](https://docs.k3s.io/zh/quick-start))集群
>
> 以下为在 k3s 下安装示例 

#### 下载 Cilium Cli

```shell
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

#### 指定 k3s `kube config`路径

```shell
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

#### 安装最新版本 Cilium

> 注: 编写文档日期 2026-01-19 最新版本是: `v1.18.6`

```shell
cilium install --version 1.18.6
```

#### 验证安装

查看运行状态

```shell
cilium status --wait
```

测试连接

```shell
cilium connectivity test
```

## 有哪些坑需要注意的?

### 默认不支持 HTTP/1.0 协议

### Linux内核版本需要 > 4.9, Linux内核需支持 eBPF