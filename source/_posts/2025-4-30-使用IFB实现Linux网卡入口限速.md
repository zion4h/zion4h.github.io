---
title: 使用 Linux 流量控制（ifb+tc）实现 Host 上下行带宽管理实录
date: 2025-04-30 14:52:44
categories: [Linux, Network]
tags: [tc, ifb, 流控，带宽管理]
toc: true
---

为实现对 Host 的上下行速度控制，需要借助 Linux 流量控制相关工具。Linux 流量控制包含三个部分：**流量分类、流量标记、流量策略**。

我主要参考了 [ArchWiki Advanced traffic control](https://wiki.archlinux.org/title/Advanced_traffic_control)。资料中提到需要禁用 **TCP 分段卸载（TSO）**——否则它会绕过 tc 去节省 CPU（实际上是“负优化”）。不过，由于我本身用 ifb + tbf 提前处理了流量，这个问题表面上看并未暴露，但为保险起见，我依然把每个 host 的 tso 都关掉了。（参考：[SUSE 关于 TSO 关闭说明](https://www.suse.com/support/kb/doc/?id=000016894)）

<!--more-->

---
## 前言：为什么要给宿主机“系上安全带”？

这台物理服务器运行着 8 个 KVM 虚拟机，分别承载实时推理与文件同步等业务。当虚拟机切换模型或版本时，需要从主机预置的模型仓库拉取大体积文件。为了减少公网带宽浪费，我把「模型下载、镜像更新」统一下沉到宿主机，由它先行完成外网下载，再让虚拟机通过 10 GbE 内部网络极速同步（实际测试约 380MBps+， 接近磁盘速度）。

问题很快出现：

1. 宿主机在短时间内“满血”跑 wget/aria2c，公网出口被 200 Mbit/s​+ 的流量抢占。
2. 虚拟机的日常业务——尤其是用户上传日志、实时推理结果回传——立刻遭遇高 RTT、包重传甚至超时。
3. 简单的 trickle、aria2c --max-download-limit 无法覆盖所有场景，比如重装系统时 dnf/apt 依赖下载同样会把带宽吃光。

因此，我希望满足三点需求：
- 虚拟机完全不限速，继续用满带宽做业务；
- 宿主机出网（上传）≤ 50 Mbit/s，入网（下载）≤ 100 Mbit/s；
- 方案可脚本化、一次配置后随系统启动自动生效。

Linux Traffic Control（tc）配合 IFB 虚拟网卡正好满足这一目标：
- 用 tbf/qdisc 对宿主机 egress 做简单上传限速；
- 用 ingress + mirred redirect + ifb 把宿主机 download 流量“转送”到虚拟网卡上，再做下行限速；
- 对同一物理网卡上绑着的 KVM vNIC 不做任何规则，让它们继续全速跑。

---

## IFB 原理与数据通路

为什么需要 IFB？因为 Linux 内核在 **ingress** 方向只能做 *policing*（丢包），不能做 *shaping*（排队、匀速发包）。借助 IFB，我们把“入口流量”变成“出口流量”，就能挂任何 egress-qdisc（TBF/HTB/FQ-CodeL 等）来精细限速。

数据流向可分五步（示例网卡 `eno1`，虚拟网卡 `ifb0`）：

```
+-------+   +------+                 +------+
|ingress|   |egress|   +---------+   |egress|
|qdisc  +--->qdisc +--->netfilter+--->qdisc |
|eth1   |   |ifb1  |   +---------+   |eth1  |
+-------+   +------+                 +------+
```

逐项拆解：

1. NIC 收到下行数据包，驱动把 skb 交给内核。
2. 在 `eno1` 的 `ingress qdisc`（句柄默认 `ffff:`）上执行 u32 filter；  
   我们用 `mirred egress redirect dev ifb0` 把 skb 的 `dev` 字段改写成 `ifb0` 并“重新注入”协议栈，等价于“流量被扔进另一张虚拟网卡”。
3. skb 现在是以 **ifb0 → egress** 的身份继续向前，因此能套上任何 egress-qdisc；我们对这里做 *shaping*，达到“宿主机下载限速”目的。
4. 排完队的 skb 进入路由子系统（和普通本机发包无异），决定还是走 `eno1`。
5. 最终到达 `eno1` 的 egress qdisc（此处我们另外挂 TBF 做“上传限速”），再交给驱动下发。

关键事实与常见误解
- IFB 并不能“提前”截流；包必须先进入物理网卡，才能被镜像到 IFB。  
- 经过 mirred 的 skb 会重新走一次 netfilter，因此 iptables / nftables 统计会把这部分包算作“本地产生”，不要被双计数吓到。  
- ingress qdisc 只能做 policer（如 `police rate 100mbit drop`）；真正的排队靠 ifb。  
- TSO/GRO/LRO 若未关闭，仍可能让大包绕过 qdisc，生产环境建议全关或至少关掉 TSO/GSO。

参考阅读  
- kernel doc `Documentation/networking/ifb.rst` [link](https://wiki.linuxfoundation.org/networking/ifb)
- Thomas Graf《Linux Traffic Control – Mirred Action Deep Dive》 
- [StackExchange: “How is the IFB device positioned in the packet flow of the Linux kernel?”](https://unix.stackexchange.com/questions/288959/how-is-the-ifb-device-positioned-in-the-packet-flow-of-the-linux-kernel)

---

## 需求与网络结构

本例需求：

- VM 不限制上下行带宽；
- Host 限制到约 50% —— 下行约 100Mbps，上行约 50Mbps；

当前 VM 使用 ensp35（48g-3）或 eno2（48g-4）网卡，Host 都统一走 eno1，天然实现了分流，简化了配置工作。

- **出口上传带宽限制**非常简单，通过 qdisc 队列即可：
    - （一条命令搞定）
- **入口下载带宽限制**较难，在物理层面通常需要通过路由器丢包实现；但本地可用 ifb 虚拟网卡来处理进入 eno1 的流量。

---

## IFB 配置及限速设置流程

### 1. 加载 ifb 模块与激活虚拟网卡

```sh
modprobe ifb
ip link add ifb0 type ifb
ip link set ifb0 up
```

### 2. 重定向 eno1 入站流量到 ifb0

```sh
# 加入口方向 qdisc 到 eno1
tc qdisc add dev eno1 handle ffff: ingress

# 将 eno1 入站数据全部重定向输出到 ifb0
tc filter add dev eno1 parent ffff: protocol ip u32 match u32 0 0 action mirred egress redirect dev ifb0
```

> tc 支持在不同 qdisc 分支（如 root: 或 parent ffff:）下分别挂载不同 filter。如果直接 `tc filter show dev eno1`，默认显示 root 分支。

### 3. 控制下行速率（即 host 下载速率）

参考：[tc 参数写法速查](https://cheat.sh/tc)

```sh
# 下行测速：以 100mbit 示例，可自行调整
tc qdisc add dev ifb0 root tbf rate 100mbit burst 32kbit latency 400ms
```

### 4. 控制上传速度（host 出口速率）

```sh
tc qdisc add dev eno1 root tbf rate 50mbit burst 32kbit latency 400ms
```

---

## 注意规则增删及调试

流控规则不能直接修改，只能按分支类型、方向等删除后重加。如：

```sh
# 删除原有规则后再加新规则
tc qdisc del dev eno1 root      
tc qdisc del dev eno1 ingress   
tc qdisc del dev ifb0 root      
```

---

## 测试方法

- 外网速度测试：使用 [Speedtest 网页](https://www.speedtest.net/)，当然 Linux 用 speedtest-cli 即可。
- 内网测速工具：使用 iperf3 和另一台内网设备（如 Mac Pro）

示例：

```sh
# 对方 (Mac Pro) 开启服务端：
iperf3 -s

# 本机（客户机）作为客户端测速：
iperf3 -c <服务器 IP>
```

---

## 总结及常见坑

- 推荐关闭所有物理网卡 TSO 等加速特性，避免内核旁路 tc：[操作指引](https://www.suse.com/support/kb/doc/?id=000016894)
- 各种流控参数和复杂用法可查阅  
    - [ArchWiki Traffic Control 指南](https://wiki.archlinux.org/title/Advanced_traffic_control)  
    - [TC 官方文档 & 用法详解](http://linux-ip.net/gl/tc-filters/tc-filters-node3.html)  
    - [命令在线速查 cheat.sh/tc](https://cheat.sh/tc)

---

如有疑问欢迎评论交流。