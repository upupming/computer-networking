# 网络层下

## Internet 路由

Internet 网络非常庞大，因此使用层次路由。AS 内部路由协议也称为内部网络协议 IGP(interior gateway protocols)。最常见的 AS 内部路由协议是『路由信息协议』（RIP(Routing information Protocol)）、『开放最短路径优先』（OSPF(Open Shortest Path First)）和『内部网关路由协议（IGRP(Interior Gateway Routing Protocol)）。IGRP 是Cisco 公司的私有协议。

### RIP 协议

RIP 协议早于 1982 年随 BSD-UNIX 操作系统发布。根据距离向量路由算法，计算最短路由。回顾一下：

+ 距离度量用跳步数表示，每条链路 1 个跳步
+ 每隔 30s，邻居之间交换一次 DV，即通告（advertisement）
+ 每次通告最多 25 个目的子网（IP 地址形式）

比如某个路由器的路由信息（跳步数）可能是：

![](https://i.loli.net/2018/10/22/5bcdeaa8e30df.png)

例子：

![](https://i.loli.net/2018/10/22/5bcdedf4778ab.png)

A, B, C, D 四个路由器连接了一些子网，这个网络中的路由器 D 的路由表可能是这样的（这里多了下一跳 hops）：

![](https://i.loli.net/2018/10/22/5bcdeec0401c9.png)

如果路由器 A, D 之间相互交换距离向量，D 将会对路由表进行更新，如果说 A 到 z 的跳步数为 4，因为 D 到 A 的跳步数为 1，那么 D 的路由表会更新为：

![](https://i.loli.net/2018/10/22/5bcdf045b4705.png)

这就是距离向量算法的计算过程，略有不同（多了下一跳）。

#### 链路失效、恢复

如果 180 秒没有收到通告，那么就认为邻居链路失效，经过该邻居的路由不可用，因此需要重新计算路由。在这之后又是同样的过程：向邻居发送新的通告，邻居如果转发表改变的话再依次向外发送通告。

那么链路失效信息能否快速传播到全网呢？不一定，可能会发生无穷计数问题。但可以采用最大有效度量（无穷大距离= 16 hops）来避免产生这个问题。同时还可以采用毒性逆转技术用于预防乒乓环路。

RIP 路由表是利用一个称作 route-id(daemon) 的应用层进程进行管理。通告报文周期性地通过 UDP 数据发送。

![](https://i.loli.net/2018/10/22/5bcdf2af4943f.png)

### OSPF 协议

OSPF 是开放的协议，采用链路状态分组，在链路系统内每个路由器扩散自己的分组（通告），每个路由器构造完整的网络（AS）拓扑图，利用 Dijkstra 算法计算路由。

OSPF 通告中每个入口对应一个邻居，在整个 AS 范围泛洪，其 OSPF 保文直接封装到 IP 数据报中。

与 OSPF 及其相似的一个路由协议是 IS-SI 路由协议。

与 RIP 协议相比，有许多优点：

+ 每个 OSPF 保文在被认证后，才会使用，因此更加安全
+ 允许使用多条相同费用的路径，负载均衡
+ 对于每条链路，可以针对不同的 TOS 设置多个不同的费用度量
+ 集成单播路由与多播路由
+ 支持对大规模 AS 再次进行分层

#### 分层的 OSPF 协议

两级分层的 OSPF 网络包括局部区（Area）和主干区（Backbone）：

+ 链路状态通告只限于区内
+ 每个路由器掌握所在区的详细拓扑
+ 只知道去往其他区网络的方向

![](https://i.loli.net/2018/10/23/5bcdf69376f7c.png)

曲边界路由器（Area Border Routers）用来汇总到达所在区网络的距离，通告给其他区边界路由器。

![](https://i.loli.net/2018/10/23/5bcdf7b80e78c.png)

主干路由器（Backbone）主要在主干区内运行 OSPF 路由算法。

![](https://i.loli.net/2018/10/23/5bcdf80ac687f.png)

AS 边界路由器（AS boundary routers）用来连接其他 AS。

![](https://i.loli.net/2018/10/23/5bcdf87262677.png)

### BGP 协议

边界网关协议（BGP(Border Gateway Protocol)）是事实上的标准域间路由协议，是将 Internet “粘合”为一个整体的关键。

BGP 为每个 AS 提供了一种手段，分为两种：

+ eBGP：从邻居 AS 获取子网可达性信息
+ iBGP：向所有 AS 内部路由器传播子网可达性信息

基于可达性信息与策略，我们可以确定到达其他网络的好路径。

BGP 协议的关键在于『允许子网向 Internet 其余部分通告它的存在』。

BGP 会话（session）主要用于两个 BGP 路由器（Peers）之间交换 BGP 报文，通告去往不同目的前缀（prefix）的路径。

BGP 报文有以下几种：

+ OPEN: 与 peer 建立 TCP 连接，并认证发送方
+ UPDATE: 通告新路径或撤销原路径
+ KEEPALIVE: 在无 UPDATE 时，保活连接；也用于对 OPEN 请求的确认
+ NOTIFICATION: 报告先前报文的差错；也被用于关闭连接

在下面的网络结构中：

![](https://i.loli.net/2018/10/23/5bcdfc51a59c4.png)

当 AS3 通告一个前缀给 AS1 时，承诺可以将数据报转发给该子网，在通告中会聚合网络前缀。

在 3a 与 1c 之间，AS3 利用 eBGP 会话向 AS1 发送前缀可达性信息。1c 则可以利用 iBGP 向 AS1 内的所有路由器分发新的前缀可达性信息。1b （也可能不）可以进一步通过 1b- 到 2a- 的 eBGP 会话，向 AS2 通告新的可达性信息。

当路由器获得新的前缀可达性时，即在其转发表中增加关于该前缀的入口（路由项）。

通告的前缀信息包括 BGP 属性，即『前缀 + 属性 = 路由』。

BGP 有下面两个重要属性：

+ AS-PATH(AS 路径)
    
    包含前缀通告所经过的 AS 序列，例如：AS 67, AS 17

+ NEXT-HOP（下一跳）

    一个 AS-PATH 的路由器接口指向的下一跳 AS，可能从当前 AS 到下一跳 AS 存在多条链路


网关路由器收到路由通告后，利用其输入策略（import policy）决策接受或拒绝该路由。比如某个 AS 从不将流量路由到 AS x，因此 BGP 路由也被称为基于策略（policy-based）的路由。


路由器可能获知到达某目的 AS 的多条路由，基于以下准则选择最终使用的路由：

+ 依靠本地偏好（preference）值属性进行策略决策（policy decision）
+ 选择最短的 AS-PATH
+ 选择最近的 NEXT-HOP 路由器，这被称为『热土豆路由』
+ 其他的附加准则

在下面的图中：

![](https://i.loli.net/2018/10/23/5bce681ec3878.png)

+ A, B, C 是『提供商网络/AS』（provider network/AS）
+ x, w, y 是『客户网络/AS』（customer network/AS）
+ w, y 是『桩网络/AS』（stub network/AS），因为只与一个其他 AS 相连
+ x 是『双宿网络/AS』（dual-homed network/AS），因为和两个其他 AS 相连
    + 如果 x 不期望经过它路由 B 到 C 的流量，x 不会向 B 通告任何一条到达 C 的路由

应用路由选择策略选择路由，A 向 B 通告一条路径 Aw，B 向 X 通告路径 BAw，那么 B 是否应该向 C  通告路径 BAw 呢？答案是【不】，因为 w 和 c 均不是 B 的客户，B 路由 CBAw 的流量没有任何『收益』。B 期望强制 C 通过 A 向 w 路由流量，期望只路由去往/来自其客户的流量。

那么为什么要采用不同的 AS 内与 AS 间路由协议呢？主要考虑了以下几点：

+ 策略（policy）
  + inter-AS 期望能够管理控制流量如何被路由，谁路由经过其网络等
  + intra-AS 期望单一管理，无需策略决策
+ 规模（scale）
  + 层次路由节省路由表大小，减少路由更新流量
  + 适应大规模互联网
+ 性能（performance）
  + intra-AS 侧重性能
  + inter-AS 策略主导