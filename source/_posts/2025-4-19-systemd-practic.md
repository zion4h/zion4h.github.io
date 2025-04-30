---
title: Systemd 实践 - KVM 虚拟机自动重启
date: 2025-04-19 18:24:39
categories: [Linux]
cover: https://i.imgur.com/VoPapJr.jpg
thumbnail: https://i.imgur.com/VoPapJr.jpg
toc: true
---
最近遇到一个需求，需要使用 `systemd` 实现 KVM 虚拟机的自动重启功能。

<!--more-->

## 1. 单次任务脚本：自动重启已关闭的虚拟机

首先，编写一个 Shell 脚本：遍历所有关闭（shut off）状态的虚拟机，并尝试启动。

```bash
#!/bin/bash
virsh list --all --state-shutoff --name | while read vm; do 
    echo "尝试启动虚拟机: $vm" 
    virsh start "$vm" 
    if [ $? -eq 0 ]; then 
        echo "成功启动虚拟机: $vm"
        logger -t VM_AUTOSTART "启动记录: $vm"
    else 
        echo "启动失败: $vm" >&2 
    fi
done
```

**注意**：要确保脚本有执行权限，否则无法被 systemd 调用。

```bash
chmod +x /root/vm_autostart.sh
```

---

## 2. 使用 systemd 定时定期执行

systemd 定时任务（timer）通常需要对应的 service 和 timer 两个文件：

### 2.1 服务配置文件

路径：`/etc/systemd/system/vm-autostart.service`

```ini
[Unit]
Description=Check and auto-start KVM VMs in shut off state

[Service]
Type=oneshot
ExecStart=/root/vm_autostart.sh
```

### 2.2 定时器配置文件

路径：`/etc/systemd/system/vm-autostart.timer`

```ini
[Unit]
Description=Auto-check KVM VMs every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
AccuracySec=1s

[Install]
WantedBy=timers.target
```

- 解释：
    - `OnBootSec=5min`            # 启动后延迟 5 分钟运行
    - `OnUnitActiveSec=5min`      # 之后每 5 分钟运行一次
    - `Type=oneshot`              # 表示脚本只执行一次并退出，非常适合自动任务

---

**激活定时器：**

```bash
systemctl daemon-reload
systemctl enable --now vm-autostart.timer
```

> 说明：和常规 long-running 的服务不同，看状态时，`.service` 为 inactive，`.timer` 应为 active（waiting），计时等待下次触发。

---

## 3. 查看日志

可通过如下命令查看服务执行和 logger 输出的日志：

- 查看服务日志：
    ```bash
    journalctl -u vm-autostart.service
    ```

- 查看 logger 输出（使用 Tag 检索）:
    ```bash
    journalctl -t VM_AUTOSTART
    ```

---

**小结：**  
配合 systemd 定时器，实现 KVM 虚拟机宕机自动检测与重启，脚本简洁易扩展，日志也便于追溯。如果有更精细化需求，也可以改写成 Python 或用 ansible playbook 实现。

---
