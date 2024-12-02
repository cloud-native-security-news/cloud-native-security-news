---

tags: 云原生安全资讯,安全会议,k8s,gpu,nvidia
author: ssst0n3
spec: v0.1.1
version: v0.1.0

---

# [KubeCon + CloudNativeCon Europe 2024]: Mastering GPU Management in Kubernetes Using the Operator Pattern

| Item            | Content        | Note     |
|-----------------|----------------|----------|
| Talk Name   | Mastering GPU Management in Kubernetes Using the Operator Pattern |
| Conference Name | KubeCon + CloudNativeCon Europe 2024 |
| Talker          |  Shiva Krishna Merla, NVIDIA |
| | Kevin Klues, NVIDIA |
| Date            | 2024-03-22 |
| Materials       | [schedule](https://kccnceu2024.sched.com/event/1YeQY/mastering-gpu-management-in-kubernetes-using-the-operator-pattern-shiva-krishna-merla-kevin-klues-nvidia)   |
|                 | [video](https://www.youtube.com/watch?v=jbpIFCkEEng)      |
|                 | [slide](https://static.sched.com/hosted_files/kccnceu2024/65/Mastering%20GPU%20Management%20in%20Kubernetes%20Using%20the%20Operator%20Pattern.pptx.pdf)      |


## 1. What

gpu operator

* 开源项目地址：https://github.com/NVIDIA/gpu-operator
* 文档：https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html

## 2. Why

一个k8s集群可能包括很多节点，在集群中运维GPU存在以下痛点：

* 不同节点软件栈可能异构: 驱动版本、配置不同; 有些节点没有GPU
* 驱动程序配置: kernel modules, libs, services / daemons, cli
* 各节点GPU的特定配置: GPU分区、Clock Speed、GPU 持久模式等
* 运行时配置: 不同runtime的配置不同; 指定特定runtime作为GPU运行时 ;修改后要reload
* GPU 健康监控: GPU异常无法自动传播到k8s

## 3. How to Use

### 3.1 前提条件

* k8s(各容器组件升级到较新版本)
* node有GPU 硬件

### 3.2 安装

参考官方文档: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/getting-started.html#prerequisites

安装helm、gpu-operator

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 && chmod 700 get_helm.sh && ./get_helm.sh
$ helm repo add nvidia https://helm.ngc.nvidia.com/nvidia && helm repo update
"nvidia" has been added to your repositories
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "nvidia" chart repository
Update Complete. ⎈Happy Helming!⎈
$ helm install --wait --generate-name -n gpu-operator --create-namespace nvidia/gpu-operator --version=v24.6.1
NAME: gpu-operator-1732846659
LAST DEPLOYED: Fri Nov 29 10:17:42 2024
NAMESPACE: gpu-operator
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

如果`nvidia-operator-validator`有问题，删除pod重建。

```
$ kubectl delete pod -n gpu-operator nvidia-operator-validator-jv6n5
```

hello world

```
cat << EOF > cuda-vectoradd.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
```

```
$ kubectl apply -f cuda-vectoradd.yaml
$ kubectl logs pod/cuda-vectoradd
[Vector addition of 50000 elements]
Copy input data from the host memory to the CUDA device
CUDA kernel launch with 196 blocks of 256 threads
Copy output data from the CUDA device to the host memory
Test PASSED
Done
```

## 4. How it Works

```
$ kubectl get pods -n gpu-operator
NAME                                                              READY   STATUS      RESTARTS   AGE
gpu-feature-discovery-m4kxw                                       1/1     Running     0          91m
gpu-operator-1733126787-node-feature-discovery-gc-7dd879f54phld   1/1     Running     0          98m
gpu-operator-1733126787-node-feature-discovery-master-b9667h4vj   1/1     Running     0          98m
gpu-operator-1733126787-node-feature-discovery-worker-9l7kw       1/1     Running     0          98m
gpu-operator-1733126787-node-feature-discovery-worker-sbkjz       1/1     Running     0          98m
gpu-operator-6db7fb794d-v7fvz                                     1/1     Running     0          98m
nvidia-container-toolkit-daemonset-h2d95                          1/1     Running     0          91m
nvidia-cuda-validator-z7sjx                                       0/1     Completed   0          86m
nvidia-dcgm-exporter-86vc6                                        1/1     Running     0          91m
nvidia-device-plugin-daemonset-p8t5j                              1/1     Running     0          86m
nvidia-driver-daemonset-wm2h7                                     1/1     Running     0          91m
nvidia-operator-validator-p9cb2                                   1/1     Running     0          86m
```

### 4.1 Node Feature Discovery(NFD)

利用 [Node Feature Discovery (NFD)](https://github.com/kubernetes-sigs/node-feature-discovery) 来执行这种标签化。NFD 为节点打标签，以指示在哪些node上安装驱动程序。

### 4.2 Core Service

#### 4.2.1 driver

nvidia-driver-daemonset 负责安装 driver

```
$ kubectl logs nvidia-driver-daemonset-wm2h7 -n gpu-operator
nvidia driver modules are not yet loaded, invoking runc directly
DRIVER_ARCH is x86_64
Creating directory NVIDIA-Linux-x86_64-550.90.07
Verifying archive integrity... OK
Uncompressing NVIDIA Accelerated Graphics Driver for Linux-x86_64 550.90.07
...
========== NVIDIA Software Installer ==========

Starting installation of NVIDIA driver version 550.90.07 for Linux kernel version 5.15.0-113-generic

Stopping NVIDIA persistence daemon...
Unloading NVIDIA driver kernel modules...
Unmounting NVIDIA driver rootfs...
Checking NVIDIA driver packages...
Updating the package cache...
Resolving Linux kernel version...
Proceeding with Linux kernel version 5.15.0-113-generic
Installing Linux kernel headers...
Installing Linux kernel module files...
Generating Linux kernel version string...
Compiling NVIDIA driver kernel modules...
warning: the compiler differs from the one used to build the kernel
  The kernel was built by: gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
  You are using:           cc (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
/usr/src/nvidia-550.90.07/kernel/nvidia-drm/nvidia-drm-drv.c: In function 'nv_drm_register_drm_device':
/usr/src/nvidia-550.90.07/kernel/nvidia-drm/nvidia-drm-drv.c:1750:5: warning: ISO C90 forbids mixed declarations and code [-Wdeclaration-after-statement]
 1750 |     bool bus_is_pci =
      |     ^~~~
Skipping BTF generation for /usr/src/nvidia-550.90.07/kernel/nvidia-peermem.ko due to unavailability of vmlinux
Skipping BTF generation for /usr/src/nvidia-550.90.07/kernel/nvidia-drm.ko due to unavailability of vmlinux
Skipping BTF generation for /usr/src/nvidia-550.90.07/kernel/nvidia-modeset.ko due to unavailability of vmlinux
Skipping BTF generation for /usr/src/nvidia-550.90.07/kernel/nvidia.ko due to unavailability of vmlinux
Skipping BTF generation for /usr/src/nvidia-550.90.07/kernel/nvidia-uvm.ko due to unavailability of vmlinux
Relinking NVIDIA driver kernel modules...
Building NVIDIA driver package nvidia-modules-5.15.0-113...
Installing NVIDIA driver kernel modules...

WARNING: The nvidia-drm module will not be installed. As a result, DRM-KMS will not function with this installation of the NVIDIA driver.


ERROR: Unable to open 'kernel/dkms.conf' for copying (No such file or directory)


Welcome to the NVIDIA Software Installer for Unix/Linux

Detected 8 CPUs online; setting concurrency level to 8.
Unable to locate any tools for listing initramfs contents.
Unable to scan initramfs: no tool found
Installing NVIDIA driver version 550.90.07.
...
Loading ipmi and i2c_core kernel modules...
Loading NVIDIA driver kernel modules...
+ modprobe nvidia
+ modprobe nvidia-uvm
+ modprobe nvidia-modeset
+ set +o xtrace -o nounset
Starting NVIDIA persistence daemon...
Mounting NVIDIA driver rootfs...
Done, now waiting for signal

```

为了安装内核驱动，提供了privileged

```
kubectl get pod nvidia-driver-daemonset-wm2h7 -n gpu-operator -o yaml
apiVersion: v1
kind: Pod
...
spec:
...
  containers:
  - args:
    - init
    command:
    - nvidia-driver
    image: nvcr.io/nvidia/driver:550.90.07-ubuntu22.04
    imagePullPolicy: IfNotPresent
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - rm -f /run/nvidia/validations/.driver-ctr-ready
    name: nvidia-driver-ctr
    resources: {}
    securityContext:
      privileged: true
      seLinuxOptions:
        level: s0
    startupProbe:
      exec:
        command:
        - sh
        - -c
        - nvidia-smi && touch /run/nvidia/validations/.driver-ctr-ready
      failureThreshold: 120
      initialDelaySeconds: 60
      periodSeconds: 10
      successThreshold: 1
      timeoutSeconds: 60
...
  hostPID: true
  initContainers:
  - args:
    - uninstall_driver
    command:
    - driver-manager
...
    image: nvcr.io/nvidia/cloud-native/k8s-driver-manager:v0.6.10
    imagePullPolicy: IfNotPresent
    name: k8s-driver-manager
    resources: {}
    securityContext:
      privileged: true
...
```


#### 4.2.2 nvidia-container-toolkit

nvidia-container-toolkit-daemonset 负责安装 `nvidia-container-toolkit`

```
$ kubectl logs -n gpu-operator nvidia-container-toolkit-daemonset-h2d95
Defaulted container "nvidia-container-toolkit-ctr" out of: nvidia-container-toolkit-ctr, driver-validation (init)
IS_HOST_DRIVER=false
NVIDIA_DRIVER_ROOT=/run/nvidia/driver
DRIVER_ROOT_CTR_PATH=/driver-root
NVIDIA_DEV_ROOT=/run/nvidia/driver
DEV_ROOT_CTR_PATH=/driver-root
time="2024-12-02T08:16:14Z" level=info msg="Parsing arguments"
time="2024-12-02T08:16:14Z" level=info msg="Starting nvidia-toolkit"
time="2024-12-02T08:16:14Z" level=info msg="Verifying Flags"
time="2024-12-02T08:16:14Z" level=info msg=Initializing
time="2024-12-02T08:16:14Z" level=info msg="Installing toolkit"
time="2024-12-02T08:16:14Z" level=info msg="disabling device node creation since --cdi-enabled=false"
time="2024-12-02T08:16:14Z" level=info msg="Installing NVIDIA container toolkit to '/usr/local/nvidia/toolkit'"
...
$ lsmod |grep nvidia
nvidia_modeset       1343488  0
nvidia_uvm           4665344  2
nvidia              54206464  12 nvidia_uvm,nvidia_modeset
drm                   622592  4 drm_kms_helper,nvidia,cirrus
$ kubectl exec nvidia-driver-daemonset-wm2h7 -n gpu-operator -- nvidia-smi
Mon Dec  2 09:31:13 2024       
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 550.90.07              Driver Version: 550.90.07      CUDA Version: 12.4     |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                        |               MIG M. |
|=========================================+========================+======================|
|   0  Tesla T4                       On  |   00000000:00:0D.0 Off |                    0 |
| N/A   28C    P8              9W /   70W |       1MiB /  15360MiB |      0%      Default |
|                                         |                        |                  N/A |
+-----------------------------------------+------------------------+----------------------+
                                                                                         
+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI        PID   Type   Process name                              GPU Memory |
|        ID   ID                                                               Usage      |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

nvidia-container-toolkit 被安装到 `/usr/local/nvidia/toolkit`

```
$ find / -name "nvidia-container-cli"
find: File system loop detected; ‘/run/nvidia/driver/run/nvidia’ is part of the same file system loop as ‘/run/nvidia’.
/usr/local/nvidia/toolkit/nvidia-container-cli
/mnt/paas/runtime/overlay2/cf651f978de937f7e91decd6936c32ebe642ac377c69792cbf4c352ea621e343/diff/usr/bin/nvidia-container-cli
/mnt/paas/runtime/overlay2/6c59e6cca5c6c0d5360ca0bde5b69e971d23aed9ef98b8e4fd7f5c0d54e89a39/merged/usr/bin/nvidia-container-cli
$ ls /usr/local/nvidia/toolkit/
libnvidia-container-go.so.1       libnvidia-container.so.1.16.1  nvidia-container-cli       nvidia-container-runtime.cdi       nvidia-container-runtime-hook.real    nvidia-container-runtime.real  nvidia-ctk.real
libnvidia-container-go.so.1.16.1  nvidia-cdi-hook                nvidia-container-cli.real  nvidia-container-runtime.cdi.real  nvidia-container-runtime.legacy       nvidia-container-toolkit
libnvidia-container.so.1          nvidia-cdi-hook.real           nvidia-container-runtime   nvidia-container-runtime-hook      nvidia-container-runtime.legacy.real  nvidia-ctk
```

nvidia-container-toolkit-daemonset 挂载了相关的主机目录，将nvidia-container-toolkit相关文件安装到这些目录。

```
$ kubectl get pod -n gpu-operator nvidia-container-toolkit-daemonset-h2d95 -o yaml
apiVersion: v1
kind: Pod
...
spec:
...
  containers:
  - args:
    - /bin/entrypoint.sh
    command:
    - /bin/bash
    - -c
...
    image: nvcr.io/nvidia/k8s/container-toolkit:v1.16.1-ubuntu20.04
    imagePullPolicy: IfNotPresent
    name: nvidia-container-toolkit-ctr
    resources: {}
    securityContext:
      privileged: true
      seLinuxOptions:
        level: s0
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /bin/entrypoint.sh
      name: nvidia-container-toolkit-entrypoint
      readOnly: true
      subPath: entrypoint.sh
    - mountPath: /run/nvidia/toolkit
      name: toolkit-root
    - mountPath: /run/nvidia/validations
      name: run-nvidia-validations
    - mountPath: /usr/local/nvidia
      name: toolkit-install-dir
    - mountPath: /usr/share/containers/oci/hooks.d
      name: crio-hooks
    - mountPath: /driver-root
      mountPropagation: HostToContainer
      name: driver-install-dir
    - mountPath: /host
      mountPropagation: HostToContainer
      name: host-root
      readOnly: true
    - mountPath: /var/run/cdi
      name: cdi-root
    - mountPath: /runtime/config-dir/
      name: docker-config
    - mountPath: /runtime/sock-dir/
      name: docker-socket
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-ll77d
      readOnly: true
...
  hostPID: true
  initContainers:
  - args:
    - nvidia-validator
    command:
    - sh
    - -c
...
    image: nvcr.io/nvidia/cloud-native/gpu-operator-validator:v24.6.1
    imagePullPolicy: IfNotPresent
    name: driver-validation
    resources: {}
    securityContext:
      privileged: true
      seLinuxOptions:
        level: s0
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /run/nvidia/driver
      mountPropagation: HostToContainer
      name: driver-install-dir
    - mountPath: /run/nvidia/validations
      mountPropagation: Bidirectional
      name: run-nvidia-validations
    - mountPath: /host
      mountPropagation: HostToContainer
      name: host-root
      readOnly: true
    - mountPath: /host-dev-char
      name: host-dev-char
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-ll77d
      readOnly: true
 ...
```

### 4.3 K8S Plugin

**k8s-device-plugin**

https://github.com/NVIDIA/k8s-device-plugin

* 公开集群每个节点上的 GPU 数量
* 跟踪 GPU 的运行状况
* 在 Kubernetes 集群中运行支持 GPU 的容器。

**gpu-feature-discovery**

https://github.com/NVIDIA/k8s-device-plugin/tree/main/cmd/gpu-feature-discovery

NVIDIA GPU Feature Discovery（GPU 特性发现）是一个用于 Kubernetes 的软件组件，它可以自动为节点上可用的 GPU 集生成标签。

这一功能主要用于自动识别并公开节点上的 GPU 特性，例如 CUDA 驱动版本、GPU 型号、内存大小等，这些标签可用于 Kubernetes 集群中的调度决策，以确保应用或服务被调度到具备所需 GPU 特性的节点上。例如，某些特定的机器学习框架可能需要特定版本的 CUDA 支持，通过 GPU Feature Discovery，可以确保这些框架仅部署到支持相应 CUDA 版本的节点上。

### 4.4 Monitoring

收集 metrics

```
$ kubectl logs nvidia-dcgm-exporter-86vc6 -n gpu-operator
Defaulted container "nvidia-dcgm-exporter" out of: nvidia-dcgm-exporter, toolkit-validation (init)
2024/12/02 08:18:27 maxprocs: Leaving GOMAXPROCS=8: CPU quota undefined
time="2024-12-02T08:18:27Z" level=info msg="Starting dcgm-exporter"
time="2024-12-02T08:18:27Z" level=info msg="DCGM successfully initialized!"
time="2024-12-02T08:18:27Z" level=info msg="Not collecting DCP metrics: This request is serviced by a module of DCGM that is not currently loaded"
time="2024-12-02T08:18:27Z" level=info msg="Falling back to metric file '/etc/dcgm-exporter/dcp-metrics-included.csv'"
time="2024-12-02T08:18:27Z" level=warning msg="Skipping line 20 ('DCGM_FI_PROF_GR_ENGINE_ACTIVE'): metric not enabled"
time="2024-12-02T08:18:27Z" level=warning msg="Skipping line 21 ('DCGM_FI_PROF_PIPE_TENSOR_ACTIVE'): metric not enabled"
time="2024-12-02T08:18:27Z" level=warning msg="Skipping line 22 ('DCGM_FI_PROF_DRAM_ACTIVE'): metric not enabled"
time="2024-12-02T08:18:27Z" level=warning msg="Skipping line 23 ('DCGM_FI_PROF_PCIE_TX_BYTES'): metric not enabled"
time="2024-12-02T08:18:27Z" level=warning msg="Skipping line 24 ('DCGM_FI_PROF_PCIE_RX_BYTES'): metric not enabled"
time="2024-12-02T08:18:27Z" level=info msg="Initializing system entities of type: GPU"
time="2024-12-02T08:18:27Z" level=info msg="Not collecting GPU metrics; Error getting devices count: Cannot perform the requested operation because NVML doesn't exist on this system."
time="2024-12-02T08:18:27Z" level=info msg="Not collecting NvSwitch metrics; no fields to watch for device type: 3"
time="2024-12-02T08:18:27Z" level=info msg="Not collecting NvLink metrics; no fields to watch for device type: 6"
time="2024-12-02T08:18:27Z" level=info msg="Not collecting CPU metrics; no fields to watch for device type: 7"
time="2024-12-02T08:18:27Z" level=info msg="Not collecting CPU Core metrics; no fields to watch for device type: 8"
time="2024-12-02T08:18:27Z" level=info msg="Kubernetes metrics collection enabled!"
time="2024-12-02T08:18:27Z" level=info msg="Pipeline starting"
time="2024-12-02T08:18:27Z" level=info msg="Starting webserver"
time="2024-12-02T08:18:27Z" level=info msg="Listening on" address="[::]:9400"
time="2024-12-02T08:18:27Z" level=info msg="TLS is disabled." address="[::]:9400" http2=false
```

## 5. 总结

还有一些高级功能，我没有继续学习。

从安全角度看，为了安装驱动，设置了很多高权限的daemonset, 这可能会带来一些风险，有待进一步分析。

> This document use [Cloud Native Security News: Researcher Introduction](https://github.com/ssst0n3/security-research-specification/blob/main/cloud-native-security-news/security-conference-talk-learning.md) as template.
