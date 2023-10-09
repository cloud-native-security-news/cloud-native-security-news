---

tags: 云原生安全资讯,项目介绍
spec version: v0.2.4
version: v0.1.0

---

# 项目介绍: rootlesskit

* 项目地址：https://github.com/rootless-containers/rootlesskit

## 1. What

RootlessKit是使用user_namespaces(7)实现的Linux原生 "fake root" 工具。

## 2. Why

RootlessKit 是为了以非特权用户运行 Docker 和 Kubernetes 而设计的，以保护主机上真正的root用户，来应对可能的容器逃逸攻击。

一些类似的工具不能支撑运行 rootless 容器，或缺少相关功能：

基于 LD_PRELOAD 的工具 (不足以运行 rootless 容器，不支持静态二进制文件) :

* [fakeroot](https://wiki.debian.org/FakeRoot)

基于系统调用 ptrace(2) 的工具 (不足以支撑 rootless 容器)：

* [fakeroot-ng](https://fakeroot-ng.lingnu.com/)
* [proot](https://proot-me.github.io/)

同 RootlessKit 一样，基于 user_namespace(7) 实现的工具(但不支持 --copy-up, --net 等功能)：

* [unshare -r](http://man7.org/linux/man-pages/man1/unshare.1.html)
* [podman unshare](https://github.com/containers/libpod/blob/master/docs/source/markdown/podman-unshare.1.md)
* [become-root](https://github.com/giuseppe/become-root)

## 3. How 

### 3.1 How to use

依赖

```
root@m:~# apt install uidmap
root@m:~# cat /proc/sys/kernel/unprivileged_userns_clone 
1
```

安装 rootlesskit

```
test@m:~$ mkdir -p bin
test@m:~$ curl -sSL https://github.com/rootless-containers/rootlesskit/releases/download/v1.1.1/rootlesskit-$(uname -m).tar.gz | tar Cxzv bin
rootlessctl
rootlesskit
rootlesskit-docker-proxy
```

试用

```
test@m:~$ id
uid=1000(test) gid=1000(test) groups=1000(test)
test@m:~$ ./bin/rootlesskit bash
root@m:~# id
uid=0(root) gid=0(root) groups=0(root)
root@m:~# ls -l /etc/shadow
-rw-r----- 1 nobody nogroup 1160 Oct  7 11:36 /etc/shadow
root@m:~# cat /etc/shadow
cat: /etc/shadow: Permission denied
```

```
test@m:~$ ./bin/rootlesskit --copy-up=/etc bash
root@m:~# ls -lah /etc/shadow
lrwxrwxrwx 1 root root 20 Oct  7 11:43 /etc/shadow -> .ro3709914014/shadow
```

### 3.2 How it works

#### (1) user namespaces + newuidmap/newgidmap

rootless 容器使用 [user_namespaces(7)](http://man7.org/linux/man-pages/man7/user_namespaces.7.html) 来模拟特权以创建容器。利用 user ns 中的 CAP_SYS_ADMIN 和 CAP_NET_ADMIN 执行特权操作，例如创建 mount namespaces, network namespaces, TAP 设备等。这一点上和 unshare 功能相同

```
test@m:~$ id
uid=1000(test) gid=1000(test) groups=1000(test)
test@m:~$ unshare --user --map-root-user
root@m:~# id
uid=0(root) gid=0(root) groups=0(root)
root@m:~# cat /proc/self/status | grep -i capeff
CapEff: 0000003fffffffff
root@m:~# cat /proc/self/uid_map 
         0       1000          1
```

但仅映射单个 uid/gid 不足以运行需要多个 uid/gid 的容器，rootless 容器使用 SETUID程序 newuidmap/newgidmap 解决这个问题，他们会从 /etc/subuid, /etc/subgid 中读取配置文件。

```
test@m:~$ ./bin/rootlesskit bash
root@m:~# cat /proc/self/uid_map 
         0       1000          1
         1     100000      65536
```

这意味着 user ns 中的 uid 和host uid 的映射关系为 ：

| Host UID | UserNS UID |
| -- | -- |
| 1000 | 0 |
| 100000 | 1 |
| 100001 | 2 |
| … | … |
| 165535 | 65536 |

#### (2) network namespaces

创建 network namespaces 不只是为了分配 IP 地址，还是为了隔离 容器化进程 对主机 UNIX socket 的访问。

然而 vEth 对 不能在没有特权时 跨 userns 创建。LXC 使用 SETUID 程序 lxc-user-nic 来创建 vEth 对； 也可使用 TAP 设备运行一个名为 [slirp](https://rootlesscontaine.rs/glossary#slirp) 的用户模式网络栈，它可以将 Ethernet 数据翻译为 非特权的 socket 系统调用。

rootlesskit 支持多种驱动

* `--net=host`: 共享主机 network namespace (默认)
  * 无权限执行 network-namespaced 操作，例如创建 iptables 规则, 运行 tcpdump
  * 不涉及性能损耗
* `--net=pasta`:  [pasta](https://passt.top/passt/) (experimental)
  * 暂不支持 UDP 端口转发
  * TCP 端口转发速度快，支持保留源IP
* `--net=slirp4netns`:  [slirp4netns](https://github.com/rootless-containers/slirp4netns) (推荐)
  * 基于mount namespace, seccomp 的加固
  * 性能损耗，但优于 VPNKit
  * 仅支持 TCP, UDP, ICMP Echo
* `--net=vpnkit`: [VPNKit](https://github.com/moby/vpnkit)
  * 性能损耗
  * 仅支持 TCP, UDP, 不支持 ICMP Echo
  * 不支持 IPv6
* `--net=lxc-user-nic`: `lxc-user-nic` (experimental)
  * 最低性能损耗
  * 低安全
  * 不支持 IPv6
  * 不支持 `--detach-netns`

## 4. 源码

跳过命令行参数解析，程序运行入口为 `parent.Parent()` 函数

主要流程基本在该函数中，包括：

* unshare namespaces
* 配置网络
* 端口转发

详细代码待后续 rootlesskit 专项研究分析，本文仅作大致介绍。

https://github.com/rootless-containers/rootlesskit/blob/v2.0.0-alpha.1/pkg/parent/parent.go#L128

```go
func Parent(opt Opt) error {
	...
	cmd := exec.Command("/proc/self/exe", os.Args[1:]...)
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Pdeathsig:  syscall.SIGKILL,
		Cloneflags: syscall.CLONE_NEWUSER | syscall.CLONE_NEWNS,
	}

	if opt.NetworkDriver != nil {
		if !opt.DetachNetNS {
			cmd.SysProcAttr.Unshareflags |= syscall.CLONE_NEWNET
		}
	}

	if opt.CreatePIDNS {
		// cannot be Unshareflags (panics)
		cmd.SysProcAttr.Cloneflags |= syscall.CLONE_NEWPID
	}
	if opt.CreateCgroupNS {
		cmd.SysProcAttr.Unshareflags |= unix.CLONE_NEWCGROUP
	}
	if opt.CreateUTSNS {
		cmd.SysProcAttr.Unshareflags |= unix.CLONE_NEWUTS
	}
	if opt.CreateIPCNS {
		cmd.SysProcAttr.Unshareflags |= unix.CLONE_NEWIPC
	}
        ...
	if err := setupUIDGIDMap(cmd.Process.Pid, opt.SubidSource); err != nil {
		return fmt.Errorf("failed to setup UID/GID map: %w", err)
	}
        ...
	if opt.NetworkDriver != nil {
		var netns string
		if opt.DetachNetNS {
			netns = filepath.Join("/proc", strconv.Itoa(cmd.Process.Pid), "root", filepath.Clean(opt.StateDir), "netns")
		}
		netMsg, cleanupNetwork, err := opt.NetworkDriver.ConfigureNetwork(cmd.Process.Pid, opt.StateDir, netns)
		if cleanupNetwork != nil {
			defer cleanupNetwork()
		}
		if err != nil {
			return fmt.Errorf("failed to setup network %+v: %w", opt.NetworkDriver, err)
		}
		msgParentInitNetworkDriverCompleted.U.ParentInitNetworkDriverCompleted = netMsg
	}
        ...
	if opt.PortDriver != nil {
		// wait for port driver to be ready
		select {
		case <-portDriverInitComplete:
		case err = <-portDriverErr:
			return err
		}
		// publish ports
		for _, p := range opt.PublishPorts {
			st, err := opt.PortDriver.AddPort(context.TODO(), p)
			if err != nil {
				return fmt.Errorf("failed to expose port %v: %w", p, err)
			}
			logrus.Debugf("published port %v", st)
		}
	}
        ...
	if err := cmd.Wait(); err != nil {
		return fmt.Errorf("child exited: %w", err)
	}
        ...
}

```

## 5. 参考

* https://github.com/rootless-containers/rootlesskit/
* https://rootlesscontaine.rs/how-it-works/

----

> 本文使用[云原生安全资讯：项目推荐](https://github.com/ssst0n3/security-research-specification/blob/main/cloud-native-security-news/project-introduction.md)作为文档基线
