---

tags: 云原生安全资讯,安全会议,k8s,GPT
spec: v0.1.0
version: v0.1.0

---

# [KubeCon + CloudNativeCon Europe 2024]: The State of K8sGPT: Your Troubleshooting Companion

| Item            | Content        | Note     |
|-----------------|----------------|----------|
| Talk Name   | The State of K8sGPT: Your Troubleshooting Companion |
| Conference Name | KubeCon + CloudNativeCon Europe 2024 |
| Talker          |  Thomas Schuetz  |
| Date            | 2024-03-19 |
| Materials       | [schedule](https://kccnceu2024.sched.com/event/1aQWc/the-state-of-k8sgpt-your-troubleshooting-companion-project-lightning-talk)   |
|                 | [video](https://www.youtube.com/watch?v=6KdBPjIZSok&list=PLj6h78yzYM2N8nw1YcqqKveySH6_0VnI0&index=308)      |
|                 | [slide](https://static.sched.com/hosted_files/kccnceu2024/6d/state_of_k8sgpt.pdf)      |


## 1. What

K8sGPT 是一个开源项目，旨在通过集成先进的人工智能技术来增强 Kubernetes 集群的监控和问题诊断能力。该项目提供了一个命令行工具，可以对 Kubernetes 集群进行扫描、诊断和分析，帮助用户以简单的英语理解和处理集群中的问题。K8sGPT 支持与多种 AI 服务提供商如 OpenAI、Azure、Cohere、Amazon Bedrock、Google Gemini 和本地模型进行集成，从而使得集群管理更加智能化和自动化。

* 开源项目地址：https://github.com/k8sgpt-ai/k8sgpt
* 文档：https://docs.k8sgpt.ai/

## 2. Why

1. K8s的故障排除非常复杂
2. 同样的问题会反复出现。
3. 人类可以发现问题，而AI可以提供解决问题的方案。

## 3. How

<img src="images/demo4.gif" width=650px; />

## 4. milestone

* 2023-03-28 发布 release 0.1.0
* 2023-04-11 Github项目获得 1000 star
* 2023-04-27 发布diyige Operator 版本
* 2023-12-19 加入 CNCF Sandbox

## 5. 未来计划

* 增加与更多云服务商的集成
* 支持对CRDs的扫描
* 提供额外的通知渠道，例如slack等
* 与其他AI/ML工具集成

## 6. 思考

作为 Operator 引入到 k8s 集群中，可能会带来新的风险：

1. 特别是其需要几乎所有资源的读取权限，可能会成为攻击或利用的一环
2. 对数据泄漏的担忧(虽然项目支持本地模型来规避该风险)

> This document use [Cloud Native Security News: Researcher Introduction](https://github.com/ssst0n3/security-research-specification/blob/main/cloud-native-security-news/security-conference-talk-learning.md) as template.
