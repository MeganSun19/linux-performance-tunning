# cl-resource-query — Spectrum-X ASIC 资源诊断

> NCP-AIN 考纲：**Domain 5.1**（D5 = 20% 权重）"Use tools like cl-resource-query to diagnose forwarding table exhaustion"

---

## 1. 核心心智模型

```
┌──────────────────────────────────────────────────────────────┐
│  控制面 (FRR/zebra/kernel)         数据面 (ASIC KVD)          │
│  ─────────────────────────         ────────────────────       │
│  ip route show: 100k routes        Spectrum-2 LPM: 上限 82k   │
│  vtysh 查 BGP: 全部正常 ✓          硬件物理限制                │
│  show evpn mac: 全部学到 ✓                                    │
│           │                                │                  │
│           └──────── switchd sync ──────────┘                  │
│                                            │                  │
│                                  ⚠️ 装不下时:                 │
│                                  partial allocation           │
│                                  → 随机丢包 (silent drop)     │
│                                  → 控制面看一切正常           │
│                                  → 只有 cl-resource-query 暴露│
└──────────────────────────────────────────────────────────────┘
```

**关键认知**：控制面正常 ≠ 数据面正常。`cl-resource-query` 是看 ASIC 真相的唯一工具。

---

## 2. 命令速查

```bash
# Linux 原生 (所有版本)
sudo cl-resource-query

# NVUE 等价 (CL 5.0+, 考试推荐答案)
nv show platform asic resource          # 全部
nv show platform asic resource global   # 仅路由/MAC/ECMP/...
nv show platform asic resource acl      # 仅 ACL (11 张子表细分)
```

> ★ **NVUE 比 cl-resource-query 多出的关键字段**：`ECMP-nexthops`（组员总数）、`Total-FID-count`（VXLAN VNI）、ACL 11 张子表细分。AI/EVPN 场景排障必看。

---

## 3. 4 张 ASIC 表关系（必背）

```
┌─────────────────────────────────────────────────────────────┐
│  数据包到达 → ASIC 查表顺序                                   │
│                                                               │
│  1. Host Table (/32 精确匹配)  ◀── ARP/邻居/EVPN Type-2      │
│         │ 未命中                                              │
│         ▼                                                     │
│  2. LPM Table (/1~/31 前缀)    ◀── BGP/OSPF/EVPN Type-5      │
│         │ 命中, 拿到 next-hop                                 │
│         ▼                                                     │
│  3. ECMP Table (组) ──────────▶  存指针数组, 每组 N 成员     │
│         │ hash(5-tuple) % N                                   │
│         ▼                                                     │
│  4. Adjacency Table (MAC+出口) ◀── 实际转发信息              │
└─────────────────────────────────────────────────────────────┘

一条 8-way ECMP 路由消耗:
  1 LPM + 1 ECMP group + 8 Adjacency + N Host (N=对端 GPU 数)
```

**容量共享规则**：
- 多条路由 next-hop 完全相同 → 共享 1 个 ECMP 组
- 多个 ECMP 组成员相同 → 共享 Adjacency
- 但实际 AI 场景策略多样，共享率很低 → ECMP 最先满

---

## 4. Spectrum-2 default profile 上限（必背数字）

| 资源 | 上限 | AI 场景瓶颈度 |
|------|------|--------------|
| IPv4 host entries | 41,360 | ★★★ EVPN-Sym 易爆 |
| IPv4 route entries (LPM) | 82,720 | ★★ |
| **ECMP entries** | **8,571** | **★★★★★ AI 必看** |
| Unicast Adjacency | 33,087 | ★★★ |
| MAC entries | 57,903 | ★★ 云租户 |
| Total Mcast Routes | **1,000** | **★★★★ EVPN-MH 必爆** |

> ★ **AI 网络中 ECMP 永远是第一瓶颈**，因为 Rail-Optimized 拓扑里每条路由都是 8 路 ECMP（8 个 spine），ECMP 组数随 leaf/路由数线性增长。

---

## 5. Forwarding Profile（11 种，记住 5 个重点）

```
┌────────────────┬────────────────────────────────────────────┐
│ Profile        │ 何时用                                      │
├────────────────┼────────────────────────────────────────────┤
│ default        │ 通用 (但 AI/EVPN 几乎都不够)               │
│ l2-heavy       │ 多租户云 (MAC 115k, host 87k)              │
│ l2-heavy-1/2/3 │ 极端云租户 (MAC 239k+, 牺牲 LPM)           │
│ v4-lpm-heavy   │ 边界路由器 (LPM 256k)                       │
│ ipmc-heavy     │ EVPN-MH 多播 (Mcast 8k, 推荐)              │
│ ipmc-max       │ 极端多播 (Mcast 15k, 但仅部分验证 ⚠️)      │
│ ecmp-nh-heavy  │ AI 集群 (ECMP 32k+, CL 5.9.2+, 不支持      │
│                │ warm restart)                              │
└────────────────┴────────────────────────────────────────────┘

记忆口诀:
  l2-heavy-N     → MAC 翻倍
  v4/v6-lpm-N    → LPM 翻倍
  ipmc-heavy/max → 组播翻倍
  ecmp-nh-heavy  → ECMP 翻倍
```

### 切换命令

```bash
nv set system forwarding profile <name>
nv config apply
# ⚠️ switchd 重启, 数据面中断 30s-2min
# ⚠️ 必须维护窗口
```

### 5 个陷阱（考试爱出）

1. **切 profile 会断流量** 30s-2min（switchd 重启）
2. **ipmc-max 显示 15000 但只验证到 ~10000**，别无脑选 max
3. **ecmp-nh-heavy 不支持 warm restart**（CL 5.9.2+）
4. **Spectrum-1 与 Spectrum-2/3/4 profile 不兼容**
5. **ACL entries 是 18B 单宽计数**，IPv6 ACL（54B）实际占 3 倍

---

## 6. AI 网络容量规划三板斧

### 6.1 IP 规划：按 Rail 分网段

```
错误: 按机架划分 IP → 跨 Rail 流量混乱
正确: 按 Rail 划分 IP → 一个 Rail 一个网段

  Rail-0: 10.0.0.0/22 (或更大)
  Rail-1: 10.1.0.0/22
  ...

  网段大小按规模选:
    小集群 (<500 GPU):    /24
    中型 (500-5000):      /22 ~ /20
    大型 (>5000):         /16
```

### 6.2 路由汇总：spine 侧 aggregate-address

```bash
# spine 上配 BGP 路由聚合
router bgp 65001
  address-family ipv4 unicast
    aggregate-address 10.0.0.0/20 summary-only
                                  ^^^^^^^^^^^^
                                  关键: 只发汇总, 不发具体 /32
```

**效果（1024 GPU 集群）**：

| | LPM | Host |
|--|----|------|
| 无汇总 | 1024 条 | 1024 条 |
| 按 Rail /20 汇总 | **8 条** | **8 条** |
| 节省 | 99% | 99% |

**3 个汇总原则**：
1. **spine 汇总，leaf 不汇总**（leaf 要响应本地 ARP，需精确 /32）
2. **按 Rail 维度汇总，不跨 Rail**（跨 Rail 汇总破坏拓扑信息）
3. **汇总范围必须连续**，否则黑洞路由

### 6.3 选对 profile

```
AI Rail-Optimized 集群:    ecmp-nh-heavy
EVPN-MH + 多播:            ipmc-heavy
EVPN Symmetric 多租户云:   l2-heavy
互联网边界:                v4-lpm-heavy
```

---

## 7. 三个故障场景（背模板）

### 场景 A：AI 集群扩容后 NCCL 卡死

```
现象:
  - 扩容后 NCCL all-reduce 卡在 init
  - ping 部分通部分不通 (50% 黑洞)
  - BGP/路由表控制面全部正常

诊断:
  sudo cl-resource-query
    ECMP entries:        8200,  95% of max  8571   🚨
    Unicast Adjacency:  31800,  96% of max 33087   🚨
  
  cat /var/log/switchd.log | grep partial
    [WARN] route 10.0.5.0/24: partial allocation (4 of 8 nexthops)
                                                  ^^^^^^^^^^^^^^^
                                  ★ 沉默杀手: ASIC 仍按 8-way hash
                                  hash 到 [4-7] 索引 = 黑洞

根因: 扩容时漏配 BGP aggregate-address
       → 1024 个 /32 全传到 spine
       → 每条 = 1 个 ECMP 组 → 表满

修复:
  1. 临时: 切 ecmp-nh-heavy profile (ECMP 8.5k → 32k+)
  2. 根治: spine 配 aggregate-address summary-only
```

### 场景 B：EVPN-MH 多播突然丢包

```
现象:
  - 新加 200 个组播应用后, 原组播部分丢包

诊断:
  sudo cl-resource-query | grep Mcast
    Total Mcast Routes: 1000, 100% of max 1000  🚨

根因: default profile mcast 只有 1000
      PIM-SM 每组占 (*,G) + N×(S,G)
      EVPN-MH 每 VLAN 额外占 BUM 复制项
      估算: VLAN 数 × 3 + 业务组播 (S,G) 数

修复: nv set system forwarding profile ipmc-heavy
      Mcast: 1k → 8k

★ 不要无脑选 ipmc-max (15k 验证不充分)
```

### 场景 C：EVPN Sym 多租户云 VM ARP 不通

```
现象:
  - 新 VM 跨 leaf 不通, 本 leaf 内通
  - vtysh show evpn mac vni 全部正常 (控制面假象)

诊断:
  sudo cl-resource-query
    IPv4 host entries: 41280, 99% of max 41360  🚨
  
  switchd.log:
    [ERROR] host route x.x.x.x/32: install failed, table full
    [ERROR] EVPN type-2 (MAC+IP): not programmed

根因: EVPN Symmetric Type-2 路由 = MAC + IP /32
      每个远端 VM 都消耗本地 host entry
      公式: VM 数 × 2 (本地+远端) × VRF 复制效应

修复: nv set system forwarding profile l2-heavy
      host: 41k → 87k, MAC: 57k → 115k

★ 教训: 控制面看不出, 必须 cl-resource-query 验证 ASIC
```

---

## 8. 三场景对比

| 场景 | 主要瓶颈 | 触发原因 | Profile |
|------|---------|---------|---------|
| AI 集群扩容 | **ECMP + Adjacency** | 漏配 aggregate-address | ecmp-nh-heavy |
| EVPN-MH 组播 | **Mcast (1000 满)** | EVPN-MH BUM + 业务组播双重 | ipmc-heavy |
| EVPN Sym 多租户 | **host entries** | Type-2 携带 IP /32 | l2-heavy |

---

## 9. 排障组合命令

```bash
# 1. ASIC 资源体检
sudo cl-resource-query

# 2. 看是否有 partial allocation (沉默杀手)
cat /var/log/switchd.log | tail -100 | grep -i partial

# 3. 对比内核 vs ASIC (有多少没下发)
ip route | wc -l
sudo vtysh -c "show ip route summary"

# 4. 看具体路由是否在 ASIC
sudo vtysh -c "show ip route 10.0.0.0/24"
# 期望: 输出含 "fib" 标记 = 已下发 ASIC

# 5. EVPN 场景
sudo vtysh -c "show evpn mac vni all" | wc -l

# 6. ACL 细分 (NVUE 独有)
nv show platform asic resource acl
```

---

## 10. 易错点速查

| ❌ 错误认知 | ✅ 正确 |
|------------|--------|
| 内核路由全 = ASIC 全 | 内核无上限，ASIC 才有限制 |
| 控制面 OK = 数据面 OK | 必须 cl-resource-query 验证 |
| 切 profile 不断流 | **断** 30s-2min |
| ipmc-max 33k 能用 | **仅验证 ~10k** |
| ACL entries 数 = 实际占用 | 18B 单宽计数，IPv6 占 3 倍 |
| Adjacency 满 = 路由满 | Adjacency 是唯一下一跳，可能路由多但下一跳少 |
| AI 网络 MAC 表会满 | **永远不会**，L3 拓扑每 leaf MAC ≈ 接入 GPU 数 |
| partial allocation 会主动报警 | **不会**，silent drop，必须看 switchd.log |

---

## 11. 自测题

1. **AI 网络中哪个表最先满？为什么？**
   <details><summary>答</summary>ECMP entries。Rail-Optimized 8-way ECMP，每条路由消耗 1 个 ECMP 组，路由数随 leaf 数线性增长。</details>

2. **partial allocation 是什么？后果？**
   <details><summary>答</summary>ASIC 装不下完整 ECMP 成员（如 8 个只装下 4 个），但 hash 仍按 8 算，hash 到不存在的索引 = silent drop。50% 黑洞，无主动告警。</details>

3. **EVPN Symmetric 为什么 host 表先满？**
   <details><summary>答</summary>Type-2 携带 MAC+IP，每个远端 VM 都占本地 host /32 entry。default 上限 41k，5000 VM × 2 (本地+远端) 易爆。</details>

4. **切 forwarding profile 影响数据面吗？**
   <details><summary>答</summary>是，switchd 重启，流量中断 30s-2min，必须维护窗口。</details>

5. **AI 集群路由汇总三原则？**
   <details><summary>答</summary>① spine 汇总 leaf 不汇总；② 按 Rail 维度，不跨 Rail；③ 范围连续，否则黑洞。</details>

6. **ipmc-max 为什么不推荐？**
   <details><summary>答</summary>显示 15000 但官方只验证到 ~10000，边缘特性可能不工作。优先选 ipmc-heavy (8k)。</details>

7. **ecmp-nh-heavy 的两个限制？**
   <details><summary>答</summary>① 需 CL 5.9.2+；② 不支持 warm restart。</details>

8. **Adjacency vs ECMP entries 区别？**
   <details><summary>答</summary>Adjacency = 唯一下一跳（IP+MAC+出口）；ECMP entries = 组数。8-way ECMP 路由 = 1 ECMP entry + 8 Adjacency。</details>

---

## 12. 一句话总结

> `cl-resource-query` 是 AI 网络的**容量体检报告**。任何资源 > 80% 警惕，> 95% 立即调整。
>
> 三个永恒规律：
> 1. **AI 网络 ECMP 永远最先满**（Rail-Optimized 必然结果）
> 2. **EVPN 网络 host/Mcast 永远最先满**（Type-2 + BUM 双重消耗）
> 3. **控制面正常 ≠ 数据面正常**（partial allocation 是沉默杀手）

---

## 参考资料

- [Cumulus Linux 5.12 Resource Diagnostics](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-512/Monitoring-and-Troubleshooting/Resource-Diagnostics/)
- [Forwarding Table Size and Profiles](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-59/Layer-3/Routing/Forwarding-Table-Size-and-Profiles/)
- [ACL Hardware Limitations](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-59/System-Configuration/Access-Control-Lists/Access-Control-List-Configuration/#hardware-limitations-for-acl-rules)
