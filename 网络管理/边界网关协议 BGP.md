# 什么是BGP

 BGP是边界网关协议，用于在不同的AS（自治系统）之间传递路由，分为eBGP和iBGP.
 AS: AS是一个自治系统，由同一组织统一管理的一组网络，对外以一个整体为单位，同一个AS拥有相同的路由策略

eBGP：在不同AS之间传递路由
iBGP：把从外部学到的路由传递到内部

## BGP状态机

 BGP有6个状态：
 - Idle: 在这个状态下，路由器拒绝所有链接，做一些初始化操作，监听Tcp 174端口，重置connectretry计时器
 - Connect：在这个状态中，开始等待TCP3次握手完成
 - Active：如果TCP三次握手没有成功，就会来到Active状态，在这个状态下开始尝试重连TCP
 - ConnectSent：如果TCP三次握手成功，就会来到这个状态，这个状态会发送Open消息到对端路由器
 - ConnectConfirm：如果收到对端路由器的open消息，那么就会来到这个状态，并且接收keepalive消息
 - established：收到对端路由器发来的keepalive消息后，连接正式建立

BGP的4种消息类型：
- Open：建立邻里关系
- Update：更新路由信息
- Keepalive：保持连接
- Notification：错误通知，连接断开通知，用于致命错误

## BGP路由选择中的13条规则

所有的路由规则只有在路由前缀相同的时候才会执行策略，前缀不同的时候不需要

下面的规则按照优先级，大致可以分为四块 `管理员设定`，`路由质量`，`稳定性和开销`,`兜底策略`：
1. **weight**，管理员设定，只在本地路由器中起效，权重越高，优先级越高
2. **local preference**，管理员设定，前面的weight作用于单台路由器，local pref作用于整个AS，决定当前AS从哪个出口连接到外部网络，数值越大，优先级预告
3. 本地始发路由，他会判断当前路由是否是本地自己生成，自己本地生成优先级要高于从外部学习来的
4. AS-Path，AS-Path表示路由经过的AS数量，越小则越近，反之就越远，选择近的
5. Origin，起源路由，有三个类型，IGP，EGP，InComplete，其中优先级是IGP>EGP>InComplete
6. MED：这个跟local pref相对应，但是不同的是它决定外部进来的流量走那个入口，med数值越小优先级越高
7. eBGP和iBGP的选择，eBGP的优先级大于iBGP, 因为iBGP是在将外部学习到的路由传递到AS内，所以会绕路，有eBGP能通向外部则优先选择eBGP
8. 下一跳的消耗，比较在AS内部传递到出口前的消耗，消耗越小优先级越高
9. 优先选择老的eBGP，因为老的已经工作了一段时间，表明他更稳定
10. route id，这个是兜底策略，route i越小的优先级越高
11. cluster list长度，这个在拥有中转路由器的网络中，经过的中转路由器数量越少，优先级越高
12. 领居IP地址越小的优先级越高
13. BGP path ID：数值越小越高

## BGP 故障排查

| State             | Desc                    | Issue                   |
| ----------------- | ----------------------- | ----------------------- |
| Idle              | 无任何连接尝试                 | BGP没有启动、接口Down          |
| Connect           | 正在等带TCP三次握手             |                         |
| Active            |                         | 说明TCP三次握手没有成功、也可能是防火墙组织 |
| OpenSent          | 发送Open消息，等待对方回的Open     | 可能是对方发的Open没有收到         |
| OpenConfirm       | 收到对方的Open，准备接受KeepAlive | 可能是没有收到Keepalive        |
| Established->Idle | 状态抖动                    | 连接不稳定，keepalive 丢包      |
