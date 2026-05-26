# Cumulus Linux Lab 笔记：EVPN Symmetric Routing + MLAG

> **来源**：NVIDIA Air "Distributed EVPN Symmetric Routing" Demo Lab
> **平台**：Cumulus Linux 5.16.1 (Cumulus VX)
> **配套文件**：`/Users/megsun/Documents/AI/NVIDIA DSX Air.html`
> **日期**：2026 年 5 月

本笔记整理与 Cumulus Linux Lab 相关的所有讨论点：SONiC 架构原理、与 Cumulus 对比、BGP unnumbered、MLAG、EVPN 对称路由、VRR、NVUE，并附完整 lab 配置解读。

---

## 目录

1. [SONiC 架构与对比](#1-sonic-架构与对比)
2. [Cumulus 接口命名 swp](#2-cumulus-接口命名-swp)
3. [vtysh 与 FRR 架构](#3-vtysh-与-frr-架构)
4. [BGP Unnumbered](#4-bgp-unnumbered)
5. [MLAG 与 Leaf 间 BGP 邻居](#5-mlag-与-leaf-间-bgp-邻居)
6. [Cisco vs Cumulus 对照](#6-cisco-vs-cumulus-对照)
7. [Lab 拓扑全景](#7-lab-拓扑全景)
8. [EVPN Symmetric Routing 原理](#8-evpn-symmetric-routing-原理)
9. [VRR 任播网关](#9-vrr-任播网关)
10. [Lab 完整配置逐段解读](#10-lab-完整配置逐段解读)
11. [验证命令速查](#11-验证命令速查)
12. [与 AI 训练网络的关系](#12-与-ai-训练网络的关系)

---

## 1. SONiC 架构与对比

> 虽然本 Lab 以 Cumulus 为主，但 SONiC 是 NVIDIA Spectrum 平台另一个官方 NOS 选项，理解它有助于把握现代 NOS 设计的全貌。

### 1.1 SONiC 是什么

**SONiC** = Software for Open Networking in the Cloud

- **起源**：Microsoft Azure 2016 年开源
- **托管**：Linux Foundation（OCP - Open Compute Project）
- **定位**：开源、厂商无关的网络操作系统
- **生产用户**：Microsoft Azure、Meta、阿里云、腾讯云、LinkedIn、Comcast
- **NVIDIA 角色**：商业化版本 **NVIDIA Enterprise SONiC**（2023 商业化），与 Cumulus 双线并行

### 1.2 整体架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                        管理层 (Management)                        │
│  CLI(sonic-cli)   gNMI/gNOI   REST API   SNMP   Ansible          │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────┐
│                  控制平面 (Control Plane)                         │
│                                                                  │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐ │
│  │ bgp  │ │ lldp │ │teamd │ │ snmp │ │ pmon │ │dhcp_ │ │ mgmt │ │
│  │(FRR) │ │      │ │(LAG) │ │      │ │      │ │relay │ │      │ │
│  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘ │
│     │        │        │        │        │        │        │     │
│     └────────┴────────┴────┬───┴────────┴────────┴────────┘     │
│                            ▼                                     │
│                ┌───────────────────────┐                         │
│                │   Redis DB (中枢)      │  ← 所有组件的"数据总线" │
│                │                       │                         │
│                │  CONFIG_DB  (配置)    │                         │
│                │  APPL_DB    (应用状态)│                         │
│                │  STATE_DB   (运行态)  │                         │
│                │  COUNTERS_DB(统计)    │                         │
│                │  ASIC_DB    (ASIC 态) │                         │
│                └───────────┬───────────┘                         │
│                            ▼                                     │
│                ┌───────────────────────┐                         │
│                │   swss container      │  ← Switch State Service │
│                │   (orchagent 协调器)  │  ← 所有变更统一编排     │
│                └───────────┬───────────┘                         │
└────────────────────────────┼─────────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│              Hardware Abstraction Layer (HAL)                    │
│                                                                  │
│            ┌─────────────────────────────┐                       │
│            │     syncd container         │  ← 同步 ASIC_DB → SAI │
│            └──────────────┬──────────────┘                       │
│                           ▼                                      │
│            ┌─────────────────────────────┐                       │
│            │    SAI (Switch Abstraction  │  ← 标准化 ASIC API    │
│            │         Interface)          │                       │
│            └──────────────┬──────────────┘                       │
└───────────────────────────┼──────────────────────────────────────┘
                            ▼
┌──────────────────────────────────────────────────────────────────┐
│              ASIC 硬件层 (厂商各自实现 SAI)                       │
│                                                                  │
│   NVIDIA Spectrum │ Broadcom Tomahawk │ Marvell Teralynx │ ...   │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 关键组件解释

#### 1.3.1 Redis DB（中枢神经）

SONiC 把"所有状态"放在 Redis 内存数据库里，按用途分多个 DB：

| DB 名 | 编号 | 用途 |
|-------|------|------|
| **CONFIG_DB** | 4 | 用户配置（接口、VLAN、BGP 邻居等）。**真相源** |
| **APPL_DB** | 0 | 应用进程发布的"期望状态"（如 BGP 进程要下发的路由） |
| **STATE_DB** | 6 | 系统运行时状态（接口 oper status、邻居状态等） |
| **COUNTERS_DB** | 2 | 端口/队列流量统计、SNMP 计数 |
| **ASIC_DB** | 1 | ASIC 实际编程状态（SAI 对象树） |
| **LOGLEVEL_DB** | 3 | 日志级别 |
| **FLEX_COUNTER_DB** | 5 | 灵活计数器配置 |

**关键认知**：
- 所有容器**不直接互相调用**，全部通过 Redis 通信
- 这种"发布/订阅"模型让模块解耦
- 但也带来"配置改 A 间接影响 B"的副作用（见 1.6）

#### 1.3.2 容器一览

| 容器 | 内部进程 | 职责 |
|------|---------|------|
| **bgp** | FRR (bgpd, zebra, staticd) | BGP/OSPF/static 路由 |
| **lldp** | lldpd | LLDP 邻居发现 |
| **teamd** | teamd | LAG/bond 管理 |
| **snmp** | snmpd | SNMP 服务 |
| **syncd** | syncd | **核心**：Redis ASIC_DB ↔ SAI 同步 |
| **swss** | orchagent, portsyncd, intfmgrd, vlanmgrd... | **核心协调器** |
| **pmon** | sensord, fancontrol, ledd | 平台监控（风扇、温度、LED） |
| **dhcp_relay** | dhcrelay | DHCP 中继 |
| **database** | redis-server | Redis 实例 |
| **telemetry** | gnmi-server | gNMI/gNOI 流式遥测 |

#### 1.3.3 SAI（Switch Abstraction Interface）

```
SAI 的价值：
    ┌──────────────────────────────┐
    │   SONiC 上层逻辑（一份）      │
    └──────────────┬───────────────┘
                   │
                   ▼ 调用 SAI API（标准化）
    ┌──────────────────────────────┐
    │   SAI Adapter (厂商各自实现) │
    └──┬────┬────┬────┬────┬───────┘
       │    │    │    │    │
       ▼    ▼    ▼    ▼    ▼
    NVIDIA Broadcom Marvell Barefoot Cisco
    Spectrum Tomahawk Teralynx Tofino  Silicon One
```

- SAI 由 OCP 标准化（C 头文件 + 行为规范）
- ASIC 厂商各自实现 SAI Adapter（通常闭源）
- SONiC 因此**真正跨厂商**

#### 1.3.4 swss + orchagent（协调器）

```
swss container 内部：
  ┌──────────────────────────────────────┐
  │  portsyncd  → 处理端口事件            │
  │  intfmgrd   → 处理接口配置            │
  │  vlanmgrd   → 处理 VLAN 配置          │
  │  nbrmgrd    → 处理邻居/ARP            │
  │  ...                                  │
  │  ┌────────────────────────────────┐   │
  │  │  orchagent (核心)              │   │
  │  │  - 订阅 APPL_DB 变化          │   │
  │  │  - 处理依赖（先建 VLAN 再加端口）│   │
  │  │  - 写入 ASIC_DB                │   │
  │  └────────────────────────────────┘   │
  └──────────────────────────────────────┘
```

**orchagent 是所有配置变更的"漏斗"**：不管哪个容器要改 ASIC，都得通过它，避免竞态条件。

### 1.4 配置变更完整流程

以"添加一个 VLAN"为例：

```
用户操作：config vlan add 100
   ↓
[1] sonic-cli 调用 sonic-cfggen
   ↓
[2] 写入 CONFIG_DB (Redis DB 4)
   "VLAN|Vlan100": {"vlanid": "100"}
   ↓
[3] vlanmgrd（在 swss 容器）订阅了 CONFIG_DB 变化，收到通知
   ↓
[4] vlanmgrd 创建内核 bridge（Linux 层），写入 APPL_DB
   ↓
[5] orchagent 订阅了 APPL_DB 变化，收到通知
   ↓
[6] orchagent 调用 SAI API，把 VLAN 对象写入 ASIC_DB
   ↓
[7] syncd 订阅了 ASIC_DB 变化，调用真正的 SAI Adapter
   ↓
[8] SAI Adapter 把 VLAN 编程到 ASIC 硬件
   ↓
[9] 完成，状态写入 STATE_DB
```

**6 跳的配置链路** vs 传统 NOS 的 2 跳。换来的是：解耦、可观测、可调试。

### 1.5 SONiC vs Cumulus 全面对比

| 维度 | SONiC | Cumulus Linux |
|------|-------|---------------|
| **起源** | Microsoft 开源 (2016) | Cumulus Networks (2010)，NVIDIA 收购 (2020) |
| **基础 OS** | Debian | Debian |
| **架构风格** | 容器化微服务 + Redis | 单体 Linux + 标准守护进程 |
| **配置 DB** | Redis (CONFIG_DB)，JSON 模型 | NVUE YAML + 内核 + FRR vtysh |
| **路由栈** | FRR（在 bgp 容器） | FRR（直接系统进程） |
| **配置 CLI** | `config` / `sonic-cli` / `show` | `nv set` (NVUE) / `vtysh` |
| **配置体验** | 命令式，需理解多组件 | 声明式，object model 完整 |
| **学习曲线** | 陡（要懂 Redis/容器） | 平缓（标准 Linux + NVUE） |
| **故障恢复** | 容器自愈，秒级 | 进程级，类似 systemd |
| **升级粒度** | 单容器（`docker restart`） | 整机包升级 |
| **可观测性** | Redis 直接查，Prometheus 友好 | Linux 工具 + NetQ |
| **厂商支持** | 多 ASIC（SAI 标准） | 主要 NVIDIA Spectrum + 部分 Broadcom |
| **商业支持** | NVIDIA Enterprise SONiC, Aviz, Edgecore | NVIDIA（原厂） |
| **典型用户** | 超大规模云（Azure/Meta/阿里） | 企业、金融、中等规模 AI |
| **配置模板化** | Jinja2 + minigraph.xml | NVUE + Ansible |
| **AI 集成度** | 与 Spectrum-X 集成（NVIDIA 版本） | 与 Spectrum-X 深度集成 |

### 1.6 SONiC 容器化的"独立性"边界

这是 SONiC 架构最容易被误解的点：

```
✅ 独立的层面（工程层）：
   - 进程生命周期独立（一个容器 crash 不影响其他容器进程）
   - 资源隔离（cgroup CPU/内存）
   - 版本升级独立（docker pull / restart）
   - 故障域隔离
   - 开发流程独立（Microsoft 维护 BGP、NVIDIA 维护 syncd）

❌ 不独立的层面（功能层）：
   - 共享同一块 ASIC 硬件
   - 共享 ASIC 资源（TCAM、队列、端口）
   - 共享 Redis 数据库
   - 控制平面流量路径互相影响
```

**经典反例**：删除 control-plane ACL 中允许 TCP 179 的规则
→ BGP 容器进程 100% 正常运行
→ 但 ASIC 不再把 BGP 报文转给 CPU
→ Hold timer 超时后 BGP 邻居 down → 路由全撤

**类比**：

```
SONiC 容器架构 ≈ Kubernetes 微服务
  - 订单服务、支付服务独立部署
  - 但共享同一个 MySQL → 删除关键表全崩
  
SONiC 上：
  - BGP、ACL、LLDP 容器独立部署
  - 但共享同一块 ASIC → 删除 control-plane ACL 全断
```

**核心认知**：SONiC 的容器化是"**软件工程的独立**"，**不是"网络功能的独立**"。网络协议栈的功能耦合是协议本质决定的，任何 NOS 都无法消除。

### 1.7 在 AI/HPC 领域的应用

| 领域 | SONiC 应用情况 |
|------|---------------|
| HPC（超算 TOP500） | 极少。InfiniBand 主导，无动力换 |
| AI 超大规模云 | 快速渗透（Azure 引领，趋势明确） |
| AI 中小集群 | Cumulus 仍是主流（NVUE 体验好） |

### 1.8 性能对比：SONiC vs 传统 NOS

**反直觉认知**：SONiC 在"数据平面性能"上**没有优势**，只在"控制平面工程效率"上有显著优势。

| 维度 | SONiC vs 传统 NOS | 备注 |
|------|------------------|------|
| 数据平面带宽 | **无差异** | ASIC 决定 |
| 数据平面延迟 | **无差异** | ASIC 决定 |
| RoCE 性能 | **无差异** | ASIC + NIC 决定 |
| BGP 收敛时间 | 基本相同（3-5s） | FRR 实现一致 |
| 接口配置生效 | SONiC 略慢（多跳） | 6 跳 vs 2 跳 |
| 整机启动时间 | SONiC 略慢（90-120s vs 60-90s） | 容器启动开销 |
| **组件升级中断** | **SONiC 快 10-60×** | 5-30s vs 5-15min |
| **故障恢复时间** | **SONiC 快 10×** | 2-5s vs 30-60s |
| **配置吞吐量** | **SONiC 快 5-10×** | gNMI/Redis 并行 |
| 内存占用 | SONiC 高 2-3×（3-5GB vs 1-2GB） | 容器 + Redis 副本 |
| 磁盘占用 | SONiC 高 3×（1.5-2GB vs 500MB） | 容器镜像 |

### 1.9 对 AI 训练性能影响有多大？

```
AI 训练性能影响因素：

  ┌────────────────────────────────────────────────────┐
  │  因素                影响    NOS 是否相关          │
  ├────────────────────────────────────────────────────┤
  │  GPU 算力           ★★★★★   ❌ 完全无关          │
  │  NVLink 带宽        ★★★★★   ❌ 完全无关          │
  │  RDMA 带宽          ★★★★    ❌ 由 ASIC/NIC 决定  │
  │  RDMA 延迟          ★★★★    ❌ 由 ASIC/NIC 决定  │
  │  拥塞控制效果       ★★★★    ❌ 由 ASIC 决定      │
  │  拓扑 (Rail-Opt)    ★★★     ❌ 由物理连线决定     │
  │  故障恢复速度       ★★      ✅ NOS 有影响        │
  │  运维自动化效率     ★★      ✅ NOS 有影响        │
  └────────────────────────────────────────────────────┘
  
  → NOS 对训练性能的直接影响 < 5%
  → 99% 性能由硬件 + 拓扑 + 拥塞控制决定
```

**结论**：
- AI 训练性能：SONiC 不会让训练更快
- AI 集群运维：SONiC 在 10,000+ 节点规模下显著降低运维成本（50%+）
- 小规模集群（< 100 节点）：SONiC 收益不明显，反而增加复杂度

### 1.10 NVIDIA 的双轨策略

```
NVIDIA Spectrum 硬件 (SN5600 / SN4000)
            │
   ┌────────┴────────┐
   │                 │
   ▼                 ▼
Cumulus Linux    NVIDIA Enterprise SONiC
   │                 │
   ├ 中等规模 AI    ├ 超大规模云
   ├ 企业 DC        ├ Azure-style fabric
   ├ NVUE 体验佳    ├ 拥抱开源社区
   └ 商业支持完整   └ 多厂商兼容
```

NVIDIA 不强推任何一种，让客户按场景选择。两者都与 **Spectrum-X** 深度集成（Adaptive Routing、增强 ECN、CNP 加速等）。

### 1.11 学习优先级建议

| 目标 | 推荐 |
|------|------|
| NCP-AIN 认证 | **Cumulus + NVUE 优先**，SONiC 作为加分项 |
| 企业 / 中等 AI 集群 | Cumulus + NVUE |
| 超大规模云 / AI 云 | SONiC 必学（Redis、orchagent、容器调试） |
| 跨厂商混合采购 | SONiC |
| 快速上手 | Cumulus（NVUE 声明式体验佳） |

---

## 2. Cumulus 接口命名 swp

### 2.1 命名规则

- **`swp` = Switch Port**（Cumulus 专属命名）
- 与面板物理端口一一对应：`swp1` ~ `swp64`
- Breakout 子端口：`swp1s0`, `swp1s1`, `swp1s2`, `swp1s3`

### 2.2 厂商对比

| 厂商 | 命名风格 | 示例 |
|------|---------|------|
| Cumulus | `swp<N>` | `swp51` |
| SONiC | `Ethernet<N>` | `Ethernet0` |
| Cisco NX-OS | `Ethernet<slot>/<N>` | `Ethernet1/51` |
| Arista EOS | `Ethernet<N>` | `Ethernet51` |
| Juniper | `xe-/et-/ge-` | `et-0/0/51` |
| Linux server | `eth<N>` / `ens` / `enp` | `eth0`, `ens33` |

### 2.3 关键认知

Cumulus 是"装在交换机硬件上的 Linux"，`swp51` 在内核里就是标准 Linux 网络设备：

```bash
ip link show swp51
ip addr show swp51
ethtool swp51
tcpdump -i swp51
```

---

## 3. vtysh 与 FRR 架构

### 3.1 vtysh = Virtual TeletYpe SHell

是 FRR（FRRouting）提供的**统一 CLI shell**。

### 3.2 FRR 架构

```
┌──────────────────────────────────────────────────┐
│              vtysh (统一 CLI 入口)               │
└────────┬──────┬──────┬──────┬──────┬─────────────┘
         │      │      │      │      │
         ▼      ▼      ▼      ▼      ▼
      ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
      │bgpd│ │ospfd│ │zebra│ │pim│ │...│   ← 每个协议独立 daemon
      └────┘ └────┘ └────┘ └────┘ └────┘
         ↑      ↑      ↑      ↑      ↑
         各 daemon 有自己的 VTY socket
```

### 3.3 使用方式

```bash
# 交互模式
sudo vtysh
leaf01# show bgp summary

# 单命令模式（脚本友好）
sudo vtysh -c 'show bgp sum'

# 多条命令串联
sudo vtysh -c 'show bgp sum' -c 'show ip route'
```

### 3.4 Cumulus 上的两种配置路径

| 路径 | 用途 | 风格 |
|------|------|------|
| **NVUE** (`nv set ...`) | 生产配置 | 声明式、有事务、可回滚 |
| **vtysh** | 排障、临时调试 | 命令式、直接、灵活 |

### 3.5 与 Cisco IOS 命令对照

| Cisco IOS | Cumulus / FRR |
|-----------|---------------|
| `show ip bgp summary` | `sudo vtysh -c 'show bgp summary'` |
| `show ip route` | `sudo vtysh -c 'show ip route'` 或 `ip route show` |
| `show interface brief` | `sudo vtysh -c 'show interface brief'` 或 `nv show interface` |
| `configure terminal` | `sudo vtysh` → `configure terminal` |

---

## 4. BGP Unnumbered

### 4.1 一句话定义

> 用接口名（如 `swp51`）建立 BGP 邻居，**不需要给点对点链路配 IP 地址**。
> 底层用 **IPv6 Link-Local 地址 + RFC 5549**（IPv6 next-hop for IPv4 routes）实现。

### 4.2 传统 BGP vs Unnumbered

```
传统 (Numbered):
  leaf01 ─── swp51 ─── 10.0.1.1/30 ─── 10.0.1.2/30 ─── swp1 ─── spine01
  
  问题：100 leaf × 4 spine = 400 个 /30 子网要规划

BGP Unnumbered:
  leaf01 ─── swp51 ════════════════════ swp1 ─── spine01
              ↑                          ↑
         IPv6 LLA 自动生成           IPv6 LLA 自动生成
         fe80::xxxx                 fe80::yyyy

  配置（leaf01）：
    nv set vrf default router bgp neighbor swp51 remote-as external
```

### 4.3 工作流程

1. **邻居发现**：接口启用 IPv6 → 自动生成 LLA → IPv6 RA 通告
2. **会话建立**：BGP 用 LLA 建立 TCP 179 会话
3. **路由交换**：IPv4 路由的 next-hop 是 IPv6 LLA（RFC 5549）
4. **转发**：查 IPv4 路由表 → NDP 解析 LLA → 用对端 MAC 封装 IPv4 包

### 4.4 厂商支持

| 厂商 | 支持情况 |
|------|---------|
| Cumulus Linux | ✓ 原生主推 |
| SONiC | ✓ 通过 FRR |
| FRR (开源) | ✓ 原生 |
| Arista EOS | ✓ 较新版本 |
| Cisco NX-OS | ⚠ 部分平台 |
| Cisco IOS/IOS-XE | ✗ 不支持 |

---

## 5. MLAG 与 Leaf 间 BGP 邻居

### 5.1 为什么 MLAG 对之间需要 BGP？

正常的 Spine-Leaf 拓扑中 Leaf 之间不应有 BGP，但当两台 Leaf 组成 MLAG 对时**必须**有：

```
故障场景：leaf01 上行全断

   spine01    spine02    spine03    spine04
      X swp51   │swp52      │swp53     │swp54  ← leaf01 上行全断
      X         │           │          │
┌─────┴──────┐ ┌┴──────────┐┌─┴─────────┴─┐
│  leaf01    │ │  leaf02   │
│  上行全断  │ │  正常     │
└─────┬──────┘ └─────┬─────┘
      │  peerlink.4094 │
      │◄──── BGP ─────►│  ← 救命路径
      └────────────────┘

  server → leaf01 → peerlink → leaf02 → spine
```

**没有这条 BGP 邻居**：leaf01 默认路由失效 → 所有上行流量被丢弃。

### 5.2 lab 里的实际输出

```
Neighbor              V    AS      Up/Down  State/PfxRcd
leaf02(peerlink.4094) 4    65102   00:42:27  12       ← MLAG peer (numbered, VLAN SVI)
spine01(swp51)        4    65100   00:42:28  8        ← BGP unnumbered
spine02(swp52)        4    65100   00:42:28  8
spine03(swp53)        4    65100   00:42:28  8
spine04(swp54)        4    65100   00:42:28  8
```

**线索**：
- `peerlink.4094` → VLAN 4094 的 L3 SVI（Cumulus MLAG 控制 VLAN 约定）
- 接口名是 `swp51` → BGP unnumbered
- 接口名是 IP/SVI → numbered

### 5.3 Leaf 互联做 Multi-homing 的两种主流方案

| 流派 | 特点 | 适用场景 |
|------|------|---------|
| **MLAG** (传统主流) | 2 台 leaf 一组，server 用 LACP bond | 通用 DC、企业 |
| **EVPN Multihoming (ESI)** (新一代) | 通过 BGP EVPN 同步状态，可 4 台 | 大规模云、新建 DC |

**AI 训练场景的特殊性**：
- GPU 节点本身有 8 个 RDMA 网卡，每个独立接入不同 leaf（Rail-Optimized）
- **不需要 MLAG，也不需要 EVPN-MH**
- 直接 L3 + ECMP 就够了

---

## 6. Cisco vs Cumulus 对照

### 6.1 Multi-homing 方案对照

| 能力 | Cumulus Linux | Cisco NX-OS |
|------|---------------|-------------|
| 传统方案 | MLAG | vPC (virtual PortChannel) |
| 现代方案 | EVPN Multihoming | EVPN ESI Multihoming |
| Anycast 网关 | VRR | Anycast Gateway (HSRP/EVPN) |

### 6.2 术语对照

| Cumulus MLAG | Cisco vPC |
|--------------|-----------|
| peerlink | vPC peer-link |
| peer IP | vPC peer-keepalive |
| clag-id | vPC ID |
| clagd | vPC peer adjacency |
| backup-ip | peer-keepalive |

### 6.3 BGP 配置对比

```
Cisco Nexus（leaf01）：
─────────────────────────────────────
vpc domain 1
  peer-keepalive destination 10.1.1.2 source 10.1.1.1
  peer-switch
  peer-gateway

interface port-channel 100
  switchport mode trunk
  vpc peer-link

router bgp 65101
  neighbor 10.0.1.2 remote-as 65100   ← 必须用 IP

Cumulus（leaf01）：
─────────────────────────────────────
nv set interface peerlink bond member swp49,swp50
nv set mlag backup 10.10.10.2
nv set mlag peer-ip linklocal
nv set mlag mac-address 44:38:39:ff:00:01

nv set vrf default router bgp neighbor swp51 remote-as external  ← unnumbered!
```

### 6.4 设计理念差异

| 维度 | Cisco | Cumulus (NVIDIA) |
|------|-------|------------------|
| 理念 | 深度集成、垂直方案、统一管控 | Linux 化、开放、自动化优先 |
| BGP 配置 | 繁琐但 GUI 完整 | 简洁（BGP unnumbered） |
| 主战场 | 企业、金融 | 超大规模云、AI |
| AI 方案 | HyperFabric + Silicon One | Spectrum-X + Cumulus/SONiC |

---

## 7. Lab 拓扑全景

### 7.1 物理拓扑

```
                ┌─────────────────────────────────────┐
                │       Spine Layer (4 台, AS65100)   │
                │ spine01  spine02  spine03  spine04  │
                │ 10.10.10.101  .102    .103    .104  │
                └──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬┘
                   │  │  │  │  │  │  │  │  │  │  │  │
   swp51-54    swp51-54           swp51-54    swp51-54
   ┌────┴────┐ ┌────┴────┐    ┌────┴────┐ ┌────┴────┐
   │ leaf01  │═│ leaf02  │    │ leaf03  │═│ leaf04  │
   │ AS65101 │ │ AS65102 │    │ AS65103 │ │ AS65104 │
   │ 10.10.10.1│10.10.10.2│   │10.10.10.3│10.10.10.4│
   │ MLAG ──peerlink──┐ │    │ MLAG ──peerlink──┐ │
   │ anycast 10.0.1.12│      │ anycast 10.0.1.34│
   └────┬────┘ └────┬───┘    └────┬────┘ └────┬───┘
        │ bond1-3   │ bond1-3      │ bond1-3   │ bond1-3
        │           │              │           │
       ┌┴────────────┴┐           ┌┴────────────┴┐
       │ Rack 1       │           │ Rack 2       │
       │ server01-03  │           │ server04-06  │
       └──────────────┘           └──────────────┘

  + border01/02 (用于扩展实验，连接 server07/08, fw1/fw2)
```

### 7.2 关键 IPAM

| 设备 | loopback | MLAG anycast | Mgmt |
|------|----------|--------------|------|
| leaf01 | 10.10.10.1 | 10.0.1.12 | 192.168.200.6 |
| leaf02 | 10.10.10.2 | 10.0.1.12 | 192.168.200.7 |
| leaf03 | 10.10.10.3 | 10.0.1.34 | 192.168.200.8 |
| leaf04 | 10.10.10.4 | 10.0.1.34 | 192.168.200.9 |
| spine01-04 | 10.10.10.101-104 | - | 192.168.200.2-5 |

### 7.3 VLAN/VRF 划分

| VLAN | VNI | VRF | 网段 | 网关 (VRR) | 服务器 |
|------|-----|-----|------|-----------|--------|
| 10 | 10 (L2VNI) | RED | 10.1.10.0/24 | 10.1.10.1 | server01, 04 |
| 20 | 20 (L2VNI) | RED | 10.1.20.0/24 | 10.1.20.1 | server02, 05 |
| 30 | 30 (L2VNI) | BLUE | 10.1.30.0/24 | 10.1.30.1 | server03, 06 |
| 4063 | 4001 (L3VNI) | RED | (transit) | - | - |
| 4006 | 4002 (L3VNI) | BLUE | (transit) | - | - |

---

## 8. EVPN Symmetric Routing 原理

### 8.1 三层架构

```
┌───────────────────────────────────────────────────────────┐
│  层次             技术       作用                          │
├───────────────────────────────────────────────────────────┤
│  Overlay 控制   │ EVPN     │ 学习/通告 VTEP、MAC、IP     │
│  Overlay 数据   │ VXLAN    │ 跨 Spine 隧道传 L2 帧       │
│  Underlay 控制  │ BGP      │ 通告 VTEP loopback 可达性   │
│  Underlay 数据  │ IP       │ 物理 L3 IP 转发              │
│  物理拓扑       │ 铜/光纤  │ Spine-Leaf 全互联            │
└───────────────────────────────────────────────────────────┘
```

类比：Underlay = 高速公路；Overlay = 加密集装箱货车（VXLAN 隧道）；EVPN = 调度中心。

### 8.2 L2VNI vs L3VNI

```
┌──────────────────────────────────────────────────────────────┐
│  L2VNI (VNI 10/20/30):                                       │
│    - 1 个 VLAN 对应 1 个 L2VNI                                │
│    - 传输 Ethernet 帧（带 src/dst MAC）                       │
│    - 用于"同 VLAN、跨 leaf"的 L2 通信                         │
│                                                              │
│  L3VNI (VNI 4001/4002):                                      │
│    - 1 个 VRF 对应 1 个 L3VNI                                 │
│    - 传输 IP 包（内层 dst MAC = 远端 leaf MAC）              │
│    - 用于"跨 VLAN、跨 leaf"的 L3 通信                         │
│    - "对称路由"的核心机制                                    │
└──────────────────────────────────────────────────────────────┘
```

### 8.3 对称路由完整流程

**场景**：server01 (10.1.10.101, VLAN 10, VRF RED) → server05 (10.1.20.105, VLAN 20, VRF RED)

```
步骤 1：server01 发包
   src MAC: server01_mac
   dst MAC: leaf01 的 VRR MAC (10.1.10.1 的 MAC = 00:00:5e:00:01:10)
   src IP : 10.1.10.101
   dst IP : 10.1.20.105

步骤 2：leaf01 接收 → 在 VRF RED 内查路由
   show ip route vrf RED 10.1.20.105
   → B>* 10.1.20.105/32 via 10.0.1.34 (远端 anycast VTEP)
   → 路由出口：vlan4063_l3 (L3VNI 4001 对应的 SVI)

步骤 3：leaf01 封装 VXLAN
   外层 IP : src 10.0.1.12 (本端 MLAG anycast)
            dst 10.0.1.34 (leaf03/04 anycast)
   VXLAN VNI: 4001 (RED 的 L3VNI，不是 VNI 20!)
   内层 MAC: src leaf01 的 MAC
            dst leaf03 的 router MAC (通过 EVPN Type-2 学到)
   内层 IP : src 10.1.10.101
            dst 10.1.20.105

步骤 4：Spine 转发（Spine 不解 VXLAN，只看外层 IP）

步骤 5：leaf03/04（ECMP 选一台）收到
   外层 IP 10.0.1.34 = 自己 → 拆掉外层
   VNI 4001 → 找到对应 VRF = RED
   内层 dst MAC = 自己的 router MAC → 路由！

步骤 6：leaf03 在 VRF RED 内查路由
   10.1.20.105 → 直连 vlan20 → bond2

步骤 7：发包给 server05
   src MAC: leaf03 的 VRR MAC (10.1.20.1 的 MAC = 00:00:5e:00:01:20)
   dst MAC: server05_mac
```

### 8.4 对称 vs 非对称

| 模式 | 入口 leaf | 出口 leaf | 入口 leaf 需知道远端 VLAN | 扩展性 |
|------|----------|----------|--------------------------|--------|
| **非对称 (Asymmetric IRB)** | 路由 | 只桥接 | 是（每个远端 VLAN 都要配 SVI） | 差 |
| **对称 (Symmetric IRB)** | 路由 + 走 L3VNI | 路由 + 走目标 VLAN | 否（只需 VRF / L3VNI） | 好 ✓ |

**本 lab 使用对称路由**（现代 DC 主流）。

### 8.5 关键设计原则

> 多租户要求：**每个 VRF 一个 L3VNI**，所有参与该 VRF 的 leaf 必须配置**相同的 L3VNI**。
> 出口 leaf 使用 L3VNI 识别应该路由到哪个 VRF。

---

## 9. VRR 任播网关

### 9.1 配置原理

```
VLAN 10 在所有 leaf 上的网关配置（关键！）：

leaf01:
  vlan10    IP: 10.1.10.2/24    ← leaf01 自己的 IP
  vlan10-v0 IP: 10.1.10.1/24    ← VRR 虚拟网关 IP
  
leaf02:
  vlan10    IP: 10.1.10.3/24    ← leaf02 自己的 IP
  vlan10-v0 IP: 10.1.10.1/24    ← 同样的 VRR IP

leaf03:
  vlan10    IP: 10.1.10.4/24
  vlan10-v0 IP: 10.1.10.1/24    ← 全网 leaf 都用同一个 VRR IP！

leaf04:
  vlan10    IP: 10.1.10.5/24
  vlan10-v0 IP: 10.1.10.1/24
```

### 9.2 统一 VRR MAC

```
00:00:5e:00:01:10 → VRR for VLAN 10
00:00:5e:00:01:20 → VRR for VLAN 20
00:00:5e:00:01:30 → VRR for VLAN 30

全网 4 台 leaf 都用这些相同的 MAC 作为 SVI 的 MAC
→ server 不管接到哪台 leaf，ARP 永远成功
→ VM 在 leaf 之间迁移时无感知
```

### 9.3 VRR vs VRRP

| 特性 | VRR | VRRP |
|------|-----|------|
| 模式 | Active-Active | Active-Standby |
| 所有节点 | 都响应 | 只有 Master 响应 |
| 收敛时间 | 无（一直 active） | 秒级（选举） |
| 用途 | EVPN 任播网关 | 传统 HA |

---

## 10. Lab 完整配置逐段解读

> 完整配置见 leaf01 上 `nv config show` / `nv config show -o commands` 输出。
> 以下分组解读关键配置项。

### 10.1 Bridge（VLAN-aware 桥）

```
nv set bridge domain br_default type vlan-aware
nv set bridge domain br_default vlan 10 vni 10
nv set bridge domain br_default vlan 20 vni 20
nv set bridge domain br_default vlan 30 vni 30
```

- **`vlan-aware` 模式**：单个 bridge 承载多个 VLAN（vs 传统 `vlan-unaware` 每 VLAN 一个 bridge）
- **VLAN → VNI 直接映射**：VLAN 10 ↔ L2VNI 10

### 10.2 EVPN 全局开关

```
nv set evpn route-advertise svi-ip enabled
nv set evpn state enabled
```

- `svi-ip enabled`：把 SVI 的 IP/MAC 通过 EVPN Type-2 通告出去（实现 ARP 抑制）

### 10.3 服务器下联接口（bond1/2/3）

```
nv set interface bond1 bond member swp1
nv set interface bond1 bond mlag id 1            ← clag-id，MLAG 双归标识
nv set interface bond1 bridge domain br_default access 10  ← access VLAN 10

nv set interface bond1-3 bond lacp-bypass enabled      ← server 没起 LACP 也能通
nv set interface bond1-3 bond mlag state enabled       ← MLAG 双归
nv set interface bond1-3 bond mode lacp
nv set interface bond1-3 bridge domain br_default stp admin-edge enabled  ← 边缘端口
nv set interface bond1-3 bridge domain br_default stp auto-edge enabled
nv set interface bond1-3 bridge domain br_default stp bpdu-guard enabled  ← 防误接交换机
nv set interface bond1-3 link mtu 9216                 ← jumbo frame
nv set interface bond1-3 type bond
```

**关键点**：
- `clag-id` 相同的 bond 在 leaf01 / leaf02 上配对 → server 看到一个虚拟交换机
- `lacp-bypass`：开机时 LACP 还没建立也能通流量（避免 PXE boot 死锁）
- `admin-edge` + `bpdu-guard`：防止 server 接错位置变成交换机环路

### 10.4 管理口（带外网络）

```
nv set interface eth0 ipv4 dhcp-client set-hostname enabled
nv set interface eth0 ipv4 dhcp-client state enabled
nv set interface eth0 type eth
nv set interface eth0 vrf mgmt                  ← 关键：放在 mgmt VRF，与业务流量隔离
```

### 10.5 Loopback（BGP router-id + VTEP source）

```
nv set interface lo ipv4 address 10.10.10.1/32
nv set interface lo type loopback
```

- 用作 BGP router-id
- 用作 VXLAN VTEP source address（leaf01 独立的 VTEP IP）

### 10.6 MLAG Peerlink

```
nv set interface peerlink bond member swp49-50    ← 用 swp49+swp50 做 bond
nv set interface peerlink bridge domain br_default ← 加入 bridge（承载 L2 流量）
nv set interface peerlink type peerlink

nv set interface peerlink.4094 base-interface peerlink
nv set interface peerlink.4094 type sub
nv set interface peerlink.4094 vlan 4094           ← VLAN 4094 为 L3 SVI，跑 BGP
```

**两层用途**：
- `peerlink` 本体：L2 trunk，承载所有 VLAN 的 L2 流量
- `peerlink.4094`：L3 SVI，用于 leaf01 ↔ leaf02 之间的 BGP 邻居

### 10.7 上行口（Spine 互联）

```
nv set interface swp51-54 link state up
nv set interface swp51-54 type swp
```

- 没有配 IP 地址 → 走 BGP unnumbered
- 在后面 BGP 配置里直接用接口名做邻居

### 10.8 SVI + VRR 网关

```
# VLAN 10 (RED)
nv set interface vlan10 ipv4 address 10.1.10.2/24       ← leaf01 自己的 IP
nv set interface vlan10 ipv4 vrr address 10.1.10.1/24   ← VRR 虚拟网关
nv set interface vlan10 ipv4 vrr mac-address 00:00:5e:00:01:10
nv set interface vlan10 vlan 10

# VLAN 20 (RED)
nv set interface vlan20 ipv4 address 10.1.20.2/24
nv set interface vlan20 ipv4 vrr address 10.1.20.1/24
nv set interface vlan20 ipv4 vrr mac-address 00:00:5e:00:01:20
nv set interface vlan20 vlan 20

# VLAN 30 (BLUE)
nv set interface vlan30 ipv4 address 10.1.30.2/24
nv set interface vlan30 ipv4 vrr address 10.1.30.1/24
nv set interface vlan30 ipv4 vrr mac-address 00:00:5e:00:01:30
nv set interface vlan30 vlan 30
nv set interface vlan30 vrf BLUE                        ← 单独 BLUE VRF

# 批量赋 VRF
nv set interface vlan10,20 vrf RED                      ← 一行配 2 个

# 批量开 VRR
nv set interface vlan10,20,30 ipv4 vrr state enabled
nv set interface vlan10,20,30 ipv4 vrr vrr-state up
nv set interface vlan10,20,30 type svi
```

**NVUE 范围语法亮点**：`vlan10,20` 同时配置两个接口；`bond1-3` 同时配置三个。

### 10.9 MLAG 全局参数

```
nv set mlag backup 10.10.10.2          ← peer 的 loopback IP（备份心跳路径）
nv set mlag init-delay 10              ← 启动时等 10s 再加入 MLAG
nv set mlag mac-address 44:38:39:BE:EF:AA  ← 共用系统 MAC（关键！）
nv set mlag peer-ip linklocal          ← peer 通信用 IPv6 LLA（不需要 IP 规划）
nv set mlag priority 1000              ← 优先级（决定 primary）
nv set mlag state enabled
```

**`mac-address`**：两台 leaf 共用同一个 MAC 作为系统 MAC，让 server 看到的 LACP partner MAC 一致。

### 10.10 NVE / VXLAN 配置

```
nv set nve vxlan arp-nd-suppress enabled        ← 通过 EVPN 学 ARP，不广播
nv set nve vxlan mlag shared-address 10.0.1.12  ← MLAG 共用 VTEP IP（anycast）
nv set nve vxlan source address 10.10.10.1      ← leaf01 独立 VTEP IP
nv set nve vxlan state enabled
```

**两个 VTEP IP 的关系**：
- `source address 10.10.10.1`：本机独立 VTEP（用于通告 EVPN 路由）
- `shared-address 10.0.1.12`：MLAG 共用 anycast（远端 leaf 把这个作为目的地）

### 10.11 BFD（快速故障检测）

```
nv set router bfd profile bgp-underlay-bfd detect-multiplier 3
nv set router bfd profile bgp-underlay-bfd min-rx-interval 300
nv set router bfd profile bgp-underlay-bfd min-tx-interval 300
nv set router bfd state enabled
```

- 300ms × 3 = 900ms 内检测到链路故障
- 远比 BGP 默认 hold timer（90s）快

### 10.12 BGP Graceful Restart

```
nv set router bgp graceful-restart mode helper-only
nv set router bgp state enabled
```

- `helper-only`：本设备帮邻居做 GR（保留邻居路由），但自己 crash 时不依赖 GR
- 适合 leaf 这种"边缘"角色

### 10.13 VRR 全局开关

```
nv set router vrr state enabled
```

### 10.14 控制平面 ACL（重要！）

```
nv set system control-plane acl acl-default-dos inbound
nv set system control-plane acl acl-default-whitelist inbound
```

- 默认开启 Cumulus 提供的 control-plane 保护 ACL
- 这正是 SONiC 讨论里"删错 ACL 断 BGP"的对应保护机制

### 10.15 NTP / DNS / Syslog / SNMP（全部在 mgmt VRF）

```
nv set system ntp server 0.cumulusnetworks.pool.ntp.org iburst enabled
nv set system ntp vrf mgmt

nv set system dns server 1.1.1.1 vrf mgmt
nv set system dns server 8.8.8.8 vrf mgmt

nv set system syslog server 192.168.200.1 vrf mgmt

nv set system snmp-server listening-address all vrf mgmt
nv set system snmp-server readonly-community '$nvsec$...' access any
nv set system snmp-server system-location evpn_symmetric
```

**模式**：管理类服务全部走 mgmt VRF，与业务流量完全隔离。

### 10.16 What Just Happened (WJH) — NVIDIA 独有

```
nv set system wjh channel forwarding trigger l2
nv set system wjh channel forwarding trigger l3
nv set system wjh channel forwarding trigger tunnel
nv set system wjh state enabled
```

- **WJH**：NVIDIA 独有的实时丢包诊断工具
- 直接告诉你"哪个包被谁因为什么丢了"（L2/L3/tunnel 维度）
- 对 RoCE / AI 网络排障极有价值

### 10.17 自动保存

```
nv set system config auto-save state enabled
```

### 10.18 BGP 在三个 VRF 中的配置

#### 10.18.1 Default VRF（underlay）

```
nv set vrf default router bgp autonomous-system 65101
nv set vrf default router bgp router-id 10.10.10.1

# 启用两个地址族
nv set vrf default router bgp address-family ipv4-unicast state enabled
nv set vrf default router bgp address-family ipv4-unicast redistribute connected state enabled
nv set vrf default router bgp address-family l2vpn-evpn state enabled

# 邻居（使用 peer-group "underlay"）
nv set vrf default router bgp neighbor swp51 peer-group underlay
nv set vrf default router bgp neighbor swp51 type unnumbered
nv set vrf default router bgp neighbor swp52 peer-group underlay
nv set vrf default router bgp neighbor swp52 type unnumbered
nv set vrf default router bgp neighbor swp53 peer-group underlay
nv set vrf default router bgp neighbor swp53 type unnumbered
nv set vrf default router bgp neighbor swp54 peer-group underlay
nv set vrf default router bgp neighbor swp54 type unnumbered
nv set vrf default router bgp neighbor peerlink.4094 peer-group underlay
nv set vrf default router bgp neighbor peerlink.4094 type unnumbered

# Peer-group 通用属性
nv set vrf default router bgp peer-group underlay remote-as external
nv set vrf default router bgp peer-group underlay address-family l2vpn-evpn state enabled
nv set vrf default router bgp peer-group underlay bfd profile bgp-underlay-bfd
```

**亮点**：
- **peer-group**：5 个邻居共享同一组参数（remote-as external、EVPN 地址族、BFD）
- **`remote-as external`**：自动识别为 eBGP（不需指定具体 AS 号）
- **MLAG peer 也用 unnumbered**：通过 `peerlink.4094` 接口名

#### 10.18.2 VRF RED / BLUE（overlay）

```
nv set vrf RED evpn state enabled
nv set vrf RED evpn vni 4001                  ← RED 的 L3VNI
nv set vrf RED router bgp autonomous-system 65101
nv set vrf RED router bgp router-id 10.10.10.1
nv set vrf RED router bgp address-family ipv4-unicast state enabled
nv set vrf RED router bgp address-family ipv4-unicast redistribute connected state enabled
nv set vrf RED router bgp address-family l2vpn-evpn state enabled

nv set vrf BLUE evpn vni 4002                 ← BLUE 的 L3VNI
# ... 同样的 BGP 配置
```

**关键点**：
- 每个租户 VRF **独立运行 BGP 实例**
- L3VNI 是租户隔离的核心：RED=4001, BLUE=4002
- `redistribute connected`：把 VRF 内的直连路由（SVI 子网）通告出去
- `l2vpn-evpn enabled`：在该 VRF 内启用 EVPN 通告

#### 10.18.3 Mgmt VRF（静态默认路由）

```
nv set vrf mgmt router static 0.0.0.0/0 address-family ipv4-unicast
nv set vrf mgmt router static 0.0.0.0/0 via 192.168.200.1 type ipv4-address
```

- 管理网走静态默认路由（不参与 BGP）

### 10.19 AAA / 用户管理

```
nv set system aaa class nvapply action allow
nv set system aaa class nvapply command-path / permission all
nv set system aaa class nvshow action allow
nv set system aaa class nvshow command-path / permission ro
nv set system aaa class sudo action allow
nv set system aaa class sudo command-path / permission all

nv set system aaa role nvue-admin class nvapply
nv set system aaa role nvue-monitor class nvshow
nv set system aaa role system-admin class nvapply
nv set system aaa role system-admin class sudo

nv set system aaa user cumulus role system-admin
```

- **三类权限**：`nvapply`（写配置）、`nvshow`（只读）、`sudo`（系统 root）
- **三种角色**：`nvue-admin`、`nvue-monitor`、`system-admin`
- `cumulus` 用户拥有完整 `system-admin` 角色

### 10.20 全局任播 MAC（EVPN 关键）

```
nv set system global anycast-mac 44:38:39:BE:EF:12
```

- 整个 fabric 内**所有 leaf** 都用这个 MAC 作为"任播路由器 MAC"
- 配合 VRR 实现统一网关：server 无论接哪个 leaf，看到的网关 MAC 都一样

---

## 11. 验证命令速查

### 11.1 配置管理

```bash
nv config show                  # 看当前配置（YAML 格式）
nv config show -o commands      # 看配置（CLI 命令格式）
nv config diff                  # 待提交配置 vs 当前配置
nv config apply                 # 应用配置
nv config save                  # 保存到启动配置
sudo cat /etc/nvue.d/startup.yaml  # 看启动配置
```

### 11.2 Bridge / VLAN

```bash
nv show bridge domain br_default
nv show bridge domain br_default mac-table
```

### 11.3 MLAG

```bash
nv show mlag                                    # MLAG 总体状态
nv show interface --view=mlag-cc                # MLAG 接口一致性
nv show mlag consistency-checker global         # MLAG 全局一致性
sudo clagctl status                             # 老命令（仍可用）
cat /var/log/clagd.log                          # MLAG 日志
```

### 11.4 VXLAN / EVPN

```bash
nv show nve vxlan                               # VTEP 状态
nv show evpn vni                                # 所有 VNI
nv show evpn vni 10                             # 具体某 VNI
nv show evpn vni 10 mac                         # VNI 内 MAC 表
sudo vtysh -c 'show bgp l2vpn evpn vni'         # EVPN VNI 详情
sudo vtysh -c 'show bgp l2vpn evpn route type 2'  # MAC/IP 路由 (Type-2)
sudo vtysh -c 'show bgp l2vpn evpn route type 3'  # IMET 路由 (Type-3)
sudo vtysh -c 'show bgp l2vpn evpn route type 5'  # IP Prefix 路由 (Type-5)
```

### 11.5 BGP

```bash
sudo vtysh -c 'show bgp sum'                    # BGP 邻居摘要
sudo vtysh -c 'show bgp neighbor swp51'         # 具体邻居详情
nv show vrf default router bgp neighbor         # NVUE 等价命令
sudo vtysh -c 'show ip route vrf RED'           # VRF 内的 IPv4 路由表
sudo vtysh -c 'show ip route vrf BLUE'
sudo vtysh -c 'show ip route'                   # default VRF 的路由表
```

### 11.6 接口 / ARP

```bash
ip link show swp51                              # 接口状态
ip -6 addr show swp51                           # IPv6 LLA（unnumbered 用）
arp -n                                          # ARP 表
nv show interface                               # NVUE 接口总览
```

### 11.7 Lab 中 server 侧测试

```bash
# 同 VRF 同 VLAN（L2）
ping 10.1.10.104    # server01 → server04

# 同 VRF 跨 VLAN（L3 通过 EVPN）
ping 10.1.20.102    # server01 → server02
ping 10.1.20.105    # server01 → server05

# 跨 VRF（应该不通）
ping 10.1.30.103    # server01 → server03 (BLUE)
```

### 11.8 SNMP / Syslog（在 oob-mgmt-server）

```bash
snmpget -v2c -c public 192.168.200.6 iso.3.6.1.2.1.1.5.0
snmpwalk -v2c -c public 192.168.200.6 | head
sudo cat /var/log/syslog | grep 192.168.200.6
```

### 11.9 Ansible 自动化

```bash
cd Cumulus-Linux-demo/
git status
git checkout evpn_demo_nvue_5.16
ansible pod1 -i inventories/evpn_symmetric/hosts -m ping
ansible-playbook playbooks/nvue.yml -i inventories/evpn_symmetric/hosts
```

---

## 12. 与 AI 训练网络的关系

### 12.1 本 Lab 是"通用企业 DC 设计"

不是"AI 训练专用设计"。AI 集群（DGX SuperPOD）**会简化掉**很多东西。

| AI 集群 **不需要** | 本 Lab 包含 |
|------------------|-------------|
| MLAG（GPU 不双归） | ✓ MLAG |
| VXLAN（GPU 间直接 L3） | ✓ VXLAN |
| EVPN（不需要 L2 扩展） | ✓ EVPN |
| VRR（GPU 不需网关冗余） | ✓ VRR |
| 多 VRF（GPU 共享 fabric） | ✓ 多 VRF |

| AI 集群 **需要** | 本 Lab 暂未演示 |
|----------------|----------------|
| RoCEv2 (PFC/ECN) | ⚠ 未涉及 |
| Rail-Optimized 拓扑 | ⚠ 通用 Spine-Leaf |
| Adaptive Routing | ⚠ 未涉及 |
| Spectrum-X 拥塞控制 | ⚠ 未涉及 |

### 12.2 学习路径

1. **先打牢通用 DC 基础**（本 Lab）→ 理解 Cumulus、NVUE、EVPN、MLAG
2. **再迁移到 AI 网络** → 学 RoCE、Rail-Optimized、Spectrum-X
3. **二者的共性**：都用 Cumulus Linux + NVUE + BGP underlay

### 12.3 关键迁移点

- AI 场景下 `nv set qos roce` 启用 RoCE 模板
- AI 场景下用 `nv set interface swp* link breakout` 配置 800G→4×200G 拆分
- AI 场景下用 `nv set router adaptive-routing` 启用自适应路由（仅 Spectrum-X）
- 拓扑变成 Rail-Optimized：每个 leaf 只连同一 Rank 的 GPU

---

## 附录 A：本笔记涵盖的 NCP-AIN 知识点

```
✓ Cumulus Linux 基础
✓ NVUE 对象模型和 CLI（声明式配置）
✓ BGP unnumbered + peer-group
✓ MLAG 配置和验证（peerlink, clag-id, anycast-mac）
✓ VXLAN Active-Active（shared VTEP）
✓ EVPN 对称路由（L2VNI / L3VNI）
✓ VRR 任播网关（vs VRRP）
✓ 多租户 VRF（RED / BLUE / default / mgmt）
✓ BFD（300ms × 3 = 900ms 检测）
✓ Graceful Restart（helper-only）
✓ Control-plane ACL（默认保护）
✓ SNMP / Syslog / NTP / DNS（mgmt VRF）
✓ What Just Happened (WJH) 实时丢包诊断
✓ Ansible 自动化（基于 git branch）
✓ Git 化配置管理
```

## 附录 B：相关笔记交叉引用

- 通用 Leaf-Spine 架构：`clos-architecture.md`
- Rail-Optimized 拓扑：`Rail-Optimized_vs_Fat-Tree.md`, `Rail-optimized-network.md`
- RoCE 拥塞控制：`RoCE_vs_SpectrumX_Congestion_Control.md`
- RoCE Lab：`RoCEv2-PFC-lab.md`, `RoCEv2-ECN-lab.md`
- DGX SuperPOD：`DGX_SuperPOD_B300_vs_A100.md`
- EVPN 通用笔记：`EVPN.md`, `EVPN_formatted.md`

---

**笔记结束**。下一步可以做：
1. 在 lab 上实际跑 `nv show evpn vni` 等命令验证
2. 抓包看 VXLAN 实际封装（在 spine 上 tcpdump）
3. 故障注入实验（shutdown 上行口看 BGP/EVPN 收敛）
