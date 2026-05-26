# Cumulus Lab: EVPN Multi-Homing + PIM Multicast 详细笔记

> **来源**：NVIDIA Air Lab `evpn-multi-homing` + `evpn-mh-multicast`
> **学习日期**：2026-05-21
> **关联考试**：NCP-AIN
> **前置知识**：EVPN Symmetric Routing（见 `Cumulus_Lab_EVPN_Symmetric.md`）

## 目录

1. [Lab 拓扑全貌](#1-lab-拓扑全貌)
2. [EVPN-MH 核心机制](#2-evpn-mh-核心机制)
3. [Type-1 路由深度分析](#3-type-1-路由深度分析)
4. [Type-4 路由与 DF Election](#4-type-4-路由与-df-election)
5. [PIM-SM 多播详解](#5-pim-sm-多播详解)
6. [BUM 流量完整数据流](#6-bum-流量完整数据流)
7. [命令输出逐行解析](#7-命令输出逐行解析)
8. [NVUE 配置详解](#8-nvue-配置详解)
9. [Lab 1 vs Lab 2 对比](#9-lab-1-vs-lab-2-对比)
10. [NCP-AIN 考点总结](#10-ncp-ain-考点总结)
11. [易混淆概念速查](#11-易混淆概念速查)

---

## 1. Lab 拓扑全貌

### 1.1 设备清单

```
┌──────────────────────────────────────────────────┐
│                    Spine 层（4台）                 │
│  spine01  spine02  spine03  spine04              │
│   AS 65100（全部同AS，纯转发）                     │
└──────────┬──────────┬──────────┬──────────┬──────┘
           │          │          │          │
    ┌──────┴──┐  ┌────┴───┐  ┌──┴────┐  ┌──┴──────┐
    │ leaf01  │  │ leaf02 │  │ leaf03│  │  leaf04 │
    │ AS65101 │  │AS65102 │  │AS65103│  │ AS65104 │
    └────┬────┘  └────┬───┘  └───┬───┘  └────┬────┘
         │ ESI aa     │           │ ESI bb    │
         │ (共享)     │           │ (共享)    │
         ▼            ▼           ▼           ▼
    ┌────────────────────┐   ┌────────────────────┐
    │ server01/02/03     │   │ server04/05/06     │
    │ (LACP bond)        │   │ (LACP bond)        │
    └────────────────────┘   └────────────────────┘

    ┌──────────────────┐
    │ border01/02       │ ← PIM RP (Lab 2 关键)
    │ AS 65199          │
    │ MSDP mesh-group   │ ← 双 RP 同步
    └──────────────────┘
         │
    ┌────┴────┐
    │ srv07/08│ ← 预留，无配置
    │ fw1/fw2 │
    └─────────┘
```

### 1.2 VRF + VLAN + 多播组分配（关键表）

| VLAN | VNI  | VRF  | Multicast Group | 服务器                          |
|------|------|------|-----------------|--------------------------------|
| 10   | 10   | RED  | 224.1.1.10      | server01 (leaf01/02), server04 (leaf03/04) |
| 20   | 20   | RED  | 224.1.1.20      | server02 (leaf01/02), server05 (leaf03/04) |
| 30   | 30   | BLUE | 224.1.1.30      | server03 (leaf01/02), server06 (leaf03/04) |
| L3VNI 4001 | -    | RED  | -               | （Symmetric Routing transit）  |
| L3VNI 4002 | -    | BLUE | -               | （Symmetric Routing transit）  |

**关键观察：**
- 2 个 VRF (RED/BLUE) 实现 multi-tenancy
- 每个 L2VNI 一对一映射到一个多播组（优化原则）
- L3VNI 没有多播组（L3VNI 只承载路由流量，不是 BUM）

### 1.3 IP 规划

```
Loopback（VTEP）：
  leaf01 = 10.10.10.1
  leaf02 = 10.10.10.2
  leaf03 = 10.10.10.3
  leaf04 = 10.10.10.4
  border01 = 10.10.10.63 (PIM 用额外 loopback 10.10.100.100 作为 Anycast RP)
  border02 = 10.10.10.64 (同上)

Anycast Gateway（VRR）：
  VLAN 10: 10.1.10.1 (所有 leaf 共享)
  VLAN 20: 10.1.20.1
  VLAN 30: 10.1.30.1
  → 任何 leaf 都能作为本地默认网关

每 leaf 的 SVI 唯一 IP（用于 Type-2 通告）：
  vlan10: leaf01=10.1.10.2, leaf02=10.1.10.3, leaf03=10.1.10.4, leaf04=10.1.10.5
```

### 1.4 ESI 共享关系

```
ESI 命名规律：03:44:38:39:be:ef:<site>:00:00:<bond-id>
                ↑                ↑              ↑
              Type-03            站点标识       本地ID
             (manual)         aa或bb         1/2/3

leaf01 + leaf02 共享：
  bond1 → 03:44:38:39:be:ef:aa:00:00:01  (VLAN 10)
  bond2 → 03:44:38:39:be:ef:aa:00:00:02  (VLAN 20)
  bond3 → 03:44:38:39:be:ef:aa:00:00:03  (VLAN 30)

leaf03 + leaf04 共享：
  bond1 → 03:44:38:39:be:ef:bb:00:00:01
  bond2 → 03:44:38:39:be:ef:bb:00:00:02
  bond3 → 03:44:38:39:be:ef:bb:00:00:03
```

---

## 2. EVPN-MH 核心机制

### 2.1 EVPN-MH 替代 MLAG 的 4 大优势

```
1. 无 peerlink（ISL）        ← 节省端口和带宽
2. 可超过 2 台 ToR           ← 真正的 multi-active
3. 单一 BGP-EVPN 控制平面    ← 不需要专有 MLAG 协议
4. 多厂商互通                ← 标准化 RFC 7432
```

### 2.2 Ethernet Segment (ES) 概念

```
ES (Ethernet Segment) 的定义：
  "连接同一个客户设备的一组链路"
  
  实际上 = 一组 leaf + 一个客户设备 + 它们之间的所有链路

Lab 里的情况：
  bond1 ES → server01（1个 server = 1个 ES）
  bond2 ES → server02
  bond3 ES → server03

更一般情况：
  - server01 有 4 个网口分两组 → 可能形成 2 个 ES
  - 客户接的是一台交换机（不是 server）→ 那台交换机就是一个 ES 的"客户设备"
```

### 2.3 "bond" 在不同视角下的含义

```
                  server01
                ┌─────────┐
                │ eth1 eth2 │  ← server 侧的 LACP bond
                └──┬───┬──┘
                   │   │
              swp1 │   │ swp1
                ┌──▼──┐ │ ┌──▼──┐
                │leaf01│ │ │leaf02│
                │bond1 │ │ │bond1 │   ← leaf 侧的 bond
                └──────┘ │ └──────┘
                         │
                  ──同一个ES──
                  ESI=aa:01

  从 leaf01 视角：
    bond1 = leaf01 上把 swp1 包装成一个 LAG 接口
    
  从 server01 视角：
    server01 把 eth1+eth2 绑成 LACP bond
    
  从 EVPN-MH 视角：
    leaf01 bond1 + leaf02 bond1 = 同一个 ES（共享 ESI）
```

### 2.4 ESI 格式（10 字节）

```
ESI = Type(1B) + ESI-value(9B)

Type 03（manual config）：
  03:<MAC:6B>:<discriminator:3B>
  例：03:44:38:39:BE:EF:AA:00:00:01

→ 同一个 ES 的多台 leaf 配置：
  - 相同的 mac-address (44:38:39:BE:EF:AA)
  - 相同的 local-id (1)
  → 自动形成相同 ESI → 加入同一个 ES
```

### 2.5 Split-Horizon 防回环机制

```
作用：防止 BUM 在 ES 成员间循环

机制：
  1. 入口 leaf 封装 VXLAN BUM 时携带 ESI-Label
  2. 远端 ES 成员 leaf 检查 ESI-Label
  3. 如果 label 匹配自己的 ES → 丢弃
  4. 否则 → 正常转发

示例（leaf01 发 BUM）：
  leaf01 ──VXLAN BUM (ESI-Label=AA)──→ leaf02 (也是 ESI aa 成员)
                                          ↓ 检查 ESI-Label
                                          ↓ Label=AA，我也是 aa
                                          → 丢弃 ✗ (防回环)
  
  leaf01 ──VXLAN BUM (ESI-Label=AA)──→ leaf03 (是 ESI bb 成员)
                                          ↓ 检查 ESI-Label
                                          ↓ Label=AA，我是 bb
                                          → 正常转发 ✓
```

---

## 3. Type-1 路由深度分析

### 3.1 Type-1 路由的两大功能

```
┌─────────────────────────────────────────────┐
│  功能1：快速收敛                              │
│  - ES down 时，撤一条路由批量清远端 MAC      │
│  - 比等 Type-2 MAC aging 快 10-50 倍         │
│                                              │
│  功能2：多活负载均衡 (Aliasing)               │
│  - 让远端知道一个 MAC 有多个 VTEP 可达        │
│  - ECMP 流量分担到所有 ES 成员                │
└─────────────────────────────────────────────┘
```

### 3.2 EAD-per-ES vs EAD-per-EVI

```
两种 EAD（Ethernet Auto-Discovery）路由：

EAD-per-ES：
  EthTag = 0xFFFFFFFF (4294967295)
  携带 ESI-Label-Rt（Split-Horizon 标签）
  每个 leaf 每个 ES 一条
  用途：快速收敛 + 标识 Split-Horizon Label

EAD-per-EVI：
  EthTag = 0
  不携带 ESI-Label
  每个 leaf 每个 (ES × VNI) 一条
  用途：Aliasing（多活 ECMP）
```

### 3.3 Type-1 路由格式

```
[1]:[EthTag]:[ESI]:[IPlen]:[VTEP-IP]:[Frag-id]

本地路由：
*> [1]:[0]:[03:44:38:39:be:ef:aa:00:00:02]:[128]:[::]:[0] RD 10.10.10.1:2
                    10.10.10.1 (leaf01)
                                                       32768 i
                    ET:8 RT:65101:20

远端路由（经过 spine 学到）：
*> [1]:[0]:[03:44:38:39:be:ef:aa:00:00:02]:[32]:[0.0.0.0]:[0] RD 10.10.10.2:3
                    10.10.10.2 (spine01)
                                                           0 65100 65102 i
                    RT:65102:20 ET:8

字段对比：
  - 本地：IPlen=128, VTEP-IP=::, Weight=32768, Path=i
  - 远端：IPlen=32,  VTEP-IP=0.0.0.0, Weight=0, Path=0 65100 65102 i
  - 本地 Next Hop = 自己 loopback
  - 远端 Next Hop = 发路由的 leaf 的 loopback（不是 spine）
```

### 3.4 Lab 1 输出统计：78 paths / 24 prefixes

```
为什么是 24 个 prefix？
  - leaf01 本地：3 个 ESI × 2 种 EAD = 6 条
  - leaf02 远端：6 条
  - leaf03 远端：6 条
  - leaf04 远端：6 条
  → 24 个唯一 prefix ✓

为什么是 78 paths？
  - leaf01 本地：6 条 × 1 路径 = 6
  - leaf02 远端：6 条 × 4 路径（4 个 spine 各转发一份）= 24
  - leaf03 远端：6 条 × 4 路径 = 24
  - leaf04 远端：6 条 × 4 路径 = 24
  → 6 + 24 + 24 + 24 = 78 ✓
```

### 3.5 为什么 leaf02/03/04 路由比 leaf01 多？

```
你站在 leaf01 视角看：

  自己发出的路由（本地）：
    Next Hop = 10.10.10.1 (leaf01 自己)
    → 只显示 1 条（本地只有 1 条最优路径，没有多路径）
    → 6 条路由，每条只显示 1 行

  从远端学来的路由（leaf02/03/04 发的）：
    Next Hop = 10.10.10.2/3/4 (远端 leaf 的 loopback)
    → 通过 4 个 spine 各学到 1 份 = 4 条路径
    → 6 条路由 × 4 条路径 = 每个 RD 下显示 4 行

⚠️ 重要：Next Hop 永远是发路由的 leaf 的 loopback
   括号里的 spine01/02/03/04 只是告诉你这条路由是通过哪个 BGP 邻居学到的
```

### 3.6 Lab 1 vs Lab 2 的 Type-1 数量差异

```
Lab 1 (HER)：78 paths, 24 prefixes
Lab 2 (PIM)：39 paths, 12 prefixes

为什么 Lab 2 少？
  → 启用 PIM 后，部分 BUM 处理交给多播
  → 不需要那么多 EAD-per-EVI 用于 HER 复制列表
  → Type-3 (IMET) 也变化了
```

### 3.7 实用结论：是否需要严格区分两种 EAD？

```
直接结论：不用死记。

考试和工作中只需要记住：
  Type-1 (EAD) 路由的两大功能：
    - 快速收敛
    - Aliasing 多活
  
  → 这两个功能由两种 EAD 路由分工完成
  → 你不需要记谁干啥
  → 能区分是加分项，记不住完全不影响考试和工作
```

---

## 4. Type-4 路由与 DF Election

### 4.1 为什么需要 DF (Designated Forwarder)?

```
问题场景：BUM 流量（Broadcast/Unknown unicast/Multicast）

  远端 leaf03 收到一个广播帧，要发给 ESI aa:01 这个段
  → 它知道 leaf01 和 leaf02 都连着这个 ES
  → 如果两台都收到并都转发给 server01
  → server01 收到 2 份重复帧 ❌

解决方案：选一个 DF (Designated Forwarder)
  → 同一个 ES 上，只有 DF 转发 BUM 到本地 server
  → 非 DF（NDF）丢弃 BUM
```

```
ASCII 图示：

         远端 leaf03 (BUM 源)
              │
        ┌─────┴─────┐
        ▼           ▼
     leaf01      leaf02
     (DF)        (NDF)
        │           │
        ▼           ✗ 丢弃
     server01
     (只收 1 份)
```

### 4.2 Type-4 路由格式

```
格式：[4]:[ESI]:[IPlen]:[OrigIP]

示例：
*> [4]:[03:44:38:39:be:ef:aa:00:00:01]:[32]:[10.10.10.1]
                                            ↑
                                       leaf01 的 loopback

  携带的扩展团体：
    ES-Import-RT  ← 关键：只有同 ES 的成员才 import 这条路由
    DF-Election   ← 携带 df-preference 值
```

**Type-4 的唯一作用：让同 ES 的 leaf 互相发现，然后选 DF。**

### 4.3 DF Election 算法（Cumulus Preference-based）

```
步骤：
  1. 每台 ES 成员发 Type-4 路由，携带自己的 df-preference
  2. 所有成员收齐后，按以下规则排序：
     a) df-preference 高的赢
     b) 相等时，loopback IP 低的赢（tiebreaker）
  3. 每个 EVI (VNI) 独立选举 DF
     → 不同 VNI 可以选不同的 DF（负载分担）
```

**Lab 配置示例：**

```bash
# leaf01 和 leaf02 都设了 df-preference 50000
nv set interface bond1 evpn multihoming df-preference 50000

# 默认值是 32767，谁高谁是 DF
# 如果两台都设 50000 → loopback IP 小的赢
#   leaf01 (10.10.10.1) < leaf02 (10.10.10.2)
#   → leaf01 是 DF
```

### 4.4 故障切换场景

```
场景：leaf01 宕机

T+0    leaf01 down
       
T+1ms  underlay BFD 触发 BGP session 断开
       
T+10ms leaf02 收到 spine 撤销 leaf01 的所有 Type-1/2/3/4 路由
       → leaf02 重新选 DF for aa:01/02/03
       → leaf02 自己赢（唯一成员）
       → leaf02 现在是 DF
       
T+50ms 远端 leaf03/04 也收到撤销
       → 通过 Type-1 EAD-per-ES 一次性清除 leaf01 学到的所有 MAC
       → ECMP 路径自动收敛到 leaf02
       
T+100ms server01/02/03 流量完全切到 leaf02

总收敛时间：< 1 秒
对比 MLAG：通常 1-5 秒（依赖 peerlink 和 MAC 重学习）
```

---

## 5. PIM-SM 多播详解

### 5.1 为什么 VXLAN BUM 需要多播？

```
两种 BUM 复制方式：

方式 1：HER (Head-End Replication) ← Lab 1 用的
  入口 VTEP 收到 BUM 后，复制 N 份单播 VXLAN 发给 N 个远端 VTEP
  
  10 个 VTEP → 入口 leaf 发 9 份重复包
  100 个 VTEP → 入口 leaf 发 99 份重复包 ❌ 带宽爆炸
  
方式 2：PIM-SM 多播 ← Lab 2 用的
  入口 VTEP 只发 1 份多播 VXLAN
  Spine 帮忙复制分发
  
  100 个 VTEP → 入口 leaf 只发 1 份 ✓ 节省 99% 带宽
```

### 5.2 PIM-SM 三阶段流程

```
阶段 1：Join（接收方注册）
  ┌──────────────┐
  │ Receiver     │ 发 IGMP Join 224.1.1.10
  │ (server04)   │
  └──────┬───────┘
         │ IGMP
         ▼
  ┌──────────────┐
  │ leaf03 (DR)  │ → 发 PIM (*,G) Join 朝 RP
  └──────┬───────┘
         │ PIM Join
         ▼
  ┌──────────────┐
  │ spine        │ → 转发 Join
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ border (RP)  │ → 建立 (*, 224.1.1.10) state
  └──────────────┘
       构建 RPT (Shared Tree)

阶段 2：Register（发送方注册）
  ┌──────────────┐
  │ Sender       │ 发多播包 to 224.1.1.10
  │ (server01)   │
  └──────┬───────┘
         │
         ▼
  ┌──────────────┐
  │ leaf01 (DR)  │ 不知道接收方在哪里
  │              │ → 把包封装成 PIM Register
  │              │ → 单播发给 RP
  └──────┬───────┘
         │ PIM Register (unicast)
         ▼
  ┌──────────────┐
  │ border (RP)  │ 解封装 → 沿 RPT 转发
  │              │ → 同时发 (S,G) Join 朝 source
  └──────┬───────┘

阶段 3：SPT Switchover（最短路径切换）
  ┌──────────────┐
  │ leaf01       │ 收到 (S,G) Join
  │              │ → 不再封装 Register
  │              │ → 直接发原生多播包
  └──────┬───────┘
         │ Native multicast
         ▼
  ┌──────────────┐
  │ border (RP)  │ 发 Register-Stop 给 leaf01
  └──────────────┘
  
  之后接收方端 leaf03 也会发 (S,G) Join，
  把流量切到 Source-Tree（绕过 RP，最短路径）
```

### 5.3 RPT vs SPT

```
RPT (Rendezvous Point Tree / Shared Tree)：
  - 状态：(*, G)
  - 路径：所有 Source → RP → 所有 Receiver
  - 优点：状态少（每组只有一棵树）
  - 缺点：路径不最优（必须经过 RP）

SPT (Shortest Path Tree / Source Tree)：
  - 状态：(S, G)
  - 路径：Source → 直接 → Receiver（最短路径）
  - 优点：路径最优
  - 缺点：每个 (Source, Group) 一棵树，状态多

PIM-SM 默认行为：
  初始用 RPT → 流量达阈值（默认第一个包）→ 切到 SPT
```

### 5.4 RP 的角色

```
RP (Rendezvous Point) = 多播汇合点

作用：
  1. Source 不知道 Receiver 在哪 → 把包发给 RP
  2. Receiver 不知道 Source 在哪 → 朝 RP 发 Join
  3. RP 充当媒人，撮合两边
  
RP 位置选择：
  - 网络中心位置（流量经过最优）
  - 通常放在 spine 或 border
  - Lab 2 放在 border01/02

Anycast RP（高可用）：
  - 两台 border 共享同一个 RP IP（10.10.100.100）
  - 任何 leaf 朝 RP 发 Join，被最近的 border 接收
  - 需要 MSDP 同步两台 RP 的 source 信息
```

### 5.5 Anycast RP + MSDP 工作机制

```
        ┌──────────────┐         ┌──────────────┐
        │ border01     │◄────────►│ border02     │
        │ lo: 10.10.10.63        │ lo: 10.10.10.64
        │ lo2: 10.10.100.100     │ lo2: 10.10.100.100
        │  (Anycast RP)          │  (Anycast RP)
        │              │         │              │
        │ MSDP peer ──────────────│ MSDP peer    │
        │ mesh-group   │ rpmesh  │              │
        └──────┬───────┘         └──────┬───────┘
               │                        │
               ▼                        ▼
        leaf01 朝 10.10.100.100      leaf03 朝 10.10.100.100
        发 Register（source 信息）   发 Register
        → border01 接收              → border02 接收

        ↓ MSDP SA (Source Active) 消息 ↓
        border01 通过 MSDP 告诉 border02：
          "我这里有 source 10.1.10.X 在发 224.1.1.10"
        border02 知道后，可以为它的客户端构建 (S,G) Join

关键：
  - MSDP 只同步 source 信息，不同步流量
  - 防止 Anycast RP 因为知识不对称导致丢流量
```

### 5.6 DR (Designated Router) 概念

```
DR ≠ DF（容易混淆！）

DR (PIM Designated Router)：
  - PIM 协议里的概念
  - 一个 LAN 段上选一个 PIM 路由器代表
  - 负责：发 Register（source 侧）/ 发 IGMP Join 翻译（receiver 侧）
  
DF (EVPN Designated Forwarder)：
  - EVPN-MH 里的概念
  - 一个 ES 上选一个 leaf 代表
  - 负责：转发 BUM 到本地 server

在 Lab 2 里：
  leaf01 既是 DF（for ESI aa:01）
        又可能是 DR（for 它的 SVI vlan10）
  → 不要混！
```

---

## 6. BUM 流量完整数据流

### 6.1 场景设定

```
源：server01 (连 leaf01/02 via bond1, ESI aa:01)
目的：广播帧 dst MAC = ff:ff:ff:ff:ff:ff (e.g. ARP request)
VLAN: 10 → VNI 10 → Multicast Group 224.1.1.10
RP: 10.10.100.100 (Anycast，border01/02)

接收方：server04 (连 leaf03/04 via bond1, ESI bb:01)
```

### 6.2 完整 10 步数据流

```
Step 1: server01 发广播帧
  ┌────────────┐
  │ server01   │ ARP Req
  └──────┬─────┘
         │ Eth: dst=ff:ff:ff:ff:ff:ff, vlan=10
         ▼
  bond1 (LACP hash) → 50% 走 leaf01, 50% 走 leaf02
  假设走 leaf01

Step 2: leaf01 收到，判断 DF 角色
  leaf01 检查 ESI aa:01 在 VNI 10 的 DF：
    leaf01 是 DF ✓ → 继续转发
    (如果不是 DF → 丢弃)

Step 3: leaf01 查 VNI 10 的多播组
  bridge domain vlan10:
    vni = 10
    mcast-group = 224.1.1.10
  
Step 4: leaf01 封装 VXLAN 多播包
  Outer IP:
    src = 10.10.10.1 (leaf01 loopback)
    dst = 224.1.1.10 (多播组)
  VXLAN:
    VNI = 10
    ESI-Label = AA (Split-Horizon 用)
  Inner: 原始广播帧

Step 5: leaf01 PIM 行为
  if leaf01 已经有 (S=10.10.10.1, G=224.1.1.10) 状态：
    直接发原生多播 ↓
  else:
    封装为 PIM Register，单播发给 RP (10.10.100.100)
    → border01 解封装，沿 RPT 转发

Step 6: 多播包到达 spine
  spine 根据 (*, 224.1.1.10) 或 (S, G) 状态复制
  → 转发给所有有兴趣的下游 leaf
  
  关键：spine 只复制一份给每条下游链路
  → 一份包变 4 份（如果 4 个 leaf 都加入了 224.1.1.10）

Step 7: leaf02 收到（同 ES 成员）
  检查 VXLAN ESI-Label = AA
  leaf02 也是 ESI aa 成员 → Split-Horizon 丢弃 ✓
  (防止 server01 收到自己发的包)

Step 8: leaf03 收到
  检查 VXLAN ESI-Label = AA
  leaf03 是 ESI bb 成员（不匹配）→ 正常处理

Step 9: leaf03 检查 DF 角色
  leaf03 检查自己是否是 ESI bb:01 在 VNI 10 的 DF
    是 DF → 继续 ↓
    不是 DF → 丢弃（让 leaf04 转发）
  
  假设 leaf03 是 DF

Step 10: leaf03 解封装并转发到 server04
  剥掉 VXLAN header
  剥掉 ESI-Label
  恢复原始广播帧
  通过 bond1 发给 server04
  
  注意：leaf04 上的 bond1 不发（不是 DF）
  → server04 只收 1 份 ✓
```

### 6.3 关键检查点总结

```
入口（leaf01）：
  ✓ 是 DF 才转发
  ✓ 查 VNI → 多播组映射
  ✓ 加 ESI-Label

中间（spine）：
  ✓ PIM 状态决定复制
  ✓ Anycast RP 处理

出口（leaf03）：
  ✓ 检查 ESI-Label 做 Split-Horizon
  ✓ 是 DF 才转发到本地 server
```

---

## 7. 命令输出逐行解析

### 7.1 `show evpn es` —— ES 详情

```
ESI                            Type ES-IF      VTEPs
03:44:38:39:be:ef:aa:00:00:01  L    bond1      10.10.10.1,10.10.10.2
03:44:38:39:be:ef:aa:00:00:02  L    bond2      10.10.10.1,10.10.10.2
03:44:38:39:be:ef:bb:00:00:01  R               10.10.10.3,10.10.10.4

字段解释：
  Type:
    L = Local（本 leaf 上有 bond 属于这个 ES）
    R = Remote（远端 leaf 上的 ES）
    LR = Local + Remote（少见）
  
  ES-IF: 本地绑定的接口（只有 L 类型才有）
  VTEPs: 这个 ES 涉及的所有 VTEP loopback IP
```

### 7.2 `show evpn es <ESI> detail` —— ES Flags 解读

```
ESI: 03:44:38:39:be:ef:aa:00:00:01
  Type: Local Remote
  Interface: bond1
  Bridge: br_default
  State: Up
  Flags: lrf*bsA
  ...

Flags 字符含义（重要！）：
  l = Local
  r = Remote  
  f = Local active in forwarding
  *  = DF for at least one EVI
  b = Bypass mode disabled
  s = Has at least one ES-EVI in single-active
  A = Auto-discovered（通过 EVPN 学到的远端 ES）

常见组合：
  lrf*bsA = 本地+远端，本地活跃，至少一个 EVI 我是 DF，自动发现
  rA      = 纯远端，自动发现
```

### 7.3 `show evpn es-evi` —— ES × EVI 关系

```
VNI      ESI                            Flags
10       03:44:38:39:be:ef:aa:00:00:01  LR(EV)
20       03:44:38:39:be:ef:aa:00:00:02  LR(EV)
30       03:44:38:39:be:ef:aa:00:00:03  LR(EV)
10       03:44:38:39:be:ef:bb:00:00:01  R
20       03:44:38:39:be:ef:bb:00:00:02  R
30       03:44:38:39:be:ef:bb:00:00:03  R

字段解释：
  VNI: 这个 ES 在哪个 VNI 里
  Flags:
    L = Local
    R = Remote
    (EV) = 在这个 EVI（VNI）上有 EVPN 状态
  
  注意：一个 ESI 可以出现在多个 VNI 里
    bond1 是 access port for VLAN 10 → ES 出现在 VNI 10
    bond1 也可能 trunk VLAN 20 → ES 同时出现在 VNI 20
```

### 7.4 `show evpn es-vrf` —— L3 NHG 映射

```
ESI                            VRF       NHG       Installed
03:44:38:39:be:ef:bb:00:00:01  RED       1610612737  Yes
03:44:38:39:be:ef:bb:00:00:02  RED       1610612737  Yes
03:44:38:39:be:ef:bb:00:00:03  BLUE      1610612738  Yes

意义：
  - 对应远端 ES 创建一个 NHG (Next Hop Group)
  - NHG 里包含到这个 ES 所有成员 leaf 的路径
  - 实现 L3 Aliasing：去 server04 的流量 ECMP 到 leaf03 和 leaf04
  
  对于 ESI bb:01：
    NHG 包含：
      路径1：经 leaf03 (10.10.10.3) via L3VNI 4001
      路径2：经 leaf04 (10.10.10.4) via L3VNI 4001
```

### 7.5 `show ip pim state` —— PIM 状态

```
(10.1.10.10,224.1.1.10)
  Source: 10.1.10.10  Group: 224.1.1.10
  RPF nbr: 10.10.20.0
  RPF idx: swp51
  Upstream State: JOINED  
  IIF: swp51
  OIF Flags
  swp52  IF
  vlan10  IF

(*,224.1.1.10)
  Source: *  Group: 224.1.1.10
  RP: 10.10.100.100
  RPF nbr: 10.10.30.1
  ...

字段解释：
  IIF (Incoming Interface) = RPF 接口（流量从这里进来）
  OIF (Outgoing Interface) = 转发到这些接口
  RPF nbr = 上游邻居（朝 source 或 RP 的方向）
  Upstream State: JOINED = 已加入这棵树
```

### 7.6 `show ip mroute` —— 多播路由表

```
Source          Group           Proto  Input  Output      TTL  Uptime
10.1.10.10      224.1.1.10      PIM   swp51  swp52,vlan10  1   00:05:23
*               224.1.1.10      IGMP  swp51  vlan10        1   00:10:45

Flags 含义（mroute 上下文）：
  S = Sparse mode
  F = Source is registered (已注册到 RP)
  T = SPT-bit set（已经切到 SPT）
  
  SFT 全亮 = 完美状态（注册了 + 已切 SPT）
  SF (no T) = 还在用 RPT，没切到 SPT
```

### 7.7 `show ip pim upstream` —— 上游状态

```
Iif       Source          Group           State    Uptime    JoinTimer  RSTimer  KATimer
swp51     10.1.10.10      224.1.1.10      Joined   00:05:23  00:00:42   --:--:--  00:03:10
swp51     *               224.1.1.10      Joined   00:10:45  00:00:38   --:--:--  --:--:--

字段：
  Iif: RPF 接口
  State: 
    Joined = 已发 Join，上游在通流量
    NotJoined = 等待
    Pruned = 已剪除
  JoinTimer: 下次刷新 Join 倒计时（默认 60 秒）
  KATimer: Keepalive 倒计时（source 还在发 → 重置）
```

### 7.8 `show ip pim rp-info` —— RP 信息

```
RP address       Group/Prefix      Source       I am RP
10.10.100.100   224.0.0.0/4        Static       no

字段：
  RP address: 配置的 RP IP（Anycast）
  Source: Static / BSR / Auto-RP
  I am RP: 自己是不是 RP（border 上会显示 yes）
```

### 7.9 `show ip msdp mesh-group` —— MSDP 同步

```
Mesh group: rpmesh
  Source: 10.10.10.63
  Peers: 10.10.10.64

  Peer 10.10.10.64  State: established  Up/Down: 02:15:34
    SA messages received: 12
    SA messages sent: 8

字段：
  Source: 本端 loopback（不是 Anycast IP！是真实 loopback）
  Peers: 同 mesh-group 的其他 RP
  State: established = MSDP 会话已建立
  SA: Source Active 消息计数（多少个 source 同步过）
```

---

## 8. NVUE 配置详解

### 8.1 配置结构（6 大模块）

```
┌─────────────────────────────────────┐
│  1. Underlay：BGP + eBGP unnumbered │
│  2. VXLAN：source loopback + flood  │
│  3. Bridge：VLAN ↔ VNI ↔ mcast-grp  │
│  4. VRR：Anycast Gateway            │
│  5. EVPN-MH：ESI + DF + bonds       │
│  6. PIM：RP + interface enable      │
└─────────────────────────────────────┘
```

### 8.2 模块 1：Underlay BGP

```bash
# 接口（eBGP unnumbered，无 IP）
nv set interface swp1-4 type swp

# BGP 基础
nv set vrf default router bgp autonomous-system 65101
nv set vrf default router bgp router-id 10.10.10.1

# Neighbor（unnumbered，直接用接口名）
nv set vrf default router bgp neighbor swp1 remote-as external
nv set vrf default router bgp neighbor swp1 type unnumbered

# 启用 IPv4 unicast + EVPN
nv set vrf default router bgp address-family ipv4-unicast enable on
nv set vrf default router bgp address-family l2vpn-evpn enable on
nv set vrf default router bgp neighbor swp1 address-family l2vpn-evpn enable on
```

### 8.3 模块 2：VXLAN 源接口

```bash
# Loopback
nv set interface lo ip address 10.10.10.1/32
nv set interface lo ip address 10.10.100.100/32  # Anycast RP (仅 border)

# VXLAN 源
nv set nve vxlan source address 10.10.10.1
nv set nve vxlan arp-nd-suppress on  # ARP suppression
```

### 8.4 模块 3：Bridge + VLAN + VNI + Multicast ⭐核心⭐

```bash
# 创建 bridge domain
nv set bridge domain br_default vlan 10
nv set bridge domain br_default vlan 20
nv set bridge domain br_default vlan 30

# VLAN ↔ VNI 映射 + 多播组（Lab 2 关键）
nv set bridge domain br_default vlan 10 vni 10 mcast-group 224.1.1.10
nv set bridge domain br_default vlan 20 vni 20 mcast-group 224.1.1.20
nv set bridge domain br_default vlan 30 vni 30 mcast-group 224.1.1.30

# Lab 1 没有 mcast-group，用的是 head-end replication：
# nv set bridge domain br_default vlan 10 vni 10  ← Lab 1
```

### 8.5 模块 4：VRR Anycast Gateway

```bash
# SVI 唯一 IP（per leaf）
nv set interface vlan10 ip address 10.1.10.2/24  # leaf01
# leaf02: 10.1.10.3, leaf03: 10.1.10.4, leaf04: 10.1.10.5

# VRR Anycast IP + MAC（所有 leaf 相同）
nv set interface vlan10 ip vrr address 10.1.10.1/24
nv set interface vlan10 ip vrr mac-address 00:00:5e:00:01:01
nv set interface vlan10 ip vrr state up

# 绑定 VRF
nv set interface vlan10 ip vrf RED
nv set interface vlan20 ip vrf RED
nv set interface vlan30 ip vrf BLUE
```

### 8.6 模块 5：EVPN-MH 配置

```bash
# 全局启用 EVPN-MH
nv set evpn multihoming enable on

# Bond + ESI（同 ES 的两台 leaf 配相同 mac/id）
nv set interface bond1 bond member swp1
nv set interface bond1 evpn multihoming segment local-id 1
nv set interface bond1 evpn multihoming segment mac-address 44:38:39:be:ef:aa
nv set interface bond1 evpn multihoming segment df-preference 50000

# 自动生成 ESI：03:44:38:39:be:ef:aa:00:00:01
```

### 8.7 模块 6：PIM 配置（Lab 2 新增）

```bash
# 启用 PIM
nv set router pim enable on

# 在 underlay 接口启用 PIM
nv set interface swp1-4 router pim enable on
nv set interface lo router pim enable on  # ⚠️ Loopback 也要！

# RP 配置（所有节点都要知道 RP）
nv set router pim rp 10.10.100.100 group-range 224.0.0.0/4

# Border 额外配置：MSDP（Anycast RP 同步）
nv set router msdp mesh-group rpmesh source 10.10.10.63  # border01
nv set router msdp mesh-group rpmesh member 10.10.10.64

# border02：
# nv set router msdp mesh-group rpmesh source 10.10.10.64
# nv set router msdp mesh-group rpmesh member 10.10.10.63
```

### 8.8 关键注意事项

```
⚠️ Loopback 必须启用 PIM！
  原因：VXLAN 源 IP = loopback
  PIM Register 需要从 loopback 发出
  没启用 PIM 的 loopback → 无法触发 Register → BUM 黑洞

⚠️ Anycast RP 用额外 loopback IP
  border01 主 loopback：10.10.10.63（underlay BGP router-id）
  border02 主 loopback：10.10.10.64
  共享 Anycast IP：10.10.100.100（专用于 PIM RP）
  
  MSDP source 用主 loopback（10.10.10.63/64）
  PIM RP 配置用 Anycast IP（10.10.100.100）

⚠️ mcast-group 必须按 VNI 独立配置
  不能多个 VNI 共用一个多播组
  → 每个 VNI 一个 mcast-group
```

---

## 9. Lab 1 vs Lab 2 对比

### 9.1 配置差异

| 项目 | Lab 1 (EVPN-MH only) | Lab 2 (EVPN-MH + PIM) |
|------|---------------------|------------------------|
| BUM 复制方式 | Head-End Replication (HER) | PIM-SM 多播 |
| Bridge 配置 | `vni 10` | `vni 10 mcast-group 224.1.1.10` |
| PIM | 不启用 | 启用，配 RP |
| RP | 无 | Anycast RP @ border (10.10.100.100) |
| MSDP | 无 | mesh-group rpmesh |
| border 角色 | 普通 leaf | 额外作为 PIM RP |

### 9.2 数据平面差异

```
Lab 1 (HER) BUM 复制：
  leaf01 收 BUM → 复制 3 份 → 单播 VXLAN 给 leaf02/03/04
                            (3 份独立的单播包发上 spine)
  
Lab 2 (PIM) BUM 复制：
  leaf01 收 BUM → 1 份多播 VXLAN → 发给 RP/沿 SPT
                                  (spine 帮忙复制)
```

### 9.3 控制平面差异（BGP 路由数）

```
Lab 1：78 paths / 24 prefixes
Lab 2：39 paths / 12 prefixes

为什么 Lab 2 少？
  → HER 需要 Type-3 (IMET) 路由维护"复制列表"
  → PIM 不需要那么多 Type-3
  → Type-1 EAD-per-EVI 数量也下降
```

### 9.4 何时选哪个？

```
HER (Lab 1)：
  ✓ VTEP 数量少（< 10 台）
  ✓ 配置简单（无需 PIM）
  ✓ 单播 underlay 足够（无需多播 underlay）
  ✗ 大规模时带宽浪费

PIM (Lab 2)：
  ✓ VTEP 数量多（> 20 台，强烈推荐 > 50）
  ✓ BUM 流量大（如频繁 ARP 广播、未知单播）
  ✓ underlay 已经跑多播
  ✗ 配置复杂，需要 RP 高可用方案

实际部署：
  - 小型 DC：HER（简单可靠）
  - 大型 DC：PIM（带宽效率）
  - 超大型 DC：BUM 优化方案 + ARP suppression（避免 BUM）
```

---

## 10. NCP-AIN 考点总结

### 10.1 Tier 1：必考点（高频）

1. **EVPN-MH 替代 MLAG 的 4 优势**
   - 无 peerlink、多 ToR、统一控制面、多厂商互通

2. **ESI 结构**
   - Type-03（manual）= `03:<MAC>:<id>`
   - 同 ES 成员配置相同 → 自动加入

3. **5 种 EVPN 路由的作用**
   - Type-1: EAD（快速收敛 + Aliasing）
   - Type-2: MAC/IP
   - Type-3: IMET（HER 复制列表）
   - Type-4: ES Discovery + DF Election
   - Type-5: IP Prefix（外部路由）

4. **DF Election 算法**
   - df-preference 高的赢
   - 相等时 loopback IP 低的赢
   - 每 EVI 独立选举

5. **Split-Horizon 机制**
   - ESI-Label 在 VXLAN BUM 包里
   - 同 ES 成员丢弃匹配 label 的包

### 10.2 Tier 2：常考点（中频）

6. **PIM-SM 三阶段**
   - Join / Register / SPT Switchover

7. **RPT vs SPT**
   - (*, G) vs (S, G)
   - 何时切换

8. **Anycast RP + MSDP**
   - 两 RP 共享 IP
   - MSDP 同步 source 信息

9. **mroute Flags**
   - S/F/T 含义

10. **VRR vs VRRP**
    - VRR = 所有 leaf 同时活跃（active-active）
    - VRRP = 主备（active-standby）

### 10.3 Tier 3：少考但要懂（低频但加分）

11. **EAD-per-ES vs EAD-per-EVI 区别**
12. **L3VNI 在 Symmetric IRB 的作用**
13. **NVUE vs CLI 配置差异**
14. **MSDP 工作原理**
15. **DR 选举（PIM 概念）**

### 10.4 易错考题模式

```
Q1: "DF 是怎么选的？"
  ✗ 错答：preempt mode / IP 高的赢
  ✓ 正答：df-preference 高的赢，相等用 loopback IP 低的

Q2: "Split-Horizon 怎么工作？"
  ✗ 错答：基于 MAC 地址
  ✓ 正答：基于 VXLAN 包里的 ESI-Label，同 ES 成员丢弃

Q3: "Anycast RP 需要什么协议同步？"
  ✓ MSDP（Multicast Source Discovery Protocol）

Q4: "EVPN-MH 的 ESI Type 是什么？"
  ✓ Type-0/1/2/3/4/5，Lab 用 Type-03（manual）

Q5: "PIM-SM 默认什么时候从 RPT 切到 SPT？"
  ✓ 默认收到第一个包就切（threshold 0）
```

---

## 11. 易混淆概念速查

| 概念 A | 概念 B | 关键区别 |
|--------|--------|----------|
| **DR** (PIM) | **DF** (EVPN) | DR 选 PIM 代表；DF 选 EVPN BUM 转发者 |
| **EAD-per-ES** | **EAD-per-EVI** | EthTag=0xFFFFFFFF 带 ESI-Label；EthTag=0 不带，per VNI |
| **RPT** | **SPT** | (*,G) 经 RP；(S,G) 直接最短路径 |
| **Type-3 IMET** | **Type-1 EAD** | Type-3 用于 HER 复制列表；Type-1 用于 MH 收敛+Aliasing |
| **VRR** | **VRRP** | VRR 所有成员同时活跃；VRRP 主备 |
| **HER** | **PIM 多播** | HER 入口复制 N 份单播；PIM 入口发 1 份多播，spine 复制 |
| **L2VNI** | **L3VNI** | L2VNI 承载二层流量；L3VNI 承载 Symmetric IRB 三层流量 |
| **ES** | **ESI** | ES 是逻辑段；ESI 是它的 ID |
| **MLAG peerlink** | **EVPN-MH** | MLAG 需 ISL；MH 无 ISL |
| **Anycast Gateway** | **Anycast RP** | 网关 IP 共享给 leaf；RP IP 共享给 border |
| **MSDP** | **PIM** | MSDP 同步 source；PIM 建多播树 |
| **(*, G) RPF** | **(S, G) RPF** | 前者朝 RP；后者朝 source |
| **DF preference** | **BGP local-pref** | DF 用于 EVPN-MH 选举；BGP 用于 best-path |

---

## 12. 故障排查命令速查

```bash
# EVPN-MH 排查
net show evpn es                   # ES 列表
net show evpn es <ESI> detail      # ES 详情（含 DF 状态）
net show evpn es-evi               # ES × EVI
net show evpn es-vrf               # L3 NHG
net show bgp l2vpn evpn route type 1  # Type-1
net show bgp l2vpn evpn route type 4  # Type-4 (DF election)

# PIM 排查
net show pim state                 # PIM 状态
net show pim upstream              # 上游邻居
net show pim rp-info               # RP 信息
net show ip mroute                 # 多播路由表
net show msdp mesh-group           # MSDP 同步

# 数据平面验证
sudo bridge fdb show               # FDB 表（MAC 学习）
sudo ip route show vrf RED         # VRF 路由
sudo tcpdump -i swp1 -n 'udp port 4789'  # 抓 VXLAN
```

---

## 13. 总结

```
EVPN-MH 解决了什么：
  → 替代 MLAG，提供多活、无 peerlink、多厂商互通

PIM 解决了什么：
  → 替代 HER，大规模 VTEP 时节省 BUM 复制带宽

两者结合（Lab 2）：
  → 现代云数据中心的标准方案
  → NVIDIA Spectrum + Cumulus 默认推荐
  → NCP-AIN 重点考察

学完这两个 Lab 你应该能：
  ✓ 解释 EVPN-MH 的 4 大优势
  ✓ 画出 DF Election 流程
  ✓ 追踪 BUM 流量端到端路径
  ✓ 区分 5 种 EVPN 路由类型作用
  ✓ 配置 EVPN-MH + PIM Anycast RP
  ✓ 用 show 命令诊断常见问题
```

---

**笔记完成日期**：2026-05-21
**下一步建议**：
- 复习本笔记 + Symmetric Lab 笔记
- 准备 NCP-AIN 模拟题
- 可选：转向 Spectrum-X / Adaptive Routing / InfiniBand 主题
