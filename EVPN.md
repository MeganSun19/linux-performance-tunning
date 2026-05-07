# RFC 7432 BGP MPLS-Based Ethernet VPN

##  基础：
1. 核心目标：解决VPLS在multi-homing,多路径负载均衡和配置简化的限制。
2. 控制平面学习：不同于VPLS在数据平面学习MAC地址，EVPN在控制平面学习

3. 几个重要概念：
ES: ethernet segment: CE连到PE的链路
ESI: Ethernet segment identifier, 唯一标识ES的10字节非0整数。ESI 0 denotes a single-homed site.
EVI: An EVPN instance spanning the Provider Edge (PE) devices participating in that EVPN.
MAC-VRF: A Virtual Routing and Forwarding table for Media Access Control (MAC) addresses on a PE.
type 1 - Ethernet Auto-Discovery (A-D) route
type 2 - MAC/IP Advertisement route
type 3 - Inclusive Multicast Ethernet Tag route    (BUM traffic)
type 4 - Ethernet Segment route    (选举DF)

ESI Label Extended Community： multihoming的时候用作split horizon.

4. Multihoming Functions
- 自动发现 (Auto-discovery)：PE 通过交换以太网段路由，自动发现连接到同一 ES 的其他 PE。
- 快速收敛 (Fast Convergence)：当 PE 到 ES 的连接失效时，PE 只需撤销 以太网 A-D 路由，即可触发远程 PE 更新所有关联该 ES 的 MAC 地址转发路径，而无需逐条撤销 MAC 路由。
    - EVPN 中，MAC 地址是通过 BGP 控制平面学习并发布的。当一个多归属的以太网段（Ethernet Segment, ES）发生故障（例如 PE 到 CE 之间的链路失效）时，如果按照传统方式逐条撤销受影响的 MAC/IP 公告路由，收敛时间将取决于路由的数量。在拥有成千上万个 MAC 地址的高规模环境中，这种逐条撤销的方式会导致收敛速度极慢。
为了实现快速收敛，EVPN 引入了“以太网自动发现路由”，特别是 “每以太网段 A-D 路由” (Ethernet A-D per ES route)。当 PE 检测到其与多归属 CE 之间的连接失效时，它会立即在 BGP 中撤销对应以太网段的 A-D 路由。远程 PE 收到撤销通知后，无需等待成百上千条具体的 MAC 路由被逐一撤销，而是直接更新其下一跳邻接关系。
* 对于 PE 之间的网络节点或链路故障，EVPN 依赖于现有的 MPLS 快速重路由 (Fast Reroute) 机制来实现 50 毫秒级的恢复
- 水平分割 (Split Horizon)：为防止全活动（All-Active）模式下的 BUM 流量回环，PE 使用 ESI 标签。接收端 PE 若发现数据包带有的 ESI 标签与入站 ES 匹配，则丢弃该包。
一句话总结：
1. 水平分割的核心概念
当一个 CE（客户边缘设备）通过一个以太网段（ES1）连接到多个 PE（例如 PE1 和 PE2）且处于**全活动（All-Active）**模式时，如果 CE 发送一个 BUM 数据包给非指定转发者（non-DF）PE1，PE1 会将其转发给同一 EVPN 实例（EVI）中的所有其他 PE，包括作为该网段 DF 的 PE2。
问题：如果不加控制，PE2 可能会将该包重新转发回 CE，导致环路。
解决方案：水平分割过滤确保 PE2 在收到该包后，发现它源自同一个 ES，从而禁止将其转发回该 CE。
2. ESI 标签 (ESI Label) 的作用
实现水平分割的关键工具是 ESI 标签。
定义：它是一个标识流量来源以太网段的 MPLS 标签。
发布：所有在全活动模式下的 PE 必须通过“每以太网段 A-D 路由 (Ethernet A-D per ES routes)”发布其关联 ES 的 ESI 标签。
适用性：在全活动模式下是强制的，在单活动模式下也强烈建议使用，以防止故障恢复期间的瞬时环路。
3. 入站复制 (Ingress Replication) 下的具体程序
在入站复制模式下，ESI 标签的分配和转发逻辑如下：
A. 标签分配与通告 (Label Assignment)
下游分配：每个 PE 为其连接的 ES 分配一个下游分配 (Downstream Assigned) 的 ESI 标签。
本地编程：发布该标签的 PE 会在本地平台标签空间中进行编程：当收到带有此标签的报文时，禁止将其转发到分配该标签的那个 ES 上。
B. 入站 PE 的封装规则 (Ingress PE Rules)
当入站 PE 收到来自本地 CE 的 BUM 报文并准备发往远程多归属 PE 时：
非 DF 入站 PE：在发送给该 ES 的 DF 节点时，必须压入由该 DF 发布的 ESI 标签。
通用建议：入站 PE（无论是否为 DF）都应该压入由该 ES 的其他非 DF 节点发布的 ESI 标签。
禁忌：如果目的地 PE 并没有连接到该报文进入 EVI 时的那个源 ES，则入站 PE 绝不能在该副本中包含 ESI 标签。
C. 报文封装结构
在入站复制场景下，报文的 MPLS 标签栈从底部向上依次为：
ESI 标签：位于底部，标识源 ES（由出口 PE 分配）。
EVPN 标签：由出口 PE 在“包含性组播路由”中发布，用于标识 EVI/VLAN。
传输 LSP 标签：用于在核心网中到达目的 PE（如 RSVP-TE 或 LDP 标签）。
D. 出口 PE 的处理与过滤 (Egress PE Processing)
当目的地 PE 收到报文并剥离传输标签后：
识别 EVI：根据顶层的 EVPN 标签确定报文属于哪个实例及需要复制到的 ES 集合。
检查 ESI 标签：
如果下一层标签是该 PE 为源 ES 分配的 ESI 标签，PE 必须禁止将该报文转发到该 ES 上。
如果该 ESI 标签不是该 PE 分配的，则丢弃该包。

Ingress replication: 
    1. 假设PE1, PE2双归到CE1上，PE1为non-DF, PE2 是DF,每个PE都会为其本地连接的网段分配一个本地唯一ESI标签，随后PE2会构造一种特殊的BGP路由，即Ethernet A-D per ES route. 
    2. 在发布上述AD路由时，PE2会在路由中附带一个特定的BGP 扩展团体属性，即ESI Label extended community. 
    3. PE2通过MP-BGP将这条路由发送给邻居，PE1作为同一个EVPN实例：EVI的参与者，会import 这条路由。
    4. PE1接收到PE2发出的Per ES AD route后，会解析其中的ESI标签扩展团体属性，从而知道：如果我要把从ES1进入的BUM流量发给PE2,我必须压入这个标签。
    5. PE1将从CE1收到的BUM流量压入标签发给PE2时，PE2看到这个标签就知道这个包最初是从ES1进来的，因此PE2将不会将其发回给本地的ES1,即CE1.

- 别名 (Aliasing)：在全活动模式下，即使某 PE 尚未学习到本地 MAC 地址，远程 PE 也可以根据 A-D 路由信息将流量负载均衡到连接该 ES 的所有 PE 上。
- 指定转发者 (DF) 选举：使用服务雕刻 (Service Carving) 算法。PE 根据收到的 ES 路由构建有序的 IP 地址列表，并通过 (VLAN ID mod N) 的规则在多台 PE 间平均分配 BUM 流量的转发职责。



