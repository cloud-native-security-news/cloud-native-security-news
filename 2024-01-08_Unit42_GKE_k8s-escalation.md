---

tags: Cloud Native Security News,云漏洞案例
author: ssst0n3
spec_version: v0.1.3
version: v0.1.1
changelog:
  - v0.1.1: update filename, metadata

---

# 1. 云漏洞案例: Unit42: GKE组合漏洞 k8s 提权

* 原文地址: https://unit42.paloaltonetworks.com/google-kubernetes-engine-privilege-escalation-fluentbit-anthos/

## 1.1 简介

本文系 Unit42/Yuval Avrahami 的 k8s 提权 (Container Escape == Cluster Admin?) 思想的实践。

Unit42/Shaul Ben Hai 发现 GKE 中的两个漏洞，组合起来可获得 k8s 的完整控制权限。第一个漏洞是 FluentBit 容器 挂载了主机的 /var/lib/kubelet/pods 目录，导致可以读取任意pod的serviceaccount token; 第二个漏洞是 Anthos Service Mesh 权限较大，拥有创建pod权限。

## 1.2 漏洞详情

### 前提条件

1. 攻击者已获得 FluentBit 容器的命令执行权限/任意文件读取权限，或可通过其他容器逃逸至node
2. 目标集群启用了 Anthos Service Mesh 特性

> FluentBit 是 GKE 集群中的默认日志Agent，以 DaemonSets 的形式默认部署。这意味着 FluentBit 会被安装在集群中的每个节点上，默认启用。

> Anthos Service Mesh 特性是 Google 对 Istio 的实现， 启用该特性时，Istio-cni-node DaemonSet 将被安装在集群中。

### 漏洞一: FluentBit 挂载主机 /var/lib/kubelet/pods 目录，泄漏 service account token

FluentBit 容器挂载了主机的 `/var/lib/kubelet/pods` 目录，在这个目录之下，节点上运行的每个Pod下都有一个包含projected service account token 的 `kube-api-access` 卷。

```
$ cat /var/lib/kubelet/pods/<POD-ID>/volumes/kubernetes.io~projected/kube-api-access-m2f84/token
```

由于 FluentBit 是作为 DaemonSet 部署的，这使得攻击者可以在每个节点上重复最初的入侵行为，搜索集群中任何其他 Pod 挂载的令牌。攻击者可以绘制出整个集群的映射，并找到 Istio-Installer-container 的token。

### 漏洞二: ASM CNI DaemonSet 权限过大

Anthos Service Mesh (ASM) 的CNI DaemonSet 在安装后保留了过多权限。这允许攻击者创建一个具有 ASM 的 CNI DaemonSet 权限的新 Pod。

Istio-cni-node DaemonSet 负责在集群中的每个节点上安装和配置 Istio CNI 插件。因此，它需要并拥有执行这些任务的强大权限。但是，一旦它安装并运行起来，就不再需要这么广泛的权限了。这种情况可能允许攻击者利用 DaemonSet 来获得对集群的未授权访问，例如，通过创建一个拥有高级权限的“powerful Pod”。

### 组合利用

1. 利用 FulentBit 挂载的 /var/lib/kubelet/pods 目录，找到 ASM Istio-Installer-container 的token
2. 利用 ASM 的token 创建 pod, 挂载 kube-system 的service account token, 其中 选择 clusterrole-aggregation-controller 作为要挂载的 serviceaccount 
3. 利用 clusterrole-aggregation-controller  获得 k8s 的完整控制权限，相关技术在 unit42 的 [Container Escape to Shadow Admin: GKE Autopilot Vulnerabilities](https://unit42.paloaltonetworks.com/gke-autopilot-vulnerabilities/) 文章中介绍过

## 1.3 时间线

- 2023-09-12: Palo Alto Networks 向谷歌云漏洞赏金计划提交了一份关于这个问题的报告
- 2023-09-13: Google 确认漏洞
- 2023-10-24: Google 确认漏洞绕过了重要的安全控制措施
- 2023-12-14: Google 修复, 漏洞编号 [GCP-2023-047](https://cloud.google.com/anthos/clusters/docs/security-bulletins#gcp-2023-047)

## 1.4 总结

* 该漏洞的前置条件一实际是较为苛刻的。本文默认了该前提条件的成立， 即仅关注 “Second-stage cloud attacks”
* 自 Yuval 提出 k8s 提权 已经有两年，但类似本文的 “Second-stage cloud attacks” 问题仍广泛存在，相关风险值得云厂商继续挖掘

> 本文使用 [Cloud Native Security News: 云漏洞案例](https://github.com/cloud-native-security-news/spec/blob/main/云漏洞案例.md) 作为模板。

----

本文发布已获得"云原生安全资讯"项目授权, 同步发布于以下平台

* github: [https://github.com/cloud-native-security-news/cloud-native-security-news](https://github.com/cloud-native-security-news/cloud-native-security-news)
* 知乎专栏: [云原生安全资讯](https://www.zhihu.com/column/c_1694733563684151296)
* 公众号: 石头的安全料理屋

欢迎加入 "云原生安全资讯"项目 👏 阅读、学习和总结云原生安全相关资讯 
