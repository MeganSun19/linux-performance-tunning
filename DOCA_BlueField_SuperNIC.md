# DOCA / BlueField / SuperNIC 完整笔记

> NCP-AIN 考纲：**Domain 2**（30% 权重）BlueField/DOCA + **Domain 5**（20% 权重）SuperNIC 排障
> 学时：3h（概念架构 1h + Profile/安装 1h + 三方协作 1h）

---

## 第一部分：基础概念

### 1.1 为什么会有 DOCA？

```
═══════════════════════════════════════════════════════════════
  痛点: 100Gbps 服务器, CPU 70% 用在"管道"上
═══════════════════════════════════════════════════════════════

  传统服务器 CPU 占用分布:
    业务应用              30%   ← 用户付钱的
    网络协议栈 (TCP/IP)   30%
    存储协议 (NVMe-oF)    15%
    虚拟化 (vSwitch)      10%
    安全/加解密           10%
    监控                  5%
                          ────
    基础设施合计          70%   ← 浪费

  方案: 把基础设施 offload 到独立的卡 (DPU)
        主机 CPU 100% 给业务

  NVIDIA 2020 收购 Mellanox → 2022 推出 DOCA
  
  类比记忆:
    GPU 编程 → CUDA
    DPU 编程 → DOCA      ← 完全对应的关系
```

**DOCA 全称**：Data Center Infrastructure-on-a-Chip Architecture（数据中心基础设施芯片架构）

**一句话定义**：NVIDIA 给 BlueField DPU / SuperNIC 写的 SDK 框架。

---

### 1.2 硬件家族演进（必背）

```
═══════════════════════════════════════════════════════════════
  NIC → SmartNIC → DPU → SuperNIC
═══════════════════════════════════════════════════════════════

  ① 普通 NIC
     功能: 仅收发包
     CPU 介入: 100% (协议栈全靠主机 CPU)
     例: Intel X710

  ② SmartNIC (智能网卡)
     功能: 收发包 + 部分硬件 offload (TSO/checksum/RDMA)
     CPU 介入: ~70%
     特点: 有加速器, 但没有独立 CPU
     例: Mellanox ConnectX-5/6

  ③ DPU (Data Processing Unit) ★ 关键
     功能: SmartNIC + 独立 ARM CPU + 独立内存 + 独立 OS
     CPU 介入: 接近 0% (整个 OS 可跑在 DPU)
     特点: 一张独立的小型服务器
     例: BlueField-2/3

  ④ SuperNIC (NCP-AIN 必考新概念)
     定义: AI 优化的 BlueField-3 产品形态
     不是新硬件, 是同一硬件的不同固件
     特点:
       - 强化 RoCE / Adaptive Routing
       - 与 Spectrum-X 协同设计
       - 弱化通用 offload (不跑虚拟化)
       - 价格比完整 DPU 便宜
     例: BlueField-3 SuperNIC, ConnectX-8 SuperNIC
```

### 1.3 BlueField 型号速查表（必背）

| 型号 | 速率 | ARM 核 | 用途 |
|------|------|--------|------|
| BlueField-2 | 200G | 8 (A72) | 早期通用云 |
| BlueField-3 | 400G | 16 (A78) | 通用云 / AI |
| BlueField-3 SuperNIC | 400G | 16 | AI 专用 (RoCE) |
| ConnectX-8 SuperNIC | 800G | 无独立 OS | AI 极致 |

**★ 关键区分**：
- **DPU** = 跑独立 OS，有 ARM CPU
- **SuperNIC** = AI 优化形态，BF-3 SuperNIC 跑 OS，CX-8 SuperNIC 不跑 OS

### 1.4 选型决策

```
┌──────────────────────────────────────────────────────────────┐
│ 场景                         选型                             │
├──────────────────────────────────────────────────────────────┤
│ AI 训练集群 (单租户)         SuperNIC (BF-3 或 CX-8)         │
│ AI 推理 / 多租户             BlueField-3 DPU                  │
│ 云服务器 (虚拟化 + 安全)     BlueField-3 DPU                  │
│ 安全网关                     BlueField-3 DPU                  │
└──────────────────────────────────────────────────────────────┘
```

---

### 1.5 AI 网络虚拟化的真实场景（关键认知）

**直觉误区**：以为 AI 网络都追求极致性能，不用虚拟化 → ❌

**真实情况**：4 种场景，3 种用虚拟化。

```
═══════════════════════════════════════════════════════════════
  场景 A: 单租户大模型训练       ← 不用虚拟化
═══════════════════════════════════════════════════════════════
  环境: 几千张 H100 跑一个 LLM 训练
  网络: 纯 RoCE, SuperNIC 直通
  例: Meta 训练 Llama, xAI 训练 Grok

═══════════════════════════════════════════════════════════════
  场景 B: AI 云 / GPUaaS         ← 必须虚拟化 ★
═══════════════════════════════════════════════════════════════
  环境: AWS/GCP/阿里云 GPU 实例服务
  问题: 多租户共享物理集群, 用户 A/B 必须隔离
    - 不能互看流量
    - VPC 隔离
    - 按用户计费
    - K8s 集群独立
  解决: VXLAN/Geneve overlay + ACL/防火墙 + 多租户
        → OVS 必需 → DOCA Flow 把 OVS offload 到 SuperNIC
  例: AWS P5 实例用 Nitro DPU

═══════════════════════════════════════════════════════════════
  场景 C: AI 推理混部             ← 容器虚拟化
═══════════════════════════════════════════════════════════════
  环境: 一个集群跑多模型推理 (GPT-4 + Llama + 嵌入模型)
  网络: K8s CNI (Calico/Cilium) + RoCE
  问题: K8s 网络默认走 CPU, 100Gbps 流量 = CPU 炸
  解决: DOCA Flow 加速 Calico/Cilium 数据面

═══════════════════════════════════════════════════════════════
  场景 D: 训练 + 推理混部         ← 策略灵活性
═══════════════════════════════════════════════════════════════
  环境: 白天推理, 晚上训练
  解决: DOCA + OVS 动态切换网络策略
```

**核心结论**：

> **DOCA 的价值 = "让虚拟化和性能不再对立"**
>
> 极致性能训练 → SuperNIC 直通
> 多租户云 → DPU + DOCA Flow offload → 既保留虚拟化，又保持性能

---

## 第二部分：DOCA 架构

### 2.1 DOCA 4 层架构（ASLD 助记）

```
═══════════════════════════════════════════════════════════════
  从上到下: Apps / Services / Libraries / Driver
═══════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────┐
  │ Layer 4: Apps (参考应用)                                  │
  │  - DPI 防火墙 / IPsec VPN / 加密代理 / DNS 过滤           │
  │  - 类似 CUDA samples, 拿来改                              │
  └─────────────────────────────────────────────────────────┘
                          ▲
  ┌─────────────────────────────────────────────────────────┐
  │ Layer 3: Services (开箱即用服务)                          │
  │  - doca-telemetry (遥测)                                  │
  │  - doca-firefly   (PTP 时钟同步)                          │
  │  - doca-blueman   (DPU 管理 UI)                           │
  │  - doca-hbn       (Host-Based Networking, BGP/EVPN)       │
  └─────────────────────────────────────────────────────────┘
                          ▲
  ┌─────────────────────────────────────────────────────────┐
  │ Layer 2: Libraries (核心 API, ★ 重点)                    │
  │  网络: DOCA Flow / Comm Channel / RDMA                    │
  │  安全: DOCA Crypto / Regex / IPsec                        │
  │  存储: DOCA Compress / NVMe-oF                            │
  │  GPU:  DOCA GPUNetIO ★ AI 必考                            │
  └─────────────────────────────────────────────────────────┘
                          ▲
  ┌─────────────────────────────────────────────────────────┐
  │ Layer 1: Driver / Runtime                                 │
  │  - DOCA Core (内存 / 设备发现)                            │
  │  - DPDK / SPDK (开源底层)                                 │
  │  - MLNX_OFED (网卡驱动) ★ 永远基础                       │
  └─────────────────────────────────────────────────────────┘
                          ▲
  ┌─────────────────────────────────────────────────────────┐
  │ Layer 0: BlueField 硬件                                   │
  └─────────────────────────────────────────────────────────┘
```

**记忆口诀**：**ASLD**（从上到下：Apps → Services → Libraries → Driver）

### 2.2 4 个必考的 DOCA Library

```
═══════════════════════════════════════════════════════════════
  ① DOCA Flow (云网络必考)
═══════════════════════════════════════════════════════════════
  作用: 在 SuperNIC ASIC 上编程硬件流表 (类似 OpenFlow)
  替代: 传统 OVS (Open vSwitch) 软件实现
  
  传统 OVS 痛点:
    100Gbps 流量, OVS 吃掉 CPU 8-16 核
  
  DOCA Flow 方案:
    flow table 装到 ConnectX ASIC
    CPU 占用 0%, 线速 100/200/400 Gbps
  
  典型用法 (match-action):
    match:  src_ip=10.0.0.1, dst_ip=10.0.0.2
    action: encap_vxlan(vni=100), forward(port_1)
  
  AI 场景: 多租户 AI 云隔离, K8s 网络加速


═══════════════════════════════════════════════════════════════
  ② DOCA GPUNetIO (AI 必考)
═══════════════════════════════════════════════════════════════
  作用: GPU 直接收发网络包, 完全绕过 CPU
  
  传统数据路径:
    NIC → CPU 内存 → CPU 处理 → GPU 内存 → GPU
              ★ CPU 中转, 延迟 + 占用 CPU
  
  GPUNetIO 路径:
    NIC → GPU 内存 → GPU
              ★ CPU 完全旁路
              ★ GPU CUDA kernel 直接调用网络 API
  
  应用场景:
    - 5G/AI-RAN (极致低延迟)
    - 实时推理
    - GPU 跑网络协议栈


═══════════════════════════════════════════════════════════════
  ③ DOCA Comm Channel
═══════════════════════════════════════════════════════════════
  作用: 主机 ↔ DPU 控制平面通信
  
  问题: DPU Mode 下, 主机和 DPU 是两个独立 OS
        怎么通信? (不是网络数据, 是控制信息)
  
  方案: 走 PCIe 专用通道, 不经过网络
        低延迟, 加密
  
  例: 主机让 DPU 加载新防火墙规则


═══════════════════════════════════════════════════════════════
  ④ DOCA Telemetry
═══════════════════════════════════════════════════════════════
  作用: DPU 遥测数据采集 + 上报
  
  采集:
    - ASIC 计数器
    - 流量统计
    - DPU CPU/内存使用率
  
  导出: Prometheus / InfluxDB / Fluentd
  
  与交换机 telemetry 区别:
    交换机 telemetry → 看 ASIC 转发
    DOCA Telemetry  → 看 DPU 上应用
    两者协同 → 端到端可观测
```

---

### 2.3 DOCA 两种部署模式（★★ 必考）

```
═══════════════════════════════════════════════════════════════
  Host Mode vs DPU Mode
═══════════════════════════════════════════════════════════════

  Host Mode (主机模式):
  ────────────────────
  SDK 装在主机 x86
  应用跑在主机 CPU
  DPU 只做"硬件加速器", 不跑独立 OS
  
  ┌────────────────────────────────┐
  │ Host x86 Linux                  │
  │  应用 + DOCA Libraries          │  ← SDK 在主机
  └─────────────┬──────────────────┘
                │ PCIe
  ┌─────────────▼──────────────────┐
  │ BlueField DPU (作为加速卡)       │
  └─────────────────────────────────┘
  
  适用: 改造旧应用, 不想换架构
  优势: 改动小, 兼容性好
  劣势: 主机 CPU 仍要参与


  DPU Mode (★ NVIDIA 推荐):
  ────────────────────────
  SDK 装在 DPU 的 ARM Linux 里
  应用跑在 DPU ARM CPU
  主机 CPU 完全不参与基础设施
  
  ┌────────────────────────────────┐
  │ Host x86 (跑业务应用)            │
  │  100% CPU 给业务                 │
  └─────────────┬──────────────────┘
                │ PCIe (只看到一个虚拟网卡)
  ┌─────────────▼──────────────────┐
  │ BlueField DPU (独立 Ubuntu/RHEL) │
  │  DOCA + 应用 (firewall/OVS)     │
  └─────────────────────────────────┘
  
  适用: 零信任安全 / 多租户 / 网络虚拟化
  优势:
    - 主机 100% 给业务
    - 安全隔离 (主机被攻破不影响 DPU)
    - DPU 独立升级
  劣势: 需要重新设计架构


═══════════════════════════════════════════════════════════════
  对比表 (考试用)
═══════════════════════════════════════════════════════════════

                  Host Mode          DPU Mode
                  ────────           ────────
  SDK 位置        主机 x86            DPU ARM
  应用跑在        主机                DPU
  主机 CPU 占用   高                  低
  适用场景        改造旧应用          云 / 零信任
  开发难度        低                  中 (交叉编译)
  安全隔离        弱                  强
  推荐?           ★ DPU Mode
```

---

## 第三部分：DOCA Profile（重点）

### 3.1 什么是 DOCA Profile

```
═══════════════════════════════════════════════════════════════
  Profile 概念
═══════════════════════════════════════════════════════════════

  类比记忆:
    Cumulus Linux:  forwarding profile (切 ASIC 资源)
    DOCA:           DOCA profile      (切 DPU 软件包)
  
  本质: 不同业务需要不同 DOCA 组件组合
        全装太占空间 (DPU eMMC 仅 32-64GB)
        按场景选 profile, 只装对应包
```

### 3.2 4 种官方 Profile（必背）

| Profile | 包含组件 | 适用 |
|---------|---------|------|
| **doca-networking** | DOCA Flow / Comm Channel / HBN / OVS | 云虚拟化 |
| **doca-roce** | RoCE 库 / GPUNetIO / AR 调优 | **AI 集群 ★** |
| **doca-security** | IPsec / TLS / Regex / DPI | 零信任安全 |
| **doca-all** | 全部组件 | 开发/测试 |

**AI 集群选型**：
- GPU 服务器 + SuperNIC → `doca-roce`
- 云服务器 + DPU → `doca-networking`
- 安全网关 + DPU → `doca-security`

### 3.3 Profile 切换步骤

```bash
═══════════════════════════════════════════════════════════════
  方式 A: 主机侧 (Host Mode)
═══════════════════════════════════════════════════════════════

# 1. 查当前 profile
doca_info --profile

# 2. 卸载旧 profile
sudo apt remove --purge 'doca-*'
sudo apt autoremove

# 3. 装新 profile
sudo apt install -y doca-roce

# 4. 验证
doca_info --profile
# 期望: Profile: doca-roce


═══════════════════════════════════════════════════════════════
  方式 B: DPU 侧 (DPU Mode)
═══════════════════════════════════════════════════════════════

# 1. SSH 到 DPU
ssh ubuntu@192.168.100.2  # 默认 rshim IP

# 2. 同样 apt 操作 (DPU 是 ARM 架构)
sudo apt remove --purge 'doca-*'
sudo apt install -y doca-roce

# 3. 重启服务
sudo systemctl restart doca-*


═══════════════════════════════════════════════════════════════
  方式 C: BFB 整包重刷 (彻底切换)
═══════════════════════════════════════════════════════════════

# 不同 profile 有对应 BFB 镜像
sudo bfb-install \
  --bfb DOCA_2.5.0_BSP_4.5.0_Ubuntu_22.04-roce.bfb \
  --rshim rshim0

# ⚠️ 整个 DPU OS 重写, 数据全清, 慎用
```

### 3.4 Profile 切换 3 个陷阱（考点）

```
┌──────────────────────────────────────────────────────────────┐
│ 陷阱 1: 不能多个 profile 共存                                 │
│         doca-roce 和 doca-networking 互斥                     │
│         强装会冲突 → 必须先卸载                                │
│                                                                │
│ 陷阱 2: 切 profile 后必须重启服务                             │
│         systemctl restart doca-*                              │
│         否则旧库还在内存                                       │
│                                                                │
│ 陷阱 3: BFB 重刷会丢配置                                      │
│         /etc/doca/ 下的所有自定义配置                          │
│         必须先备份: tar czf doca-backup.tar.gz /etc/doca/     │
└──────────────────────────────────────────────────────────────┘
```

---

## 第四部分：DOCA 安装步骤（重点）

### 4.1 4 个安装阶段总览

```
Phase 1: 准备       (确认硬件 / OS / rshim)
Phase 2: 固件升级   (★ 必须和 DOCA 版本配套)
Phase 3: 安装 DOCA  (3 种方式)
Phase 4: 验证       (RoCE 端到端测试)
```

### 4.2 Phase 1：准备

```bash
# 1. 确认硬件
lspci | grep -i mellanox
# 期望: 看到 BlueField-3 / ConnectX-7

# 2. 确认 OS 兼容
cat /etc/os-release
# 支持: Ubuntu 22.04, RHEL 8.x/9.x, SLES 15

# 3. 确认 rshim (DPU Mode 必需)
sudo systemctl status rshim
ls /dev/rshim*

# 4. 查 BlueField 当前状态
sudo mlxconfig -d /dev/mst/mt41686_pciconf0 query
sudo mlxfwmanager --query
```

### 4.3 Phase 2：固件升级（关键易错）

```bash
# 1. 装 MFT (Mellanox Firmware Tools)
sudo apt install -y mft
sudo mst start

# 2. 查当前固件版本
sudo flint -d /dev/mst/mt41686_pciconf0 query

# 3. 升级固件 (必须和 DOCA 版本配套!)
sudo flint -d /dev/mst/mt41686_pciconf0 \
  -i fw-BlueField-3-rel-32_39_2048.bin burn

# 4. 重启 DPU
sudo mlxfwreset -d /dev/mst/mt41686_pciconf0 -y reset
```

**★ 版本配套表（必查官方 release notes）**：

| DOCA | BF FW | OFED |
|------|-------|------|
| 2.5 | 32.39.x | 23.10 |
| 2.6 | 32.40.x | 24.01 |

不配套 = DOCA 装完也用不了，RoCE 不工作。

### 4.4 Phase 3：装 DOCA（3 种方式）

```bash
═══════════════════════════════════════════════════════════════
  方式 A: Host Package (推荐, 90% 场景)
═══════════════════════════════════════════════════════════════

# 1. 加仓库
wget https://.../doca-host-repo-ubuntu2204_2.5.0-xxx_amd64.deb
sudo dpkg -i doca-host-repo-ubuntu2204_*.deb
sudo apt update

# 2. 装目标 profile (自动拉 OFED 依赖)
sudo apt install -y doca-roce


═══════════════════════════════════════════════════════════════
  方式 B: BFB 刷 DPU (DPU Mode 必需)
═══════════════════════════════════════════════════════════════

# 1. 下载 BFB
wget https://.../DOCA_2.5.0_BSP_4.5.0_Ubuntu_22.04-roce.bfb

# 2. 刷写 (5-10 min)
sudo bfb-install \
  --bfb DOCA_2.5.0_BSP_4.5.0_Ubuntu_22.04-roce.bfb \
  --rshim rshim0

# 3. SSH 到 DPU
ssh ubuntu@192.168.100.2  # 默认密码 ubuntu


═══════════════════════════════════════════════════════════════
  方式 C: Container (CI/CD 友好)
═══════════════════════════════════════════════════════════════

docker pull nvcr.io/nvidia/doca/doca:2.5.0-devel
docker run --privileged --net host -v /dev:/dev \
  nvcr.io/nvidia/doca/doca:2.5.0-devel
```

### 4.5 Phase 4：验证

```bash
# 1. DOCA 版本
doca_info
# 期望: DOCA 2.5.0 / Profile: doca-roce / BF FW: 32.39.2048

# 2. OFED 版本
ofed_info -s

# 3. 设备识别
ibv_devices
# 期望: mlx5_0, mlx5_1 等

# 4. RoCE 状态
ibv_devinfo -d mlx5_0
# 关注: link_layer: Ethernet (= RoCE)
#       port_state: PORT_ACTIVE

# 5. RDMA 性能测试 (端到端)
# Server: ib_write_bw -d mlx5_0 -F
# Client: ib_write_bw -d mlx5_0 -F <server_ip>
# 期望: 接近线速
```

### 4.6 常见 5 个安装故障

| # | 故障 | 排查 |
|---|------|------|
| 1 | lspci 看不到 BlueField | `dmesg \| grep mellanox`，换 PCIe 槽 |
| 2 | rshim 无法启动 | host 和 DPU 都开 rshim → 只在 host 启用 |
| 3 | bfb-install 卡住 | md5sum 验 BFB；重启 rshim |
| 4 | ibv_devices 空 | OFED/固件不配套；对照 release notes |
| 5 | RoCE 性能不达标 | PFC/ECN 未配；profile 错；`mlnx_qos -i` 查 |

---

## 第五部分：DOCA / RDMA / RoCE 历史关系（澄清）

### 5.1 关键时间线

```
═══════════════════════════════════════════════════════════════
  DOCA 不是发明 RDMA, 是把它打包得更好用
═══════════════════════════════════════════════════════════════

  1999  InfiniBand 联盟成立, RDMA 原生支持
  2003  IB Verbs API 标准化 (libibverbs)
  2010  RoCEv1 (RDMA over Ethernet, 二层)
  2013  GPUDirect RDMA (CUDA 5.0, ★ NCCL 用)
  2014  RoCEv2 (RDMA over UDP/IP, 可路由)
  2020  NVIDIA 收购 Mellanox
  2022  DOCA 1.0 / DOCA GPUNetIO
  
  ★ RDMA/RoCE 早于 DOCA 10+ 年
  ★ DOCA 是 OFED 之上的新层, 不是替代
```

### 5.2 完整栈关系（重要）

```
═══════════════════════════════════════════════════════════════
  应用 → DOCA (可选) → MLNX_OFED (必需) → 硬件
═══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────┐
  │ 应用层                                                  │
  │  HPC:       MPI + libibverbs                           │
  │  AI 训练:   NCCL + libibverbs + GPUDirect RDMA         │
  │  AI 推理:   DOCA GPUNetIO (极致低延迟)                  │
  │  云网络:    DOCA Flow (OVS offload)                    │
  └──────────────────────────────────────────────────────┘
                          ▲
  ┌──────────────────────────────────────────────────────┐
  │ DOCA Libraries (2022+, 可选层)                          │
  │  - 在 OFED 之上, 提供更高层封装                          │
  └──────────────────────────────────────────────────────┘
                          ▲
  ┌──────────────────────────────────────────────────────┐
  │ MLNX_OFED (2010+, ★ 必需基础)                          │
  │  - libibverbs / librdmacm / libmlx5                    │
  │  - mlx5 内核驱动                                        │
  │  - perftest / ibstat / mlnx_qos                        │
  │  - 任何 RDMA/RoCE 应用都依赖这个                        │
  └──────────────────────────────────────────────────────┘
                          ▲
  ┌──────────────────────────────────────────────────────┐
  │ 硬件: ConnectX-3/4/5/6/7/8 / BlueField-2/3              │
  └──────────────────────────────────────────────────────┘

  ★ DOCA 出现前: 应用 → OFED → 硬件
  ★ DOCA 出现后: 应用 → DOCA → OFED → 硬件
                 (DOCA 可选, OFED 仍必需)
```

### 5.3 CPU Bypass 的 3 个层次（必懂）

```
═══════════════════════════════════════════════════════════════
  从普通 RDMA 到 GPUNetIO 的演进
═══════════════════════════════════════════════════════════════

  层次 1: 普通 RDMA (1999)
  ──────────────────────
  实现: libibverbs
  路径:
    Host CPU 通过 verbs 发起 RDMA 请求
    NIC 直接读写远端内存
    Host CPU 不参与数据传输 (但要发起)
  特点: 控制面 CPU 介入, 数据面 CPU bypass
  例: 任何用 ib_write_bw 的测试都是这个层次


  层次 2: GPUDirect RDMA (2013, ★ NCCL 用)
  ───────────────────────────────────────
  实现: nv_peer_mem 内核模块 + libibverbs
  路径:
    GPU memory → NIC → 网络 → 远端 NIC → 远端 GPU memory
    完全不经过 CPU 内存
  特点: GPU 和 NIC 直接 DMA
  限制:
    - GPU 和 NIC 必须在同一 PCIe Root Complex
    - 控制面仍 CPU 发起
  
  ★ 这才是你之前学 NCCL 时用的技术
  ★ 早于 DOCA 整 10 年


  层次 3: GPUNetIO (2022, DOCA 时代)
  ────────────────────────────────
  实现: DOCA GPUNetIO 库
  路径:
    GPU 自己发起网络请求 (CPU 完全不参与)
    GPU CUDA kernel 直接调用网络 API
  特点:
    - 数据面 bypass CPU
    - 控制面也 bypass CPU
  适用: 极致低延迟 (5G/AI-RAN, 实时推理)


═══════════════════════════════════════════════════════════════
  对照表
═══════════════════════════════════════════════════════════════

  技术              年份    控制面    数据面    GPU 直通
  ────              ────    ──────    ──────    ────────
  libibverbs        1999    CPU       bypass    ❌
  GPUDirect RDMA    2013    CPU       bypass    ✓ (CPU 触发)
  DOCA GPUNetIO     2022    bypass    bypass    ✓ (GPU 自主)

  ★ 你之前学的 RoCE/PFC/ECN 实验:
    用的是 libibverbs + MLNX_OFED (perftest 工具集)
    不依赖 DOCA, 是 DOCA 之前就有的传统技术栈
```

---

## 第六部分：Spectrum-X 三方协作（终极考点）

### 6.1 为什么必须三方协同

```
┌──────────────────────────────────────────────────────────────┐
│ Spectrum-X 是端到端方案, 缺一不可:                            │
│                                                                │
│ ① 只有 Spectrum 交换机, 没有 SuperNIC:                       │
│    AR 喷洒的乱序包 → 传统 NIC 当错误丢                       │
│    → 退化为普通 ECMP                                           │
│                                                                │
│ ② 只有 SuperNIC, 没有 Spectrum 交换机:                       │
│    SuperNIC 喷洒, 交换机仍按 flow hash                        │
│    → 喷洒功能浪费                                              │
│                                                                │
│ ③ 只有硬件, 没有 DOCA:                                       │
│    应用层无法调用 SuperNIC 高级特性                            │
│    → 退化为普通 RoCE                                           │
│                                                                │
│ ★ 三者同时部署, 才是真正的 Spectrum-X                        │
└──────────────────────────────────────────────────────────────┘
```

### 6.2 完整数据路径（端到端 RoCE 加速）

```
═══════════════════════════════════════════════════════════════
  GPU-A 发送 NCCL all-reduce 到 GPU-B 的完整流程
═══════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────┐
  │ GPU-A (sender) 应用: NCCL all-reduce                   │
  └─────────────────────┬────────────────────────────────┘
                        │ (1) GPU 直接 DMA 到 SuperNIC
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ SuperNIC A (BlueField-3 / ConnectX-8)                  │
  │  - DOCA GPUNetIO: GPU 直接收发 API                     │
  │  - RoCEv2 封装: 加 UDP/IP/IB BTH 头                    │
  │  - 拥塞控制: DCQCN 响应 ECN                            │
  │  - Packet Spraying: 包喷洒到多 spine ★                │
  │  - AR 协同: 标记 flow 信息                              │
  └─────────────────────┬────────────────────────────────┘
                        │ (2) 多路径并行
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ Leaf 交换机 (Spectrum-4)                                │
  │  - Adaptive Routing: 看队列深度选最优 spine            │
  │  - per-packet 喷洒 (不再 5-tuple hash)                 │
  │  - ECN 标记: 队列超阈值打 CE 标志                       │
  │  - PFC: 队列满反压上游                                  │
  └─────────────────────┬────────────────────────────────┘
                        │ (3) 经过多 spine 并行
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ Spine 交换机 × 8 (Spectrum-4)                          │
  │  - 同样 AR + ECN 标记                                   │
  │  - 包乱序到达 (多路径必然)                              │
  └─────────────────────┬────────────────────────────────┘
                        │ (4)
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ Leaf 交换机 (对端) → SuperNIC B                         │
  └─────────────────────┬────────────────────────────────┘
                        │ (5)
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ SuperNIC B                                              │
  │  - 包重排序 (Packet Reordering) ★ 关键                 │
  │    传统 NIC: 乱序 = 错误 = 重传                         │
  │    SuperNIC: 硬件重排, 应用看到顺序流                    │
  │  - 见 ECN 标记 → 发 CNP 给 SuperNIC A                  │
  │  - DMA 到 GPU-B 内存                                    │
  └─────────────────────┬────────────────────────────────┘
                        │ (6)
                        ▼
  ┌──────────────────────────────────────────────────────┐
  │ GPU-B 收到, NCCL 继续                                   │
  └──────────────────────────────────────────────────────┘
  
  ⬅─── (7) CNP 反向到 SuperNIC A → DCQCN 降速
```

### 6.3 4 大协作机制

```
═══════════════════════════════════════════════════════════════
  机制 1: Packet Spraying (包喷洒)
═══════════════════════════════════════════════════════════════

  传统:
    一个 flow (5-tuple 相同) 走一条固定路径
    → 大象流挤死小流
    → 一条路径满, 其他空闲
  
  Spectrum-X:
    一个 flow 的包打散到所有 spine (per-packet)
    → 所有路径均匀使用
    → 单 flow 也能跑满带宽
  
  谁参与:
    SuperNIC (发): 不按 flow hash, 按包打散
    交换机:        AR 配合, 选最空闲路径
    SuperNIC (收): 硬件重排乱序包


═══════════════════════════════════════════════════════════════
  机制 2: Adaptive Routing (自适应路由)
═══════════════════════════════════════════════════════════════

  传统 ECMP:
    hash(5-tuple) % N → 选 next-hop
    不感知队列状态
  
  Spectrum-X AR:
    交换机看 spine 队列深度, 选最空闲
    每包独立决策 (per-packet)
  
  ★ 没 SuperNIC 配合, AR 不能用 (乱序包会被丢)


═══════════════════════════════════════════════════════════════
  机制 3: DCQCN 增强 (拥塞控制)
═══════════════════════════════════════════════════════════════

  传统 DCQCN:
    交换机 ECN 标记 → 接收 NIC 发 CNP → 发送 NIC 降速
  
  Spectrum-X 增强:
    交换机: 更精细 ECN (基于队列深度梯度, 非阈值)
    SuperNIC: 更快 CNP 响应 (硬件实现, μs 级)
    SuperNIC: 更精细速率调整 (per-QP, 非 per-port)


═══════════════════════════════════════════════════════════════
  机制 4: 端到端 Telemetry
═══════════════════════════════════════════════════════════════

  交换机 (NetQ) + SuperNIC (DOCA Telemetry) 数据汇总
  在 NetQ / Grafana 看端到端流
  
  例: NCCL all-reduce 慢
      → 看到 leaf-3 出口队列高
      → 看到 SuperNIC-7 CNP 收得多
      → 定位: spine-5 故障导致 AR 误判
```

### 6.4 对比 InfiniBand 全栈

| 能力 | Spectrum-X (Ethernet) | InfiniBand |
|------|----------------------|------------|
| Packet Spraying | Adaptive Routing | 天生支持 (verbs) |
| 无损 | PFC + DCQCN | Credit-based 流控 |
| 拥塞控制 | DCQCN 反应式 | Credit + 主动 |
| 端到端调度 | DOCA + SuperNIC | SHARP + UFM |
| 统一管理 | NetQ + DOCA Telemetry | UFM |
| 网络协议 | UDP/IP/Ethernet | IB native |

> **★ 核心目标**：让 Ethernet 拥有接近 InfiniBand 的性能
> **★ 必须三件套**：Spectrum 交换机 + SuperNIC + DOCA，任何一个换成普通设备就退化

---

## 第七部分：DOCA Service - HBN（额外重点）

```
═══════════════════════════════════════════════════════════════
  HBN = Host-Based Networking
═══════════════════════════════════════════════════════════════

  作用: 把 Cumulus Linux 的 BGP/EVPN 功能搬到 DPU 上跑
  
  传统:
    GPU 服务器 → leaf 交换机 → BGP 邻居
    (主机不参与路由, leaf 做 EVPN)
  
  HBN:
    GPU 服务器内的 DPU → 直接和 leaf BGP 邻居
    DPU 自己跑 FRR
    DPU 自己做 EVPN VTEP
  
  优势:
    - 网络配置 100% 在 DPU 完成
    - 主机 OS 看到"原始网络", 不感知 overlay
    - 多租户隔离更彻底
  
  ★ 考点: HBN 是 DOCA Service, 不是独立产品
  ★ 位置: DOCA 4 层架构的 Layer 3 (Services)
```

---

## 第八部分：易错点速查

| ❌ 错误认知 | ✅ 正确 |
|------------|--------|
| DOCA 替代 OFED | DOCA 在 OFED 之上，OFED 仍必需 |
| AI 网络都不虚拟化 | 训练不虚拟化，云/推理必须虚拟化 |
| GPUDirect RDMA 是 DOCA 的功能 | 2013 就有，早于 DOCA 10 年 |
| GPUNetIO = GPUDirect RDMA | 不同！GPUNetIO 控制面也 bypass CPU |
| SuperNIC 是新硬件 | 是 BlueField-3 的产品形态 (固件不同) |
| Spectrum-X 是单一产品 | 是"交换机+SuperNIC+DOCA"全栈 |
| ConnectX-8 SuperNIC 跑 OS | 不跑！只有 BF-3 SuperNIC 跑 OS |
| AR 可以独立用 | 必须 SuperNIC 配合 (否则乱序丢包) |
| Profile 切完立即生效 | 必须重启服务 `systemctl restart doca-*` |
| 固件随便选 | 必须严格配套 DOCA 版本 |
| 多个 DOCA Profile 可共存 | 互斥，必须先卸载 |
| BFB 刷写保留配置 | 全清！必须先备份 `/etc/doca/` |
| Host Mode 和 DPU Mode 性能一样 | DPU Mode 主机 CPU 占用低很多 |
| HBN 是独立产品 | 是 DOCA Service，属架构 Layer 3 |

---

## 第九部分：自测题

回答前先盖答案：

1. **DOCA 全称？类比 CUDA 对应什么硬件？**
   <details><summary>答</summary>Data Center Infrastructure-on-a-Chip Architecture。CUDA→GPU；DOCA→DPU/SuperNIC。</details>

2. **DOCA 4 层架构？助记口诀？**
   <details><summary>答</summary>ASLD = Apps / Services / Libraries / Driver（从上到下）。</details>

3. **SuperNIC 和 BlueField DPU 关系？**
   <details><summary>答</summary>SuperNIC 是 BF-3 的产品形态（固件不同）。SuperNIC 强化 RoCE，弱化通用 offload，专为 AI。</details>

4. **AI 网络什么场景不用虚拟化？什么必须用？**
   <details><summary>答</summary>不用：单租户训练（性能极致）。必须用：AI 云 GPUaaS（多租户隔离）、AI 推理混部（K8s 网络）。</details>

5. **DOCA 两种部署模式？AI 集群选哪个？**
   <details><summary>答</summary>Host Mode（SDK 主机）vs DPU Mode（SDK DPU，推荐）。AI 训练用 SuperNIC 直通即可；多租户云用 DPU Mode。</details>

6. **4 种 DOCA Profile？AI 集群选哪个？**
   <details><summary>答</summary>doca-networking / doca-roce / doca-security / doca-all。AI 集群选 doca-roce。</details>

7. **Profile 切换 3 种方式？3 个陷阱？**
   <details><summary>答</summary>方式：apt 主机侧 / apt DPU 侧 / BFB 整刷。陷阱：互斥不能共存 / 切完必重启服务 / BFB 重刷丢配置。</details>

8. **DOCA 出现前，NVIDIA 怎么实现 RDMA/RoCE？**
   <details><summary>答</summary>MLNX_OFED + libibverbs（1999 至今）。NCCL 用 GPUDirect RDMA（2013），都早于 DOCA。DOCA 是 OFED 之上的新层，不是替代。</details>

9. **CPU Bypass 3 个层次？**
   <details><summary>答</summary>① 普通 RDMA (1999, libibverbs)：控制面 CPU+数据面 bypass。② GPUDirect RDMA (2013)：GPU↔NIC 直接 DMA，控制面仍 CPU。③ DOCA GPUNetIO (2022)：控制面+数据面都 bypass。</details>

10. **完整安装的 4 个 Phase？**
    <details><summary>答</summary>① 准备（硬件/OS/rshim 确认）② 固件升级（必须配套）③ 装 DOCA（3 方式：Host Package/BFB/Container）④ 验证（doca_info、ibv_devices、RDMA 性能测试）。</details>

11. **为什么固件必须和 DOCA 版本配套？**
    <details><summary>答</summary>不配套 = 装完也不识别，RoCE 不工作。DOCA 2.5↔BF FW 32.39.x↔OFED 23.10，必查官方 release notes。</details>

12. **Spectrum-X 三方协作的 4 大机制？**
    <details><summary>答</summary>① Packet Spraying（包喷洒）② Adaptive Routing（自适应路由）③ DCQCN 增强（拥塞控制）④ 端到端 Telemetry。</details>

13. **为什么三方必须协同，缺一退化？**
    <details><summary>答</summary>只有交换机无 SuperNIC：乱序包被丢→退 ECMP。只有 SuperNIC 无交换机：喷洒浪费。只有硬件无 DOCA：应用调不到。三件套缺一不可。</details>

14. **HBN 是什么？属于 DOCA 哪一层？**
    <details><summary>答</summary>Host-Based Networking，把 BGP/EVPN 搬到 DPU 上跑。属 DOCA Layer 3（Services）。</details>

15. **Spectrum-X vs InfiniBand 核心目标？**
    <details><summary>答</summary>让 Ethernet 性能接近 IB。Spectrum-X 用 AR+SuperNIC 实现 IB 的 packet spraying，用 DCQCN 实现无损。</details>

---

## 第十部分：一图总览

```
┌──────────────────────────────────────────────────────────────┐
│                  DOCA 知识图谱                                │
│                                                                │
│  概念                                                          │
│    DOCA = BlueField/SuperNIC 的 SDK                           │
│    类比 CUDA, 全称 Data Center Infra-on-a-Chip               │
│                                                                │
│  架构 (ASLD 4 层)                                              │
│    Apps → Services → Libraries → Driver → 硬件                │
│                                                                │
│  硬件家族                                                      │
│    NIC → SmartNIC → DPU → SuperNIC                            │
│    BF-2 (200G) / BF-3 (400G) / CX-8 SuperNIC (800G)           │
│                                                                │
│  部署模式                                                      │
│    Host Mode vs DPU Mode (推荐)                                │
│                                                                │
│  4 种 Profile                                                  │
│    networking / roce / security / all                          │
│    AI 选 doca-roce                                             │
│    切换: apt / BFB (互斥, 必重启, 防丢配)                      │
│                                                                │
│  安装 4 Phase                                                  │
│    准备 → 固件 (★ 配套) → 装 DOCA → 验证                     │
│                                                                │
│  4 个必考库                                                    │
│    Flow / GPUNetIO / Comm Channel / Telemetry                  │
│                                                                │
│  历史关系 (★ 易错)                                             │
│    OFED + libibverbs (1999) → 永远基础                         │
│    GPUDirect RDMA (2013) → NCCL 用                            │
│    DOCA GPUNetIO (2022) → 控制+数据面都 bypass                │
│                                                                │
│  Spectrum-X 三件套                                             │
│    交换机 + SuperNIC + DOCA                                    │
│    4 协作机制: 喷洒 / AR / DCQCN / Telemetry                  │
│    缺一退化为普通 RoCE                                          │
│                                                                │
│  HBN: DOCA Service (Layer 3), 把 BGP/EVPN 搬到 DPU            │
└──────────────────────────────────────────────────────────────┘
```

---

## 参考资料

- [DOCA Documentation Hub](https://docs.nvidia.com/doca/sdk/index.html)
- [BlueField DPU Product Page](https://www.nvidia.com/en-us/networking/products/data-processing-unit/)
- [Spectrum-X Platform](https://www.nvidia.com/en-us/networking/spectrumx/)
- [DOCA Release Notes (版本配套表)](https://docs.nvidia.com/doca/sdk/release-notes/)
- [GPUDirect RDMA (历史)](https://docs.nvidia.com/cuda/gpudirect-rdma/)

---

## 一句话总结

> **DOCA 是 BlueField/SuperNIC 的 SDK**，类比 CUDA：CUDA 是 GPU 编程，DOCA 是 DPU 编程。
>
> **核心价值**：让虚拟化和性能不再对立——多租户云既能用 OVS，又能跑满线速。
>
> **Spectrum-X 三件套**：交换机 + SuperNIC + DOCA，缺一退化为普通 RoCE。
>
> **AI 集群选型**：训练 → SuperNIC + doca-roce 直通；推理/多租户 → DPU + doca-networking。
