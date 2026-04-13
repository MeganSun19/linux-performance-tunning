# RFC 7432 BGP MPLS-Based Ethernet VPN

## 基础

### 核心目标

解决 VPLS 在 multi-homing、多路径负载均衡和配置简化方面的限制。

### 控制平面学习

不同于 VPLS 在数据平面学习 MAC 地址，EVPN 在**控制平面**学习。

### 重要概念

| 术语 | 全称 | 说明 |
|------|------|------|
| ES | Ethernet Segment | CE 连到 PE 的链路 |
| ESI | Ethernet Segment Identifier | 唯一标识 ES 的 10 字节非 0 整数。ESI 0 表示单归属站点。 |
| EVI | EVPN Instance | 跨越参与该 EVPN 的 PE 设备的 EVPN 实例 |
| MAC-VRF | MAC Virtual Routing and Forwarding | PE 上用于存储 MAC 地址的 VRF 表 |

**EVPN 路由类型：**

| 类型 | 名称 | 用途 |
|------|------|------|
| Type 1 | Ethernet Auto-Discovery (A-D) route | 以太网自动发现路由 |
| Type 2 | MAC/IP Advertisement route | MAC/IP 通告路由 |
| Type 3 | Inclusive Multicast Ethernet Tag route | BUM 流量 |
| Type 4 | Ethernet Segment route | 选举 DF |

> **ESI Label Extended Community**：在 multihoming 场景下用作 split horizon。

---

## Multihoming Functions

### 1. 自动发现（Auto-Discovery）

PE 通过交换以太网段路由，自动发现连接到同一 ES 的其他 PE。

---

### 2. 快速收敛（Fast Convergence）

当 PE 到 ES 的连接失效时，PE 只需撤销**以太网 A-D 路由**，即可触发远程 PE 更新所有关联该 ES 的 MAC 地址转发路径，而无需逐条撤销 MAC 路由。

**背景：**

EVPN 中，MAC 地址是通过 BGP 控制平面学习并发布的。当一个多归属的以太网段（ES）发生故障时，如果按照传统方式逐条撤销受影响的 MAC/IP 公告路由，收敛时间将取决于路由的数量。在拥有成千上万个 MAC 地址的高规模环境中，这种逐条撤销的方式会导致收敛速度极慢。

**解决方案：**

EVPN 引入"以太网自动发现路由"，特别是**每以太网段 A-D 路由（Ethernet A-D per ES route）**。当 PE 检测到其与多归属 CE 之间的连接失效时，它会立即在 BGP 中撤销对应以太网段的 A-D 路由。远程 PE 收到撤销通知后，无需等待成百上千条具体的 MAC 路由被逐一撤销，而是直接更新其下一跳邻接关系。

> 对于 PE 之间的网络节点或链路故障，EVPN 依赖于现有的 MPLS **快速重路由（Fast Reroute）**机制来实现 50 毫秒级的恢复。

---

### 3. 水平分割（Split Horizon）

为防止全活动（All-Active）模式下的 BUM 流量回环，PE 使用 ESI 标签。接收端 PE 若发现数据包带有的 ESI 标签与入站 ES 匹配，则丢弃该包。

#### 3.1 核心概念

一句话总结：

当一个 CE（客户边缘设备）通过一个以太网段（ES1）连接到多个 PE（例如 PE1 和 PE2）且处于**全活动（All-Active）**模式时：

- CE 发送一个 BUM 数据包给 non-DF 的 PE1
- PE1 会将其转发给同一 EVI 中的所有其他 PE，包括作为该网段 DF 的 PE2

**问题：** 如果不加控制，PE2 可能会将该包重新转发回 CE，导致环路。

**解决方案：** 水平分割过滤确保 PE2 在收到该包后，发现它源自同一个 ES，从而禁止将其转发回该 CE。

#### 3.2 ESI 标签（ESI Label）的作用

实现水平分割的关键工具是 **ESI 标签**。

- **定义**：标识流量来源以太网段的 MPLS 标签
- **发布**：所有在全活动模式下的 PE 必须通过"每以太网段 A-D 路由"发布其关联 ES 的 ESI 标签
- **适用性**：在全活动模式下是**强制的**，在单活动模式下也强烈建议使用，以防止故障恢复期间的瞬时环路

#### 3.3 入站复制（Ingress Replication）下的具体程序

##### A. 标签分配与通告（Label Assignment）

- **下游分配**：每个 PE 为其连接的 ES 分配一个下游分配（Downstream Assigned）的 ESI 标签
- **本地编程**：发布该标签的 PE 会在本地平台标签空间中进行编程——当收到带有此标签的报文时，禁止将其转发到分配该标签的那个 ES 上

##### B. 入站 PE 的封装规则（Ingress PE Rules）

当入站 PE 收到来自本地 CE 的 BUM 报文并准备发往远程多归属 PE 时：

- **非 DF 入站 PE**：在发送给该 ES 的 DF 节点时，必须压入由该 DF 发布的 ESI 标签
- **通用建议**：入站 PE（无论是否为 DF）都应该压入由该 ES 的其他非 DF 节点发布的 ESI 标签
- **禁忌**：如果目的地 PE 并没有连接到该报文进入 EVI 时的那个源 ES，则入站 PE 绝不能在该副本中包含 ESI 标签

##### C. 报文封装结构

在入站复制场景下，报文的 MPLS 标签栈从底部向上依次为：

| 位置 | 标签 | 说明 |
|------|------|------|
| 底部 | ESI 标签 | 标识源 ES（由出口 PE 分配） |
| 中间 | EVPN 标签 | 由出口 PE 在"包含性组播路由"中发布，用于标识 EVI/VLAN |
| 顶部 | 传输 LSP 标签 | 用于在核心网中到达目的 PE（如 RSVP-TE 或 LDP 标签） |

##### D. 出口 PE 的处理与过滤（Egress PE Processing）

当目的地 PE 收到报文并剥离传输标签后：

1. **识别 EVI**：根据顶层的 EVPN 标签确定报文属于哪个实例及需要复制到的 ES 集合
2. **检查 ESI 标签**：
   - 如果下一层标签是该 PE 为源 ES 分配的 ESI 标签 → **禁止**将该报文转发到该 ES 上
   - 如果该 ESI 标签不是该 PE 分配的 → **丢弃**该包

#### 3.4 Ingress Replication 流程示例

> **场景**：PE1 和 PE2 双归到 CE1，PE1 为 non-DF，PE2 为 DF

1. 每个 PE 都会为其本地连接的网段分配一个本地唯一 ESI 标签，PE2 随后构造一条特殊的 BGP 路由，即 **Ethernet A-D per ES route**
2. 在发布上述 A-D 路由时，PE2 会在路由中附带一个特定的 BGP 扩展团体属性，即 **ESI Label Extended Community**
3. PE2 通过 MP-BGP 将这条路由发送给邻居，PE1 作为同一 EVI 的参与者，会 import 这条路由
4. PE1 接收到 PE2 发出的 Per ES A-D route 后，解析其中的 ESI 标签扩展团体属性，从而得知：如果要把从 ES1 进入的 BUM 流量发给 PE2，必须压入这个标签
5. PE1 将从 CE1 收到的 BUM 流量压入标签发给 PE2 时，PE2 看到这个标签就知道这个包最初是从 ES1 进来的，因此 PE2 **不会**将其发回给本地的 ES1（即 CE1）

---

### 4. 别名（Aliasing）

在全活动模式下，即使某 PE 尚未学习到本地 MAC 地址，远程 PE 也可以根据 A-D 路由信息将流量**负载均衡**到连接该 ES 的所有 PE 上。


---

### 5. 指定转发者（DF）选举

使用**服务雕刻（Service Carving）**算法。PE 根据收到的 ES 路由构建有序的 IP 地址列表，并通过 `(VLAN ID mod N)` 的规则在多台 PE 间平均分配 BUM 流量的转发职责。
