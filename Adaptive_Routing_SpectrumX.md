# Adaptive Routing (Spectrum-X) 深度解析

> NCP-AIN Day 3 学习笔记 | Gap-3 | Domain 2 (30% 权重) 核心
> 前置: 已掌握 ECMP / Hash / RoCE / DCQCN / PFC / SuperNIC 概念

---

## 目录

1. [为什么需要 Adaptive Routing](#1-为什么需要-adaptive-routing)
2. [AR 核心思想](#2-ar-核心思想)
3. [AR 工作机制](#3-ar-工作机制)
4. [AR + DCQCN + Telemetry 三位一体](#4-ar--dcqcn--telemetry-三位一体)
5. [AR 的限制和注意事项](#5-ar-的限制和注意事项)
6. [考试速记](#6-考试速记)

---

## 1. 为什么需要 Adaptive Routing

### 1.1 传统 ECMP 工作原理回顾

```
Leaf-1 有 4 条到 Leaf-2 的等价路径 (经过 4 个 Spine):

          Spine-1   Spine-2   Spine-3   Spine-4
            │         │         │         │
            └────┬────┴────┬────┴────┬────┘
                 │         │         │
               Leaf-1              Leaf-2

ECMP 选路逻辑:
  hash(src_ip, dst_ip, src_port, dst_port, proto) % 4
  → 0/1/2/3 = 选 Spine-1/2/3/4

关键: 同一条 flow 永远走同一个 Spine (保证 TCP 不乱序)
```

### 1.2 AI 流量的特殊性 = ECMP 灾难

**互联网流量 vs AI 训练流量**

| 维度 | 互联网流量 | AI 训练流量 |
|------|------------|-------------|
| 流数量 | 成千上万小流 | 极少数巨流 |
| 单流大小 | 几 KB (HTTP) | 几 GB (一次 all-reduce) |
| 持续时间 | 短 (ms 级) | 长 (秒级) |
| Hash 分布 | 大数定律, 均匀 | 大概率冲突 |
| ECMP 效果 | ✓ 完美 | ✗ 灾难 |

**典型冲突场景**

```
256 GPU 集群, 同时只有 ~16 条 flow, 4 路 ECMP

期望:
  Flow-A → Spine-1
  Flow-B → Spine-2     每个 Spine 100Gbps, 完美
  Flow-C → Spine-3
  Flow-D → Spine-4

现实 (Hash 冲突):
  Flow-A → Spine-1
  Flow-B → Spine-1  ← 撞了!     Spine-1: 300Gbps (拥塞)
  Flow-C → Spine-3              Spine-2: 0     (空闲)
  Flow-D → Spine-1  ← 又撞了!   Spine-3: 100Gbps
                                Spine-4: 0     (空闲)

后果:
  - Spine-1 拥塞 → 触发 PFC → 整网反压
  - Spine-2/4 完全空闲 → 带宽浪费 75%
  - 集合通信慢 → GPU 等数据 → 训练效率暴跌
```

### 1.3 业界实测数据

| 场景 | 带宽利用率 |
|------|------------|
| 传统 ECMP (Meta/MS 公开论文) | 50-70% |
| 最差情况 (8 流全撞 1 Spine) | 30% |
| Spectrum-X AR | 95%+ |

**经济代价**

```
GPU $40k/张 × 256 = $10M+ 集群
带宽利用率每降 10% → GPU 等待 +X%
训练 7 天 → 实际有效计算可能只有 4 天

这就是 NVIDIA Spectrum-X 的卖点:
"传统以太网 60% 利用率 → Spectrum-X 95%+"
```

---

## 2. AR 核心思想

### 2.1 三种路由粒度对比

```
═══════════════════════════════════════════════════════════════
  ① Per-Flow ECMP (传统)
═══════════════════════════════════════════════════════════════
  粒度: 一条 flow 整体
  选路依据: 5-tuple hash

  Flow-A 全部包 ─────────▶ Spine-1
  Flow-B 全部包 ─────────▶ Spine-1  ← 撞了, 不能改

  优点: TCP 不乱序 (简单)
  缺点: AI 流场景下严重不均衡


═══════════════════════════════════════════════════════════════
  ② Per-Flowlet ECMP (中庸)
═══════════════════════════════════════════════════════════════
  粒度: flow 内的"突发" (gap >50μs 视为新 flowlet)
  选路依据: 每个 flowlet 重新 hash + 当前链路负载

  Flow-A burst1 ─▶ Spine-1
       (gap 100μs)
  Flow-A burst2 ─▶ Spine-2  ← 同 flow 可换路

  优点: 比 Per-Flow 均衡
  缺点: 依赖流量有 gap, AI 大流没 gap


═══════════════════════════════════════════════════════════════
  ③ Per-Packet Spraying (Spectrum-X AR)
═══════════════════════════════════════════════════════════════
  粒度: 每个包独立选路
  选路依据: 实时链路队列深度

  Flow-A pkt1 ─▶ Spine-1 (队列最浅)
  Flow-A pkt2 ─▶ Spine-3 (此时 Spine-1 满了)
  Flow-A pkt3 ─▶ Spine-2
  Flow-A pkt4 ─▶ Spine-4

  优点: 完美均衡, 利用率 95%+
  缺点: 必然乱序 → 需要 SuperNIC 重组 (关键!)
```

### 2.2 为什么传统网络不能做 Per-Packet

**障碍 1: TCP 乱序敏感**

```
TCP 收到乱序包 → 触发快速重传 → 假定丢包 → 降窗
传统网络一旦乱序 → 性能比拥塞还差

→ 所以传统网络只能 Per-Flow
```

**障碍 2: 接收端无法重组**

```
普通 NIC 收到乱序包:
  - 没有硬件重排序能力
  - 只能上送 CPU 软件重组 (慢)
  - RDMA 干脆不支持乱序

→ 所以 RoCE 默认要求"无丢包 + 顺序到达" (PFC 强需求)
```

**Spectrum-X 怎么破**

```
1. 用 RoCE (不是 TCP), 不依赖 TCP 重传逻辑
2. SuperNIC 硬件支持 DDP (Direct Data Placement)
   即使乱序到达, NIC 硬件能根据 RDMA 头部
   把数据直接放到正确的 GPU 内存位置
3. 全部包到齐后, NIC 通知 GPU "数据完整"

★ 核心: AR 必须 Spectrum-X 交换机 + SuperNIC 配套, 缺一不可
```

---

## 3. AR 工作机制

### 3.1 数据平面：交换机怎么选路

```
═══════════════════════════════════════════════════════════════
  Spectrum-4 ASIC 内部决策流程
═══════════════════════════════════════════════════════════════

包到达 Leaf-1, 目的是 Leaf-2:

┌─────────────────────────────────────────────────┐
│ Step 1: 查路由表                                 │
│  → 发现到 Leaf-2 有 4 条 ECMP 路径               │
│    next-hop = [Spine-1, Spine-2, Spine-3, Spine-4]│
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ Step 2: 查每个出口的实时队列深度                  │
│  → ASIC 内部 telemetry, 纳秒级更新               │
│    Spine-1 出口队列: 80% 满 (拥塞)               │
│    Spine-2 出口队列: 5%  (空闲)                  │
│    Spine-3 出口队列: 30% (轻载)                  │
│    Spine-4 出口队列: 95% 满 (拥塞)               │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ Step 3: 选最优出口                                │
│  → 算法: 选队列最浅的 (Spine-2)                  │
│  → 或者: 在前 N 浅的里随机 (避免羊群效应)         │
└────────────────────┬────────────────────────────┘
                     ▼
┌─────────────────────────────────────────────────┐
│ Step 4: 标记包 + 转发                             │
│  → 给包打 AR 标记 (告诉接收端"这是乱序包")        │
│  → 从 Spine-2 出口发出                            │
└─────────────────────────────────────────────────┘

下一个包 (10ns 后):
  Spine-2 队列可能已经升到 20% (因为大家都选它)
  重新评估 → 改选 Spine-3

★ 每个包独立决策, 纳秒级响应链路状态
```

### 3.2 关键: 出口队列深度怎么测

```
每个出口端口都有:
  ┌─────────────────────────────┐
  │ Egress Queue Counter         │
  │  - 当前缓存字节数 (实时)      │
  │  - 历史平均深度 (滑动窗口)    │
  │  - 拥塞标记 (是否触发 ECN)    │
  └──────────────┬──────────────┘
                 │ ns 级更新
                 ▼
  ┌─────────────────────────────┐
  │ AR Decision Engine (ASIC)    │
  │  - 综合 N 个出口的指标        │
  │  - 输出选路决策               │
  └─────────────────────────────┘

对比 ECMP:
  ECMP: 只看 hash, 不看队列
  AR:   每个包看队列, 动态调整
```

**全局视角: AR 不是孤立决策**

```
Spectrum-4 还可以接收:
  - 邻居交换机的拥塞信号 (RoCE CNP)
  - 下游 Spine 的反馈
  - 全网 telemetry (通过控制平面)

形成"全局感知, 本地决策"
  → 比单纯本地队列更智能
  → 避免"刚转过去也拥塞了"的尴尬
```

### 3.3 接收端: SuperNIC 怎么重组 (DDP)

```
═══════════════════════════════════════════════════════════════
  Direct Data Placement (DDP) 机制
═══════════════════════════════════════════════════════════════

传统场景 (普通 NIC):
  收到包 → 放接收队列 → CPU 排序 → 拷贝到应用内存
  乱序时: CPU 缓存 + 重排 → 慢


Spectrum-X SuperNIC (DDP):

  RDMA Write 操作示例:
    发送端: "把数据 0-10MB 写到接收端 GPU 内存 0x1000"
    数据被切成 N 个包, 每个包带 offset

  接收端 SuperNIC 收到乱序包:
    Packet 1: offset=0KB,   data → 直接写 GPU 内存 0x1000
    Packet 5: offset=4MB,   data → 直接写 GPU 内存 0x1000+4MB
    Packet 2: offset=1KB,   data → 直接写 GPU 内存 0x1000+1KB
    Packet 3: offset=2KB,   data → 直接写 GPU 内存 0x1000+2KB
    ...

  NIC 硬件维护 bitmap, 跟踪哪些 offset 已收到
  全部到齐 → 通知 GPU "10MB 数据完整, 在 0x1000"

★ 关键: 即使乱序, 数据"位置"是确定的
       所以可以直接 DMA, 不需要软件排序
```

### 3.4 完整数据路径

```
═══════════════════════════════════════════════════════════════
  Spectrum-X AR 端到端流程
═══════════════════════════════════════════════════════════════

GPU-A (发送端)
  │
  │ NCCL 发起 RDMA Write 10MB
  ▼
SuperNIC-A
  │ 切成 1000 个 1500 字节的包
  │ 每个包带 RDMA 头部 (含 offset)
  │ 全部送给 Leaf-1
  ▼
┌────────────────────────────────────────┐
│ Leaf-1 (Spectrum-4 ASIC)                │
│                                          │
│ Pkt 1 → AR decision → Spine-2 (最浅)    │
│ Pkt 2 → AR decision → Spine-1 (此时浅)  │
│ Pkt 3 → AR decision → Spine-3           │
│ Pkt 4 → AR decision → Spine-4           │
│ Pkt 5 → AR decision → Spine-2 (循环)    │
│ ...                                      │
│ 4 个 Spine 流量完全均衡 ✓               │
└────────────────────────────────────────┘
                  │
      ┌───────────┼───────────┐
      ▼           ▼           ▼
   Spine-1     Spine-2     Spine-3     Spine-4
      │           │           │           │
      └───────────┴───────────┴───────────┘
                  ▼
┌────────────────────────────────────────┐
│ Leaf-2                                   │
│ 接收乱序到达的包 (顺序: 2,1,5,3,4...)    │
│ 直接转发给 SuperNIC-B (不重排)           │
└────────────────────────────────────────┘
                  ▼
┌────────────────────────────────────────┐
│ SuperNIC-B (DDP 机制)                    │
│                                          │
│ 收到 Pkt 2 (offset=1500)                │
│   → DMA 写 GPU 内存 base+1500            │
│ 收到 Pkt 1 (offset=0)                   │
│   → DMA 写 GPU 内存 base+0               │
│ 收到 Pkt 5 (offset=6000)                │
│   → DMA 写 GPU 内存 base+6000            │
│ ...                                      │
│ 维护 bitmap: [✓✓✓✓✓...]                │
│                                          │
│ 全部到齐 → 通知 GPU-B "数据完整"          │
└────────────────────────────────────────┘
                  ▼
                GPU-B (收到完整 10MB)

关键点:
  1. 网络中: 每包独立选路, 负载完美均衡
  2. 接收时: 即使乱序, NIC 硬件直写正确内存位置
  3. 通知时: 全部到齐才告诉 GPU, 上层无感乱序
```

---

## 4. AR + DCQCN + Telemetry 三位一体

### 4.1 为什么 AR 不够, 还要 DCQCN 增强

```
═══════════════════════════════════════════════════════════════
  AR 解决"分布", DCQCN 解决"总量"
═══════════════════════════════════════════════════════════════

场景: 4 个 Leaf 同时往 Leaf-5 发数据 (incast)

          Leaf-1    Leaf-2    Leaf-3    Leaf-4
            │         │         │         │
            │ 100G    │ 100G    │ 100G    │ 100G
            └────┬────┴────┬────┴────┬────┘
                 │         │         │
                Spine layer (AR 均衡)
                 │         │         │
                 └────┬────┴────┬────┘
                      │
                      ▼ 400G 涌向 Leaf-5
                   Leaf-5 (出口只有 100G)
                      ▼
                   接收端拥塞 (incast!)

AR 帮不上: 4 个 Spine 各 100G 都满了, 已经均衡了
            但目的地 Leaf-5 还是接不住 400G

必须靠 DCQCN:
  Leaf-5 出口拥塞 → 标记 ECN
  SuperNIC 收到 ECN → 发 CNP 给 4 个发送端
  4 个发送端降速到 25G → 总共 100G, 不再拥塞
```

**三者分工**

| 机制 | 解决问题 | 作用范围 |
|------|----------|----------|
| **AR** | 路径选择不均衡 | 网络中段 |
| **DCQCN** | 端到端总量超限 | 端到端 |
| **PFC** | 极端情况丢包 | 链路层兜底 |

**缺一不可: 三者协同才是完整 Spectrum-X**

### 4.2 Telemetry 的角色

```
═══════════════════════════════════════════════════════════════
  端到端可观测性
═══════════════════════════════════════════════════════════════

交换机侧:
  - 每端口实时队列深度 (ns 级)
  - 每端口 ECN 标记次数
  - PFC 触发次数
  - AR 决策日志 (谁选了哪条路)

SuperNIC 侧:
  - 收发包速率
  - 乱序到达统计
  - CNP 发送/接收次数
  - DCQCN 当前速率

集中收集:
  NetQ / Prometheus / Grafana
  → 训练慢的时候能定位是哪个 GPU/链路/交换机的问题


典型用途:
  "为什么这次训练比上次慢 20%?"
  → 看 telemetry: Leaf-3 出口 PFC 触发 1000 次/秒
  → 定位: Leaf-3 某端口故障, 链路降速
```

---

## 5. AR 的限制和注意事项

### 5.1 AR 不是万能

**适用 vs 不适用**

| ✓ AR 适用 | ✗ AR 不适用 |
|-----------|-------------|
| RDMA/RoCE 流量 (有 SuperNIC DDP) | TCP 流量 (没法 DDP) |
| 大流为主 (AI 训练) | 混合环境 (Spectrum-X + 普通 NIC) |
| 端到端都是 Spectrum-X | 跨厂商 (其他厂商交换机不支持) |

### 5.2 混合场景的退化

```
① Spectrum-X 交换机 + 普通 ConnectX-6 NIC:
  - 交换机能做 AR
  - 但 NIC 不能 DDP
  - 乱序到达 → 上送 CPU 软件重排 → 慢
  → 不如关掉 AR, 用传统 ECMP

② 普通交换机 + SuperNIC:
  - NIC 能 DDP, 但交换机不会乱序发
  - SuperNIC 优势用不上
  → 只能享受 SuperNIC 的 RoCE 优化, AR 失效


★ 考试要点:
  AR 必须"三件套齐全":
    Spectrum-X 交换机 + SuperNIC + DOCA
  缺一退化为普通 ECMP
```

### 5.3 配置注意

```
开启 AR 不是默认的, 需要显式配置:
  - 在交换机端口启用 AR
  - 配置 AR 算法 (random / least-loaded)
  - 确保配套 SuperNIC 端开启 DDP


一致性要求:
  - 同一对 Leaf 之间所有 Spine 都要支持 AR
  - 部分 Spine 不支持 → AR 退化
  - 拓扑必须严格对称 (Rail-Optimized 天然满足)
```

---

## 6. 考试速记

### 6.1 核心知识点 (必背)

**概念**
- ECMP 在 AI 场景 = 大流 hash 冲突 → 严重不均衡
- AR = Per-Packet Spraying (每包独立选路)
- AR 必须 Spectrum-X 三件套 (交换机 + SuperNIC + DOCA)

**机制**
- 交换机端: 看实时队列深度, 选最浅的出口
- 接收端: SuperNIC DDP 直写 GPU 内存, 容忍乱序
- 全程: 上层应用无感, NCCL 不用改

**与其他机制关系**
- AR + DCQCN + PFC 三者协同, 缺一不可
- AR 解决"分布", DCQCN 解决"总量", PFC 兜底"无损"

**限制**
- 只对 RDMA 有效 (TCP 不行)
- 必须端到端 Spectrum-X (混合环境退化)
- 拓扑要对称 (Rail-Optimized 天然适合)

**数字**
- 传统 ECMP 利用率: 50-70%
- Spectrum-X AR 利用率: 95%+

### 6.2 典型考题

**Q1**: Why is traditional ECMP problematic in AI training networks?
- A. Too few paths
- **B. Few large flows cause hash collisions, leading to imbalance** ✓
- C. ECMP doesn't support 400G
- D. RDMA bypasses ECMP

**Q2**: What enables Adaptive Routing in Spectrum-X to handle out-of-order packets at the receiver?
- A. TCP reordering
- B. CPU software sorting
- **C. SuperNIC Direct Data Placement (DDP)** ✓
- D. Spine reordering

**Q3**: To deploy Spectrum-X Adaptive Routing, what components are required?
- A. Any Ethernet switch + ConnectX NIC
- **B. Spectrum-X switches + SuperNIC + DOCA** ✓
- C. InfiniBand switches only
- D. Cumulus Linux only

**Q4**: AR balances path selection. What handles end-to-end congestion (incast)?
- A. AR alone is sufficient
- B. PFC alone
- **C. DCQCN (RoCE congestion control)** ✓
- D. TCP slow start

**Q5**: In a mixed environment (Spectrum-X switches + non-SuperNIC), what happens to AR?
- A. Works normally
- **B. Degrades due to no DDP support → fallback to ECMP recommended** ✓
- C. Improves performance
- D. Causes packet loss

---

## 一句话总结

```
┌──────────────────────────────────────────────────────────────┐
│ ECMP:   AI 大流 hash 冲突 → 50-70% 利用率                     │
│ AR:     每包独立选路 + DDP 重组 → 95%+ 利用率                 │
│ 关键:   必须三件套 (Spectrum-X 交换机 + SuperNIC + DOCA)      │
│ 协同:   AR (分布) + DCQCN (总量) + PFC (兜底)                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 关联笔记

- `DOCA_BlueField_SuperNIC.md` - SuperNIC / DOCA / Spectrum-X 三件套
- `RoCE_vs_SpectrumX_Congestion_Control.md` - DCQCN / PFC / ECN 详解
- `Rail-Optimized_vs_Fat-Tree.md` - AR 依赖的对称拓扑
- `NCCL.md` - AR 服务的上层应用
