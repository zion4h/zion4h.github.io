---
title: Prometheus 全链路监控实战：  
       Nginx QPS、ComfyUI 存活、GPU 指标，到 Docker-VPN 网段踩坑记录
date: 2025-06-09 14:00:00
toc: true
tags:
  - Prometheus
  - Docker
  - Nginx
  - GPU
  - Blackbox
categories:
  - Ops
---

> 一篇给团队新人看的「一把梭」笔记：  
> • 打开 Nginx 详细指标（QPS / P99 / 5xx）  
> • 用 Blackbox 监控 ComfyUI 是否挂掉  
> • GPU 机器指标怎么进 Prometheus  
> • Docker-Compose + VPN 网段冲突到底怎么解决  
> • 最后把冗长的 Prometheus YAML 缩成 1/2  
>  
> **能抄就抄，能跑就行**。

<!--more-->

---

## 0. 环境 & 踩坑摘要

| 角色 | 说明 |
| ---- | ---- |
| 宿主机 | Ubuntu 24.04 + Docker 28.2.2 |
| VPN   | 公司 XVPN，出口段 `172.21.0.0/16` |
| Compose | 5 套（4 业务 + 1 监控），每套原本都会随机生成 `br-*` 网络（`172.18-31.*`） |
| 坑点 | 某条 Docker Bridge 撞进 `172.21.*`，VPN / 容器互 ping 全挂 |

**解决思路**

1. 手动建统一网 `dnet1`，让所有 Compose `default ➜ dnet1`  
2. 或者从根上改 `/etc/docker/daemon.json`，限定 Docker 只能用 `192.168.240.0/20`  
3. Prometheus 里统统写 **容器名：端口**，别再 `127.0.0.1` 自己打自己  
4. 能不写 `container_name` 就不写，跨项目重名会炸

下面按模块展开。

---
## 1．Nginx 负载监控  

本节目标：在 **Docker-Compose** 环境中构建一套「带 VTS 模块」的 Nginx 镜像，并通过 `nginx-vts-exporter` 把 QPS、延迟分位数、状态码分布等指标推送到 Prometheus。下面给出 **完整、可直接粘贴的** `compose`、`Dockerfile` 与 `nginx.conf`，同时解释每一行到底干了什么。  

---

### 1.1 直接使用自编译 VTS 模块的 Nginx

#### 1.1.1 docker-compose.yml 片段

```yaml
services:
  # ──────────────────────────────────────────────
  # 带 VTS 模块的 Nginx
  # ──────────────────────────────────────────────
  nginx:
    build:
      context: ./nginx-vts        # 存放 Dockerfile、nginx.conf、证书等
      dockerfile: Dockerfile
    container_name: nginx-vts     # 仅此项目内唯一即可
    volumes:
      - ./nginx-vts/base.crt:/etc/nginx/ssl/base.crt:ro
      - ./nginx-vts/base.key:/etc/nginx/ssl/base.key:ro
      - ./nginx-vts/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx-vts/proxy_common.conf:/etc/nginx/conf.d/proxy_common.conf:ro
      - ./nginx-vts/.htpasswd:/etc/nginx/.htpasswd:ro
    restart: unless-stopped
    ports:
      - "443:443"

  # ──────────────────────────────────────────────
  # VTS 指标导出器
  # ──────────────────────────────────────────────
  nginx-vts-exporter:
    image: sophos/nginx-vts-exporter:latest
    container_name: nginx-vts-exporter
    environment:
      # 1) Basic-Auth：admin/password
      # 2) 访问路径：/status/format/json   ← nginx.conf 中固定
      # 3) HTTPS：因为上面 Nginx 监听 443
      - NGINX_STATUS=https://admin:password@nginx:443/status/format/json
      # VTS 导出器自带的 TLS 校验证书开关
      - TLS_SKIP_VERIFY=true
    ports:
      - "9913:9913"                # Prometheus 即抓取这个端口
    depends_on:
      - nginx
```

要点  
1. `nginx` 服务使用 **自定义镜像**（见下文 Dockerfile），镜像中已经把 VTS 编译进来。  
2. 证书、`nginx.conf`、通用反向代理配置和 `.htpasswd` 均以只读方式挂载。  
3. `nginx-vts-exporter` 通过环境变量 `NGINX_STATUS` 指向 **Nginx 容器内部 DNS 名 nginx**，省去写 IP。  
4. 监听 9913 端口，Prometheus 里就写 `nginx-vts-exporter:9913`。  

---

#### 1.1.2 Dockerfile（双阶段编译）

> 文件放在 `./nginx-vts/Dockerfile`

```Dockerfile
#####################################################################
# 第一阶段：在官方 nginx 镜像里把 vts 模块编译进 deb 包
#####################################################################
FROM nginx:latest as buildenv

ARG NGINX_VTS_VERSION="0.2.3"

# 1. 安装必要工具
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        gnupg2 lsb-release dpkg-dev curl

# 2. 拉取官方源公钥并写入 keyring
RUN curl -fsSL https://nginx.org/keys/nginx_signing.key \
    | gpg --dearmor -o /usr/share/keyrings/nginx-archive-keyring.gpg

# 3. 添加 deb-src 仓库（为了拿到 nginx 源码包）
RUN echo "deb-src [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
     http://nginx.org/packages/debian $(lsb_release -cs) nginx" \
     > /etc/apt/sources.list.d/nginx.list

# 4. 更新索引并拉源码
RUN apt-get update && \
    apt-get source nginx

# 5. 解析当前 nginx 版本号，装好 build-dep
RUN export NGINX_VERSION=$(dpkg-parsechangelog --show-field Version \
        < ./nginx*/debian/changelog | head -n1 | cut -d'-' -f1) && \
    apt-get build-dep -y nginx

# 6. 下载 vts 模块源码
RUN curl -sL https://github.com/vozlt/nginx-module-vts/archive/\
v${NGINX_VTS_VERSION}.tar.gz \
    | tar -xz -C /

# 7. 在 debian/rules 里插入“–add-module”
RUN sed -i -r -e \
    "s|\.\/configure(.*)|\.\/configure\1 --add-module=\/nginx-module-vts-${NGINX_VTS_VERSION}|" \
    /nginx*/debian/rules

# 8. 编译成 .deb
RUN cd /nginx* && dpkg-buildpackage -b

#####################################################################
# 第二阶段：最小化运行时镜像
#####################################################################
FROM nginx:latest

# 拷贝刚才生成的 deb 安装包
COPY --from=buildenv /nginx_*.deb /tmp/

RUN apt-get update && \
    apt-get install -y /tmp/nginx_*.deb && \
    rm -rf /var/lib/apt/lists/* /tmp/nginx_*.deb

# 默认命令，前台启动
CMD ["nginx", "-g", "daemon off;"]
```

为什么用“两阶段”  
• 第一阶段装编译依赖，镜像体积 >1 GB；  
• 第二阶段只复制结果 `.deb`，最终镜像 ≈150 MB，与官方体积相当。  

---

#### 1.1.3 nginx.conf（精简只展示 VTS 相关）

> 文件放在 `./nginx-vts/nginx.conf`

```nginx
load_module modules/ngx_http_vhost_traffic_status_module.so;

events { }

http {
    # ─────────────────────────────────────────
    # 解析容器名字，禁用 IPv6 解析
    # ─────────────────────────────────────────
    resolver 127.0.0.11 ipv6=off;

    # ─────────────────────────────────────────
    # 声明一块共享内存保存 VTS 指标
    # ─────────────────────────────────────────
    vhost_traffic_status_zone;

    # ─────────────────────────────────────────
    # 业务与状态页
    # ─────────────────────────────────────────
    server {
        listen              443 ssl;
        server_name         _;

        ssl_certificate     /etc/nginx/ssl/base.crt;
        ssl_certificate_key /etc/nginx/ssl/base.key;

        # 业务 upstream /location 略……

        # -------- VTS Status Page -------------
        location /status {
            auth_basic           "vts";
            auth_basic_user_file /etc/nginx/.htpasswd;

            # 在浏览器里直接看图表
            vhost_traffic_status_display;
            vhost_traffic_status_display_format html;

            # 如果 exporter 走 JSON，需要：
            # vhost_traffic_status_display_format prometheus;
        }
    }
}
```

Tips  
1. `vhost_traffic_status_display_format html` → 手动访问 `/status` 有图形化界面；  
   导出器会强制加 `format=json`，因此无需在这里专门改 `prometheus`。  
2. `auth_basic` 保护状态页，用户名/密码与 compose 中 `NGINX_STATUS=https://admin:password@...` 对应。  
3. `resolver 127.0.0.11` 让 Nginx 在容器内部解析其它服务名。  

---

#### 1.1.4 Prometheus 抓取示例（仅供参考）

```yaml
scrape_configs:
  - job_name: nginx-vts
    static_configs:
      - targets: ['nginx-vts-exporter:9913']
```

常见指标  
```
nginx_vts_server_requests_total{code="2xx",host="api"}      # 每秒 QPS
nginx_vts_server_response_time_seconds_p99{host="api"}      # P99 延迟
nginx_vts_filter_bytes_in_total{filter="upstream",upstream="backend"}  # 流量
```

---

至此，**Nginx-VTS → Exporter → Prometheus** 整条链路就串好了：  
浏览器看 `/status` 能实时出图，Prometheus 也能收到完整指标，Grafana 再导入官方 Dashboard #2949 即可看到 QPS、5xx、延迟曲线。

---

## 2．ComfyUI 存活监控 & 重启计数  
本节把「ComfyUI HTTP 探活 + 自动发现目标 + 告警」完整打通，而且**不用传统 cron**。黑盒探针仍采用官方 `prom/blackbox-exporter`，目标文件由一个极简 **sidecar** 守护脚本循环生成，全部交给 Docker-Compose 管理。

---

### 2.1 Blackbox Exporter 服务

`docker-compose.yml` 片段：  

```yaml
###############################################################################
# Blackbox Exporter：探测 ComfyUI 是否在线（含 BASIC-AUTH、正则校验）
###############################################################################
blackbox-exporter:
  image: prom/blackbox-exporter:latest
  container_name: blackbox-exporter
  volumes:
    - ./blackbox/blackbox.yml:/config/blackbox.yml:ro      # 模块定义
  secrets:
    - comfy_pass                                            # BASIC-AUTH 密码
  command: --config.file=/config/blackbox.yml
  ports:
    - "9115:9115"
  restart: unless-stopped
```

1. `blackbox.yml` 里只保留自定义模块 `comfy_https`（见下一小节）。  
2. 密码通过 Docker Secret 注入：`/run/secrets/comfy_pass`。  
3. 监听 9115，Prometheus 抓这个端口即可。

---

### 2.2 Blackbox 配置 `blackbox.yml`

> 文件路径：`./blackbox/blackbox.yml`

```yaml
modules:
  comfy_https:
    prober: http
    timeout: 15s

    http:
      method: GET
      basic_auth:
        username: admin
        password_file: /run/secrets/comfy_pass   # ← 从 secret 读
      valid_status_codes: [200]                  # 允许 200
      follow_redirects: true
      tls_config:
        insecure_skip_verify: true               # 跳过自签证书
      fail_if_body_not_matches_regexp:
        - "CheckpointLoaderSimple"               # 返回体必须包含此关键字
```

---

### 2.3 动态目标：File-SD，但不用 cron

思路：起一个极简 sidecar **每 60 秒写一次**目标文件；Prometheus 自带的 `file_sd_configs` 负责热加载，无需 cron/系统定时器。

#### 2.3.1 共享目标目录

先在 Compose 顶部声明一个卷，用于 **Prometheus 与 sidecar 共享**：

```yaml
volumes:
  comfy_targets:
```

#### 2.3.2 Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: comfy_status
    metrics_path: /probe
    params:
      module: [comfy_https]                 # 指定上面定义的模块
    file_sd_configs:
      - files:
          - /etc/prometheus/targets_comfy/*.json
        refresh_interval: 1m
    relabel_configs:
      # 把文件里的 target 赋给 __param_target
      - source_labels: [__address__]
        target_label: __param_target
      # 展示用 instance/target 标签
      - source_labels: [__address__]
        target_label: instance
      - source_labels: [__address__]
        target_label: target
      # 真正去抓 blackbox-exporter:9115
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

挂卷别忘了：

```yaml
prometheus:
  image: prom/prometheus:latest
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    - comfy_targets:/etc/prometheus/targets_comfy          # 同一卷
    - prometheus_data:/prometheus
  ports:
    - "9090:9090"
```

---

### 2.4 告警规则（Alertmanager）

TODO

```yaml
groups:
- name: comfy
  rules:
    - alert: ComfyUI_Down
      expr: probe_success{job="comfy"} == 0
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "ComfyUI 实例不可用"
        description: "实例 {{ $labels.instance }} 已连续 2 分钟探测失败"
```

---

至此，「ComfyUI 探活 + 自动目标发现 + 掘出来的指标和告警」全部打通，而且完全**依赖 Docker-Compose 自带机制**，不占用宿主的系统级 cron 服务。

---

## 3．GPU 指标采集（DCGM-Exporter 方案）

目标：把显卡温度、功耗、显存占用、核心利用率等 DCGM 指标全部送进 Prometheus，随后在 Grafana 导入官方 Dashboard `DCGM Exporter / NVIDIA Data Center GPU Manager (ID: 12239)` 即可直接出图。

---
### 3.1 服务定义：`gpu-exporter`

```yaml
###############################################################################
# GPU Exporter —— NVIDIA 官方 DCGM Exporter
###############################################################################
services:
  gpu-exporter:
    image: nvidia/dcgm-exporter:4.2.3-4.1.3-ubuntu22.04  # 固定版本便于排障
    container_name: gpu-exporter

    # 让容器拥有对宿主 GPU 的完全访问权
    runtime: nvidia                                      # 等同于 --gpus all
    environment:
      - NVIDIA_VISIBLE_DEVICES=all                       # 全部 GPU
      # 如需屏蔽部分卡，可写: 0,2  或 GPU-UUID

    ports:
      - "9400:9400"                                      # Prometheus 抓取口
    restart: unless-stopped
```

关键点说明  
1. `runtime: nvidia` 要求宿主机已安装 **nvidia-container-toolkit**（`nvidia-docker2`）。  
2. 镜像标签采用「DCGM 版本-Driver 版本-基础 OS」，升级驱动后请一并升级镜像。  

---

### 3.2 Prometheus 抓取配置

```yaml
scrape_configs:
  - job_name: "gpu"
    static_configs:
      - targets:
          - gpu-exporter:9400       # 与 compose 里的容器名 + 端口保持一致
```

若想在查询 & 告警里区分多机，可加一条固定标签：

```yaml
  - job_name: "gpu"
    static_configs:
      - targets: ['gpu-exporter:9400']
        labels:
          node: gpu-node-01
```

---

### 3.3 常见指标速查

| 指标 | 含义 | 典型阈值 |
| ---- | ---- | -------- |
| `dcgm_gpu_utilization` | 核心利用率 (%) | > 90% 持续 10 min 预警 |
| `dcgm_memory_used` / `dcgm_memory_total` | 显存占用 / 总量 (MiB) | 剩余 < 500 MiB 警报 |
| `dcgm_power_usage` | 实时功耗 (W) | 逼近 `dcgm_power_limit` |
| `dcgm_temperature` | GPU 核温 (℃) | > 80 ℃ 告警 |

---

### 3.4 Grafana 快速出图

1. 在 Grafana → Dashboards → Import  
2. 输入 `12239`（官方 DCGM Dashboard ID）  
3. 选择数据源 `Prometheus` → Import 完成  

即可得到温度 / 利用率 / 显存 / 功耗多维度概览。

---

至此，GPU 监控链路 `DCGM Exporter → Prometheus → Grafana` 配置完毕，若后续新增 GPU 主机，只需把同样的 `gpu-exporter` 服务抄过去再加个 scrape target 即可复用。

---

## 4. Docker & VPN 网段冲突处理

### 4.1 一劳永逸：限制 Docker 地址池

`/etc/docker/daemon.json`

```json
{
  "default-address-pools": [
    { "base": "192.168.240.0/20", "size": 24 }
  ]
}
```

`systemctl restart docker`

### 4.2 快速兜底：统一外部网络 dnet1

```bash
docker network create \
  --subnet 192.168.250.0/24 \
  --gateway 192.168.250.1 \
  dnet1
```

所有 Compose 结尾追加：

```yaml
networks:
  default:
    external: true
    name: dnet1
```

---

## 5. 最终 docker-compose.monitor.yml（完整版）

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/comfy_service_url.json:/etc/prometheus/targets_comfy.json:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"
  gpu-exporter:
    image: nvidia/dcgm-exporter:4.2.3-4.1.3-ubuntu22.04
    container_name: gpu-exporter
    runtime: nvidia # 参考 https://developer.nvidia.com/container-runtime
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    restart: unless-stopped
    ports:
      - "9400:9400"
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    network_mode: host
    restart: unless-stopped
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox-exporter
    volumes:
      - ./blackbox/blackbox.yml:/config/blackbox.yml:ro
    command: --config.file=/config/blackbox.yml
    secrets:
      - comfy_pass
    ports:
      - "9115:9115"
    restart: unless-stopped
  nginx:
    build:
      context: ./nginx-vts
      dockerfile: Dockerfile
    container_name: nginx-vts
    volumes:
      - ./nginx-vts/base.crt:/etc/nginx/ssl/base.crt
      - ./nginx-vts/base.key:/etc/nginx/ssl/base.key
      - ./nginx-vts/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx-vts/proxy_common.conf:/etc/nginx/conf.d/proxy_common.conf
      - ./nginx-vts/.htpasswd:/etc/nginx/.htpasswd
    restart: unless-stopped
    ports:
      - "443:443"
  nginx-vts-exporter:
    image: sophos/nginx-vts-exporter
    container_name: nginx-vts-exporter
    environment:
      - NGINX_STATUS=https://admin:wv7edp3cx4@nginx:443/status/format/json
      - TLS_SKIP_VERIFY=true
    ports:
      - "9913:9913"
    depends_on:
      - nginx
  comfyd-linux:
    build:
      context: ./self-service
      dockerfile: Dockerfile
    container_name: comfyd-driver
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /data/recast-multi-1/docker-compose.yml:/data/recast-multi-1/docker-compose.yml
      - /data/flux-20002/docker-compose.yml:/data/flux-20002/docker-compose.yml
      - /data/flux-20003/docker-compose.yml:/data/flux-20003/docker-compose.yml
      - /data/hidream-20000/docker-compose.yml:/data/hidream-20000/docker-compose.yml
      - /data/hidream-20001/docker-compose.yml:/data/hidream-20001/docker-compose.yml
    restart: unless-stopped
    ports:
      - "4001:4001"
secrets:
  comfy_pass:
    file: ./blackbox/comfy_pass
volumes:
  prometheus_data:
networks:
  default:
    external: true
    name: dnet1
```

---

## 6. 小结

1. **Nginx** 用 VTS 模块 / exporter 拿到完整维度  
2. **ComfyUI** 用 Blackbox + file SD，2 分钟探活告警  
3. **GPU** 官方 dcgm-exporter，Grafana 官方 dashboard #12239 即可  
4. **Docker+VPN** 要么改 AddressPool，要么所有 Compose 进同一个 bridge  
5. **Prometheus** 模板化 relabel，YAML 少写一半

> 至此，从系统资源 → GPU → 入口 Nginx → 应用存活，全链路监控闭环完成。  
> Happy Hacking 🎉