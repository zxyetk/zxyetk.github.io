---
title: CentOS 安装 Kubernetes 集群
date: 2019-08-02 20:33:54
tags:
  - kubernetes
  - centos
  - k8s
menu:
  main:
    parent: posts
---

## 准备工作

- 一台或多台运行着下列系统的机器:
  - CentOS 7
  - RHEL 7
- 每台机器 2 GB 或更多的 RAM (如果少于这个数字将会影响您应用的运行内存)
- 2 CPU 核心或更多
- 集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)
- 节点之中不可以有重复的主机名，MAC 地址，product_uuid。更多详细信息请参见[这里](https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/#verify-the-mac-address-and-product-uuid-are-unique-for-every-node) 。
- 开启主机上的一些特定端口. 更多详细信息请参见[这里](https://kubernetes.io/zh/docs/setup/independent/install-kubeadm/#check-required-ports)。
- 禁用 Swap 交换分区。为了保证 kubelet 正确运行，您 **必须** 禁用交换分区。

<!--more-->

### 确保每个节点上 MAC 地址和 product_uuid 的唯一性。

- 您可以使用下列命令获取网络接口的 MAC 地址：`ip link` 或是 `ifconfig -a`
- 下列命令可以用来获取 product_uuid `sudo cat /sys/class/dmi/id/product_uuid`

一般来讲，硬件设备会拥有独一无二的地址，但是有些虚拟机可能会雷同。Kubernetes 使用这些值来唯一确定集群中的节点。如果这些值在集群中不唯一，可能会导致安装[失败](https://github.com/kubernetes/kubeadm/issues/31)。

### 检查网络适配器

如果您有一个以上的网络适配器，同时您的 Kubernetes 组件通过默认路由不可达，我们建议您预先添加 IP 路由规则，这样 Kubernetes 集群就可以通过对应的适配器完成连接。

### 检查所需端口

#### Master 节点

| 规则 | 方向    | 端口范围  | 作用                    | 使用者               |
| :--- | :------ | :-------- | :---------------------- | :------------------- |
| TCP  | Inbound | 6443*     | Kubernetes API server   | All                  |
| TCP  | Inbound | 2379-2380 | etcd server client API  | kube-apiserver, etcd |
| TCP  | Inbound | 10250     | Kubelet API             | Self, Control plane  |
| TCP  | Inbound | 10251     | kube-scheduler          | Self                 |
| TCP  | Inbound | 10252     | kube-controller-manager | Self                 |

#### Worker 节点

| 规则 | 方向    | 端口范围    | 作用                | 使用者              |
| :--- | :------ | :---------- | :------------------ | :------------------ |
| TCP  | Inbound | 10250       | Kubelet API         | Self, Control plane |
| TCP  | Inbound | 30000-32767 | NodePort Services** | All                 |

`**` [NodePort 服务](https://kubernetes.io/docs/concepts/services-networking/service/) 的默认端口范围。

任何使用 `*` 标记的端口号都有可能被覆盖，所以您需要保证您的自定义端口的状态是开放的。

虽然主节点已经包含了 etcd 的端口，您也可以使用自定义的外部 etcd 集群，或是指定自定义端口。 您使用的 pod 网络插件 (见下) 也可能需要某些特定端口开启。由于各个 pod 网络插件都有所不同，请参阅他们各自文档中对端口的要求。

### 安装 kubeadm, kubelet 和 kubectl

您需要在每台机器上都安装以下的软件包：

- `kubeadm`: 用来初始化集群的指令。
- `kubelet`: 在集群中的每个节点上用来启动 pod 和 container 等。
- `kubectl`: 用来与集群通信的命令行工具。

kubeadm **不能** 帮您安装或管理 `kubelet` 或 `kubectl` ，所以您得保证他们满足通过 kubeadm 安装的 Kubernetes 控制层对版本的要求。如果版本没有满足要求，就有可能导致一些难以想到的错误或问题。然而控制层与 kubelet 间的 *小版本号* 不一致无伤大雅，不过请记住 kubelet 的版本不可以超过 API server 的版本。例如 1.8.0 的 API server 可以适配 1.7.0 的 kubelet，反之就不行了。

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# 将 SELinux 设置为 permissive 模式(将其禁用)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
```

**请注意：**

\- 通过命令 `setenforce 0` 和 `sed ...` 可以将 SELinux 设置为 permissive 模式(将其禁用)。 只有执行这一操作之后，容器才能访问宿主的文件系统，进而能够正常使用 Pod 网络。您必须这么做，直到 kubelet 做出升级支持 SELinux 为止。 - 一些 RHEL/CentOS 7 的用户曾经遇到过：由于 iptables 被绕过导致网络请求被错误的路由。您得保证 在您的 `sysctl` 配置中 `net.bridge.bridge-nf-call-iptables` 被设为1。

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

kubelet 现在每隔几秒就会重启，因为它陷入了一个等待 kubeadm 指令的死循环。

### 在 Master 节点上配置 kubelet 所需的 cgroup 驱动

使用 Docker 时，kubeadm 会自动为其检测 cgroup 驱动在运行时对 `/var/lib/kubelet/kubeadm-flags.env` 文件进行配置。 如果您使用了不同的 CRI， 您得把 `/etc/default/kubelet` 文件中的 `cgroup-driver` 位置改为对应的值，像这样：

```bash
KUBELET_EXTRA_ARGS=--cgroup-driver=<value>
```

这个文件将会被 `kubeadm init` 和 `kubeadm join` 用于为 kubelet 获取 额外的用户参数。

请注意，您**只**需要在您的 cgroup driver 不是 `cgroupfs` 时这么做，因为 `cgroupfs` 已经是 kubelet 的默认值了。

需要重启 kubelet：

```bash
systemctl daemon-reload
systemctl restart kubelet
```

# 使用 kubeadm 创建一个单主集群

### 