# Clos 架构
leaf-spine 是clos的工业落地，但clos并不一定都指leaf-spine.
5-Stage Clos（也就是带 Super-Spine 的架构，Facebook/Meta 就在用）通常被称为 "Clos Network" 或 "Data Center Fabric"，虽然它也是由 Leaf 和 Spine 组成的，但当层级超过 3 级（物理 2 层）时，人们往往倾向于回归叫它 "Clos" 或 "Multi-tier Clos"，而狭义的 "Leaf-Spine" 通常指两层物理结构的 3-Stage 架构。
## 1. 核心特点

- **扁平化与高带宽**：相比传统三层架构，Clos 更平坦（Spine-Leaf 全互联），适合分布式存储等以东西向流量为主的场景。
- **水平扩展**：
  - 增加 Spine 交换机数量可提高核心带宽。
  - 增加 Leaf 交换机数量可扩大接入规模。
- **Non-blocking / 低延迟**：基于全互联和 ECMP，多路径转发可实现任意两点高速传输；典型 Leaf 间服务器通信约为 3 跳。

---

## 2. 关键概念

### 2.1 Leaf Switch

- 类似传统三层中的接入交换机，通常作为 ToR 直接连接物理服务器。
- 与传统接入交换机的关键差异：**L2/L3 分界点在 Leaf**，Leaf 之上通常是纯三层网络。

### 2.2 Spine Switch

- 类似核心交换机。
- Leaf 与 Spine 之间通过 ECMP 进行多路径转发。
- Spine 的角色更偏向“弹性 L3 路由骨干”，而不是传统意义上的汇聚/核心二层扩展。

### 2.3 VXLAN

- 报文封装：**L2 over L4（MAC-in-UDP）**。
- 价值：在三层网络上扩展二层，满足大二层迁移、多租户隔离等需求。

### 2.4 现代数据中心组合方式

- 路由/转发层面使用 Clos。
- 二层虚拟机网络通过 VXLAN 进行广播域隔离。
- 同时兼顾三层组网灵活性，避免二层环路与广播风暴。

### 2.5 运营商网络 vs 数据中心网络

- 运营商网络优先级通常是：可靠性优先，其次才是快速感知。
- 数据中心更强调：**路由快速收敛**。
- 经典 BGP 在多路径上默认只选最优单路径；而数据中心希望更多等价路径并行，需要相应策略与实现增强。

### 2.6 数据中心为何常用 eBGP

- 数据中心里常用 eBGP（而不是 iBGP）进行 Clos 内部路由，配合更激进计时器达到快速收敛。
- 典型建议：
  - `advertisement-interval`：`0s`
  - `keepalive` / `hold`：`3s / 9s`
  - 开启邻接变化日志（adjacency changes logging）
  - 开启 eBGP / iBGP multipath

> 注：默认 eBGP `advertisement-interval` 常见为 30s，而 iBGP 常为 0s。对密集连接型 DC 来说，30s 过长。
- BGP 可以实现per-hop TE: Use for unequal-cost Anycast load-balancing solution
  即：每个router/switch都可以根据自己的bgp policy独立选择next-hop.
  传统的IGP选路只会通过metric,全网统一。BGP可以通过LF,MED,AS_PATH等自己决定去往同一个prefx走哪条路。
  数据中心中的anycast 服务架构：
  假设DNS service遍布在A,B,C三个机架，分属leaf1,leaf2,leaf3.所有服务器都annouce 10.10.10.10/32
           Spine
       /   |   \
    Leaf1 Leaf2 Leaf3
      |      |      |
   DNS-A   DNS-B  DNS-C

   BGP anycast的作法：可以通过LP, Leaf1走dns-a...
   e,g: anycase LB, 通过BGP local-pref, med, community可以实现30% traffic --- SiteA, 50% traffic--- SiteB, 20% traffic --- SiteC，这就是uequal-cost load balancing


   
---

## 3. RFC 7938：Use of BGP for Routing in Large-Scale Data Centers

### 3.1 Clos Topology（Traffic Engineering）

- 通过 Anycast + ECMP 实现流量工程。
- 可通过协议扩展显式控制某前缀的下一跳。

### 3.2 3-Stage Clos 拓扑

路径示意：`Leaf > Spine > Leaf`

```text
   +-------+
   |       |----------------------------+
   |       |------------------+         |
   |       |--------+         |         |
   +-------+        |         |         |
   +-------+        |         |         |
   |       |--------+---------+-------+ |
   |       |--------+-------+ |       | |
   |       |------+ |       | |       | |
   +-------+      | |       | |       | |
   +-------+      | |       | |       | |
   |       |------+-+-------+-+-----+ | |
   |       |------+-+-----+ | |     | | |
   |       |----+ | |     | | |     | | |
   +-------+    | | |     | | |   ---------> M links
    Tier 1      | | |     | | |     | | |
              +-------+ +-------+ +-------+
              |       | |       | |       |
              |       | |       | |       | Tier 2
              |       | |       | |       |
              +-------+ +-------+ +-------+
                | | |     | | |     | | |
                | | |     | | |   ---------> N Links
                | | |     | | |     | | |
                O O O     O O O     O O O   Servers

                  Figure 2: 3-Stage Folded Clos Topology
```

### 3.3 Oversubscription（超配）

#### 1) 什么情况下叫 Oversubscribed？

在 Clos 拓扑中，若某层交换机下行带宽大于上行带宽，即为 Oversubscribed。

以 Tier 2（Leaf）为例：

- `N`（Downlink）：连接下游服务器/低层设备端口数
- `M`（Uplink）：连接上层 Tier 1（Spine）端口数

判断：

- `M >= N`：Fully Non-blocking / Non-interfering
- `M < N`：Oversubscribed

#### 2) 超配因子是 `N/M` 还是 `M/N`？

根据文档定义：**超配因子 = `N/M`**。

示例：`N=48`、`M=16`，超配比为 `48/16=3`，即 `3:1`。

#### 3) 为什么引入超配？

- 在固定端口预算下提升服务器密度。
- 成本优化：并非所有服务器都会持续满带宽，适度超配可降低上层设备与链路成本。

> 实务上很难做到全网完全 non-blocking。以 48 口交换机为例，若追求严格 1:1 上下行，可能只能接约 24 台服务器。

### 3.4 5-Stage Clos 拓扑

路径示意：`Leaf > Spine > Super Spine > Spine > Leaf`

---

## 4. DC Routing Overview（eBGP 视角）

### 4.1 eBGP 的工程优势

1. 相对 IGP，复杂度更低：状态机与数据结构更简单，依赖 TCP 传输。
2. 相对链路状态协议，泛洪开销更小，且无需周期性全域刷新。
3. 支持 third-party（递归解析）next-hop，便于与控制器协同实现流量工程。
4. 支持按层级规划 ASN，适配大规模 Clos。

### 4.2 常见 ASN 规划

- Tier 1 可共享 ASN。
- 每个 Tier 2 集群可分配唯一 ASN。
- 每台 Tier 3（ToR）可使用独立私有 ASN。
- 常见私有 ASN 范围：`64512-65534`（共 1023 个）。
- 大规模场景常会跨 Cluster 复用部分 ASN。

### 4.2.1 一个真实的 Fabric ASN 规划例子

例如一个 hyperscale DC：

- Spine ASN: 65000
- Leaf ASN:
  - 65100-65150
- ToR ASN:
  - 65200-65999

WAN edge policy：

```text
remove private AS
neighbor WAN remove-private-as
allow fabric routes
ip as-path access-list 10 permit ^65
deny others
ip as-path access-list 10 deny .*
```

就结束了。

### 4.3 邻居建立建议

- 使用基于直连点到点链路的 **eBGP single-hop**。
- 即使同设备间有多链路，也尽量不做 loopback/multihop 方式。

### 4.4 `allow-as-in` vs `as-override`

- **allow-as-in**（接收端）：即使 AS_PATH 中包含自身 ASN，也允许接收。
- **as-override**（发送端）：发送时若发现对端 ASN 出现在路径中，用本地 ASN 替换。

文档倾向：**优先使用 `allow-as-in`**。

补充理解：

- Tier 1 通常仍保留严格防环，不接收包含自身 ASN 的路由。
- Tier 2 之间若无直连，环路风险相对可控。

---

## 5. Prefix Advertisement

1. Clos 中点到点链路很多，若全部明细通告，易导致 FIB 压力上升、BGP 计算负担变大。
2. 推荐对基础设施链路前缀做可控汇总：例如给 Tier 1/Tier 2 下行接口分配连续地址块，以便聚合通告。
3. **Leaf 与 ToR 之间通常需要明细通告**，不宜粗粒度汇总，否则可能产生黑洞。
4. **Tier 3（ToR）服务器网段在经过 Tier 2/Tier 1 时不能汇总**：
   - 否则当 Tier 2 到某 ToR 链路故障时，汇总路由仍可能继续上告，流量到达后无路可走。

---

## 6. External Connectivity

### 6.1 Border Leaf 与 WAN 交互

- 向 WAN 通告时，通常需要移除私有 ASN，避免跨 DC ASN 冲突。
- 也有助于 WAN 侧针对 Anycast 前缀实现一致的 AS_PATH 长度与 ECMP 行为。

### 6.2 默认路由来源

- Border Leaf 往往是 DC 内唯一可引入/产生默认路由的位置。
- 常见做法：透传从 WAN 学到的默认路由。
- 为降低单链路故障影响，建议所有 Border Leaf 与上游 WAN 路由器形成充分连接关系。

---

## 7. Route Summarization at the Edge（边缘汇总）

由于每台 Tier 3 都会通告服务器网段，对外前缀可能非常多，边缘汇总可减轻外部网络负担。

但在标准 Clos（同层无 peer links）中，边缘汇总在单链路故障下有黑洞风险。常见缓解方案：

1. **互联 Border Routers**：边界路由器之间增加互联（如 full-mesh）。
2. **增加 Tier 2 到 Border 的冗余连接**：每个 Tier 1 至少连两台 Border。

两种方案都存在成本与复杂度上的 trade-off。

---

## 8. ECMP

### 8.1 基础 ECMP

- Clos 中低层设备通常对同一前缀使用所有上行等价路径做负载分担。
- 任意两台 Tier 3（ToR）之间的等价路径数，通常等于中间层（如 Spine）设备数量。
- 需要足够大的多路径扇出能力（例如 64 口设备场景常要求 32 路 ECMP）。

#### 扇出（Fan-out）直观解释

以 64 口交换机为例：

- 32 口下行，32 口上行。
- 若转发到某目的前缀可用 32 条等价上行路径，则设备应支持 32 路 ECMP。
- 若硬件仅支持 8/16 路，则其余链路无法进入 ECMP 组，带宽利用会受限。

#### 分层 ECMP

- 若硬件扇出不足，可结合 LAG 做“分层 ECMP”。
- 代价是可能增加流量极化（polarization）风险。

#### 路径等价判定

- BGP 通常基于 MED、Origin 等属性判定路径是否可等价。
- 纯三层 Clos 无 IGP 度量差异，IGP cost 常被视作等价。

> BGP 常见选路顺序可回顾：`weight`、`local-preference`、本地起源、`AS_PATH` 长度、`Origin`、`MED`、eBGP 优先于 iBGP 等。

### 8.2 跨 ASN 的 BGP ECMP（Multipath Relax）

应用负载均衡常通过在多个 ToR 通告相同 Anycast 前缀实现。

挑战：不同集群 ASN 不同，导致 AS_PATH 值不同（但长度可相同）。

解决：启用 **multipath relax / multiple-AS multipath**。

Cisco 常见命令：

```bash
bgp bestpath as-path multipath-relax
```

启用后可忽略 ASN 编号差异，仅依据路径长度等条件参与多路径。

### 8.3 加权 ECMP（Weighted ECMP）

适用于非等比例流量分配需求，例如：

- 链路故障后按剩余容量重分配；
- 按路径实际带宽进行差异化引流。

常见实现：由“中央代理/控制器”通过多跳 BGP 注入特定前缀与属性（如 Link Bandwidth Extended Community），设备据此调整哈希比例。

### 8.4 一致性哈希（Consistent Hashing）

目标：当 ECMP 成员变化（增删下一跳）时，最小化已有流重映射。

- 普通 `mod N` 哈希在链路数量变化时会重排大量流。
- 一致性哈希可让多数健康路径上的既有流保持不变，仅重分配受影响路径上的流。
- 对有状态设备（防火墙、会话追踪）尤为关键。
- 代价：通常消耗更多硬件资源（如 TCAM）。

---

## 9. 路由收敛特性

### 9.1 故障检测时间（Fault Detection Timing）

由于 Clos 内部常不依赖 IGP，BGP 需通过自身机制快速感知故障：

- **Fast Fallover**：链路 down 时立即关闭相应 eBGP 会话，可达毫秒级触发。
- **替代方式**：BFD / Ethernet CFM，可实现亚秒级检测。
- **不推荐**：仅依赖 BGP Keepalive/Hold 超时（最短常见 3s），收敛较慢。

### 9.2 扇出对收敛的影响（Update Dispersion）

大扇出 Clos 中，撤销更新可能分批到达，导致短暂不稳定。

建议：

- 支持并使用 **Update Groups**，对策略一致邻居同步下发更新。
- **不建议开启 Route Flap Dampening**，复杂收敛中易误伤正常路由。

### 9.3 故障影响范围（Failure Scope）

BGP 的距离矢量特性有利于故障域收敛控制：

- 若本地可立即切到备份路径，故障可被本地“屏蔽”，无需全网扩散。
- Tier 1/2 间故障通常主要影响相关层级，不一定波及所有 ToR。
- 配合分层 FIB（如 BGP PIC），可通过修改下一跳指针快速影响大量前缀。

最坏情况：为避免黑洞，Clos 内部禁止某些关键汇总时，Tier 2/3 链路故障可能仍扩散全网。

---

## 10. BGP PIC prefix independent convergence 前缀无关收敛

即提前计算出一条backup路径，无需等到某链路或节点失效。可以实现毫秒级收敛，但需要配合BFD等检测出故障。

1. 核心原理：层级化的FIB. 在传统的非层级化的FIB中，如果下一跳失效，系统必须遍历并修改所有依赖下一跳的prefix,可能有上万条，导致收敛时间长。

  而使用BGP PIC,只需要修改下一跳指针。将其指向预先计算好的备份路径，这通常储存在硬件FIB中。

  分离存储：前缀查找表与具体的下一跳转发信息是分离存储的。

  ```text
  BGP ---> RIB ---> FIB
  ```

2. 故障检测与触发机制

  BGP PIC 能够处理两种类型的故障：

  - 核心故障 (Core Failure)：通常是 iBGP 节点或链路失效。它依赖于 IGP 的收敛来通告失效，RIB 接收到 IGP 更新后触发 FIB 切换。
  - 边缘故障 (Edge Failure)：通常是直接连接的 eBGP 邻居或链路失效。

  为了实现亚秒级检测，必须配合 BFD (双向转发检测)。

  CEF 会直接监听 BFD 事件，一旦检测到链路 down，立即在数据平面执行“就地修改（in-place modification）”，切换至备份路径。

3. 多宿主前提：BGP PIC 要求目的地必须有多个路径（Multihomed）可达，且备份路径的下一跳必须与主路径不同。

4. 命令：

  ```bash
  # bgp additional-paths install.
  #  maximum-paths 64
  # bestpath as-path multipath-relax
  ```

  * BGP multipath是指多路径/ECMP，一般是等价的。如果使用了multipath，将不需要BGP PIC，已经存在多路径了，但他俩侧重点不同，#  maximum-paths 64

  前面还提到一个概念： # bestpath as-path multipath-relax, 允许BGP在AS_PATH长度相同但asn不同的路径之间负载均衡。

