---

title: nftables 实践——替代 iptables
date: 2025-04-19 21:18:00
cover: https://i.imgur.com/OYCdymy.jpg
thumbnail: https://i.imgur.com/OYCdymy.jpg
categories: [Linux, 防火墙]
toc: true
---

`nftables` 作为 iptables 的接班人，在实际使用中也确实感受到了它的简洁、优雅，最重要的是**装b**！

<!--more-->

## 1. 应用场景

以前用 iptables 常见的需求：

- 让具有私有 IP 的 VM 访问外网（masquerade）
- 将 VM 的 SSH 端口转发到宿主指定端口
- （可选）结合 fail2ban 做自动封禁（ps：据说有个 reaction 是 fail2ban 的轻量替代，但 fail2ban 已经足够傻瓜有效，所以没动力换）

---

## 2. 基本语法与结构

`nft` 命令一览：

```bash
nft add | delete | flush table|chain|rule ...
```

- 保留了 Linux 的“四表五链”设计。详见 [Netfilter_hooks（英文）](https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks)
- 实际用时，大多只关心 **ip 协议族**，所以可以省略（默认即为 ip）
- 创建表 (table) 后可定义链 (chain)，主要关注入口（prerouting）与出口（postrouting）
- 虽然这些属于 `nat` 表的链，但为了可读性、可维护性也可以直接用 `filter` 表，不影响效果
- rule 语法需指定 table/chain，然后写具体 statement
- statement 比较灵活，需要结合实际查阅文档或 `man nft` 页面（详见 [nft.8#STATEMENTS](https://man.archlinux.org/man/nft.8#STATEMENTS)）

![四表五链](https://people.netfilter.org/pablo/nf-hooks.png)

---

## 3. 实践案例

### (1) 让 VM 访问外网（私有 IP NAT 出口）

实现 NAT 出口最常用的就是 masquerade，流程对应 `postrouting` 阶段：

```bash
nft add rule nat postrouting masquerade
```

或带表名完整示例（首次需先建表/链）：

```bash
nft add table nat
nft add chain nat postrouting '{ type nat hook postrouting priority 100; }'
nft add rule nat postrouting masquerade
```

### (2) SSH 端口转发

比如将宿主 12221 端口流量转发到内网 VM 的 22 端口：

```bash
nft add rule nat prerouting tcp dport 12221 dnat to 192.168.1.101:22
```

---

### (3) 关于表名与链名的灵活性

nft 规则中的 `table` 和 `chain` 名字其实没有强制要求，随意起名即可。例如可以用 `小明`、`小红`替代：

```bash
nft add table xiaoming
nft add chain xiaoming xiaohong '{ type nat hook prerouting priority 0; }'
nft add rule xiaoming xiaohong tcp dport 12221 dnat to 192.168.1.101:22
```

作用完全一致。  
**另外注意**：即使在 ssh 会话过程中删除掉该 rule，也不会立即使已建立连接断开。

> **提示**：nftables 默认配置文件通常位于 `/etc/nftables.conf`，重启后会自动加载。临时添加的配置也可以用 `nft list ruleset > /etc/nftables.conf` 备份。

---

## 4. 小结与感想

- 只有简单了解并动手试过，你才会发现：**nft 比 iptables 更现代，也确实更清爽易用**。
- statement 和匹配规则更直观，灵活性强。
- 个人实际体验很友好，尤其便于批量管理和自动化。

---

## 参考资料

1. [ArchWiki: nftables](https://wiki.archlinux.org/title/Nftables)
2. [Gentoo Wiki: nftables/Examples](https://wiki.gentoo.org/wiki/Nftables/Examples)
