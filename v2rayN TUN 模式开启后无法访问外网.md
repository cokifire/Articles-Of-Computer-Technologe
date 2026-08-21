v2rayN TUN 模式开启后无法访问外网，原因是 Windows TUN + gVisor 在某些条件下会与 Chrome 冲突。
一、v2rayN TUN 官方默认配置
Strict Route：开启
 
Stack：gVisor
 
MTU：9000
 ❗ 但 gVisor 在 TUN 模式下会导致部分 Chrome 版本无法联网
 gVisor 模式与 Chrome 的 socket 特性存在兼容问题。
Chrome 有一个特性：
为性能优化，它会创建大量的 UDP/TCP 并发连接 + QUIC 协议 + 并行 DNS，多为 Raw Socket。
而 v2rayN 的 TUN（模式特别是 gVisor）有这些限制：
1️⃣ gVisor 对某些 Raw socket / advanced socket 调用支持不充分
Chrome 使用的 socket 组合非常激进，gVisor 无法完全模拟，会直接导致：
 DNS query 不返回
 
 socket handshake 阻塞
 
 QUIC 失败
 
2️⃣ Strict Route 开启后，更容易出现 Chrome 断网
Strict Route 开启意味着：
 所有流量必须走 TUN
 
 Chrome 的本地私有连接（例如自动本地 DNS）会被阻断
 
Chrome 又特别依赖系统 DNS，因此 Strict Route + gVisor = 出问题概率最高。
3️⃣ MTU 9000 太大，会导致 Chrome 的 QUIC / TLS 包被截断
这也是一个非常典型的问题。
MTU 9000 是 jumbo frame，一般用于局域网，不适合被代理的 VPN 链路。
Chrome 使用 QUIC + 大包时→ MTU 不匹配 → 直接超时。
三、立刻可用的解决方案（按稳定性从高到低）
✅  1：把 Stack 从 gVisor → system（最稳定）
选项有：
 system
 
 mixed
 
 gvisor
 
✔ 推荐：system（最兼容 Chrome）
Chrome 卡顿、断网等 90% 都是 gVisor 造成的。
system 模式性能略低，但兼容性非常高，基本不会出现不能上网。
✅  2：把 Strict Route 关闭
Strict Route 会导致 Chrome 内部自带的本地连接走不通。
设置改为：
 Strict Route：关闭
 
 Stack：system
 
这一组合 最常用且问题最少。
✅  3：把 MTU 从 9000 → 1400 或 1500
最推荐：
MTU: 1400
Chrome 的 QUIC / TLS 握手对 MTU 非常敏感。
  推荐最佳稳定设置（请这样设置）
Strict Route：关闭
Stack：system
MTU：1400
启用外部监听端口：关闭
启用 IPv6：关闭
