---
title: 服务器 KVM 虚拟化 + GPU 直通部署记录  
date: 2025-04-19 21:54:57  
cover: https://i.imgur.com/BjwkNkb.jpg
thumbnail: https://i.imgur.com/BjwkNkb.jpg
tags: [KVM, 虚拟化]  
toc: true  
---

公司新到一批 8 卡 48 GB RTX 4090 服务器，用于深度学习任务的多租户调度。本文记录从系统安装、磁盘分区、GPU 直通到虚拟机克隆全过程中踩过的坑与解决方案，以备后查，也希望能给同样需求的同学一些参考。  

<!--more-->

## 1. `virt-install` 镜像加载报错

初次执行

```bash
virt-install ... --boot hd ...
```

却收到

```
ERROR
An install method must be specified
(--location URL, --cdrom CD/ISO, --pxe, --import, --boot hd|cdrom|...)
```

原因是仅指定 `--boot hd` 并不足以告诉 libvirt *安装介质* 从何而来。  
在命令末尾补一个 “空导入” 即可解决：

```bash
--import
```

等于告诉 `virt-install`：磁盘里已有系统，把它直接拿来启动即可。

---

## 2. CUDA 与 Anaconda 环境变量陷阱

最常见的错法是把 PATH 写成硬编码：

```bash
export PATH=/usr/local/cuda-12.4/bin:/usr/local/sbin:/usr/bin
```

这样一旦 shell 里已有 PATH，会被完全覆盖。推荐写成如下“追加”形式：

```bash
echo 'export PATH=/usr/local/cuda-12.4/bin${PATH:+:${PATH}}' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.4/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}' >> ~/.bashrc
```

再次登录即可生效。

---

## 3. 服务器硬件与基础信息

- CPU：2× Intel Xeon 8452Y（共 64C128T）  
- RAM：1 TB  
- SSD：2× 3.84 TB NVMe  
- GPU：8× RTX 4090 48 GB  

快速验证：

```bash
nproc                                # ==> 128
free -h | awk '/^Mem:/ {print $2}'   # ==> ~1012G
lsblk -d -o NAME,SIZE,MODEL | grep nvme
```

GPU bus ID：

下面的 GPU 的总线需要记住，通常一张显卡包含两个 ID，一个是本体 ID，另一个是音频口 ID，在直通时都要保证两个 ID 在同一个 IOMMU 组内，以及均未被其他驱动占用。

```bash
lspci -nn | grep -i nvidia
# 01:00.0, 26:00.0, 41:00.0, 61:00.0, 81:00.0, a1:00.0, c1:00.0, e1:00.0
```

可用 `lspci -k -s <bus-id>` 查看当前驱动归属防止被占用：

```bash
lspci -k -s 23:00.0
```

---

## 4. NVMe 分区与挂载

将磁盘格式化为ext4（why？）后，查看其uuid，在写入文件表后将它挂载。

```bash
parted /dev/nvme0n1
mklabel gpt
mkpart primary 0% 100%
quit

mkfs.ext4 -L data_nvme0 /dev/nvme0n1p1

mkdir -p /mnt/nvme0
blkid /dev/nvme0n1p1 # 查看 UUID
echo 'UUID=5e9c468b-185f-48d8-b9f5-34c52e874d54 /mnt/nvme0 ext4 defaults 0 0' >> /etc/fstab
mount -a && df -h /mnt/nvme0
```

---

## 5. GPU 直通（VFIO）

1. 指定要由 VFIO 接管的设备 ID：

   ```bash
   echo "options vfio-pci ids=10de:2684,10de:22ba" > /etc/modprobe.d/vfio.conf
   ```

   - 10de:2684：RTX 4090 显卡  
   - 10de:22ba：对应的 HD‑Audio 控制器

2. 屏蔽宿主机自带驱动：

   ```bash
   echo -e "blacklist nouveau\nblacklist nvidia\nblacklist snd_hda_intel" >> /etc/modprobe.d/blacklist.conf
   update-initramfs -u
   ```

3. 热插拔示例（以 01:00.0 为例）：

   ```bash
   systemctl stop nvidia-persistenced
   modprobe -r nvidia_drm nvidia_modeset nvidia_uvm

   echo vfio-pci > /sys/bus/pci/devices/0000:01:00.0/driver_override
   echo 0000:01:00.0 > /sys/bus/pci/drivers/vfio-pci/bind

   # 同理处理 01:00.1 (audio)
   ```

其余七张卡循环同样步骤即可。

---

## 6. 创建基础虚拟机并批量克隆

安装依赖与镜像：

```bash
apt update && apt install -y qemu-kvm libvirt-daemon virtinst bridge-utils virt-manager libguestfs-tools
wget -P /var/lib/libvirt/images https://releases.ubuntu.com/jammy/ubuntu-22.04.5-live-server-amd64.iso
mkdir -p /mnt/nvme0/vm-storage
```

创建模板 VM（直通首张显卡）：

```bash
virt-install --name vmbase \
--os-variant ubuntu20.04 \
--vcpus 24 \
--memory 102400 \
--location ubuntu-22.04.5-live-server-amd64.iso,kernel=casper/vmlinuz,initrd=casper/initrd \
--graphics none \
--disk path=/mnt/nvme0/vm-storage/vmbase.qcow2,size=400 \
--network bridge=vmbr0\
--host-device 01:00.0 \
--features kvm_hidden=on \
--console pty,target_type=serial \
--extra-args='console=ttyS0,115200n8 serial'
```

## 7. 网络与 SSH 隧道配置

### 1. iptables 端口转发

宿主机公网只有 22 端口，而每台 VM 都需要独立 SSH。做法是把宿主机的非标准端口映射到各 VM：

```text
VM 列表
192.168.122.171/24
192.168.122.252/24
…
192.168.122.109/24
```

举例：把宿主机的 12223 → 192.168.122.167:22

```bash
# DNAT：修改目的地址
iptables -t nat -A PREROUTING -p tcp --dport 12223 \
        -j DNAT --to-destination 192.168.122.167:22

# FORWARD：放行转发流量
iptables -A FORWARD -p tcp -d 192.168.122.167 --dport 22 -j ACCEPT
```

其余 VM 的端口依次重复。

### 2. 本地 SSH 隧道测试

```bash
# 建立本地 12223 → 宿主机 (125.71.97.167):12223 的隧道
ssh -f -N -L 12223:192.168.122.167:22 luser@125.71.97.167

# 直接指定映射端口登陆（例如 12226）
ssh root@125.71.97.167 -p 12226
```

---

## 8.snd_hda_intel 占用音频接口导致直通失败

`RTX 4090` 带一条 PCI 子函数 `10de:22ba` (HD‑Audio)。若被 `snd_hda_intel` 抢占，VFIO 绑定会失败。处理步骤：

```bash
# 立即卸载
modprobe -r snd_hda_intel

# 永久屏蔽
echo "blacklist snd_hda_intel" >> /etc/modprobe.d/blacklist.conf
update-initramfs -u
```

如重启后仍有其它模块占用，可再执行一次：

```bash
modprobe -r nvidia_drm nvidia_modeset nvidia_uvm nvidia snd_hda_intel
```

可用 `lspci -k -s <bus-id>` 查看当前驱动归属：

```bash
lspci -k -s 23:00.0
```

---

## 9. 后期磁盘扩容与挂载流程

如果前期对磁盘大小考虑比较保守，或者后期发展出现磁盘不足的情况，都需要对现有虚拟机进行扩容。qemu 对该部分支持比较好，在 host 上扩容后，进入 vm 进行分配即可。

参考：[14.10. Re-sizing the Disk Image](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/virtualization_deployment_and_administration_guide/sect-using_qemu_img-re_sizing_the_disk_image)

1. 在宿主机上给虚拟机增加存储空间

   ```bash
   qemu-img resize filename +50G
   ```

2. 进入虚拟机后执行`fdisk -l`查看是否有增量存储未分配

   ```bash
   fdisk -l

   lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv 
   resize2fs /dev/ubuntu-vg/ubuntu-lv
   # 或者
   growpart /dev/vda 3
   ```

---

## 10.virtiofs 主机 ↔︎ 虚拟机高速共享

nfs 简单但延迟高；内核 ≥ 5.14 + libvirt ≥ 8.0 可直接用 virtiofs，性能接近本机 I/O。 

参考文档：[Sharing files with Virtiofs](https://libvirt.org/kbase/virtiofs.html)

1. 修改 VM XML（示例片段）：

```xml
<domain type='kvm'>
  …
  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>

  <devices>
    …
    <filesystem type='mount' accessmode='passthrough'>
      <driver type='virtiofs' queue='1024'/>
      <source dir='/mnt/nvme0/share'/>
      <target dir='shared_folder_tag'/>
    </filesystem>
    …
  </devices>
</domain>
```

2. 虚拟机内部挂载：

个人更喜欢将共享文件夹作为临时“通道”，所以会将文件拷贝到本地，避免高负载下 Host 的 IO 压力过大。

```bash
mount -t virtiofs shared_folder_tag /mnt/shared
cp -r /mnt/shared/data /root/
```

---

## 11. 总结

至此，**8 卡 RTX 4090 KVM 集群** 已完成：

1. NVMe 分区与挂载  
2. VFIO 直通显卡 + HD‑Audio  
3. Base VM 制作、批量克隆  
4. iptables 端口映射 + 本地 SSH 隧道  
5. 驱动占用、磁盘扩容、virtiofs 文件共享等细节优化  

过程中遇到的所有坑、完整命令与验证方式均已记录。如有疑问或更优实践，欢迎留言交流。