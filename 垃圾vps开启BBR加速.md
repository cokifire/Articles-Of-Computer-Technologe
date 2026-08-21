BBR 是 Google 在 2016 年提出的一种新型 TCP 拥塞控制算法，用于替代传统的 Cubic / Reno 等算法。
传统算法的核心逻辑是：
 通过“丢包”判断网络是否拥塞
 
 丢包后主动降速
 
BBR 的核心思想是：
主动测量链路的真实带宽（Bandwidth）和往返延迟（RTT）
 
 在“刚好跑满带宽、但不制造队列拥塞”的状态下传输数据
 
在 VPS 使用场景中，BBR 能改善以下问题：
1. 跨国网络慢、延迟高
例如：
 中国 ↔ 美国 / 日本 / 新加坡
 
 高 RTT（150ms～300ms）
 
BBR 对 高延迟链路 非常友好。
2. 晚高峰速度骤降
传统 TCP 在丢包多的时段会严重降速，BBR 更稳定。
3. 小带宽 VPS 跑不满
如 1Gbps 网卡、但 TCP 实际只有几十 Mbps，BBR 通常能明显改善。
 注意： BBR 不是“翻倍神器”，而是“更合理地跑满你本来就有的带宽”。
 
启用 BBR 的基本条件
1. 内核要求
 Linux 内核 ≥ 4.9
 
 推荐 5.x / 6.x
 
常见系统支持情况：
 Ubuntu 18.04+ ✔
 
 Debian 9+ ✔
 
 CentOS 7（需升级内核）
 
 Alma / Rocky Linux ✔
 
2. VPS 必须是 KVM / 裸金属
 KVM / XEN ✔
 
 OpenVZ（老版本） ✘（无法开启）
 
BBR 的副作用和误区
1. 并非所有线路都提升
 CN2 GIA、优化线路：提升不明显
 
 本地低延迟网络：效果有限
 
2. 对“极端丢包”线路无能为力
BBR ≠ 网络质量修复器
3. 与部分 QoS / 限速策略冲突
极少数 VPS 商家会对 BBR 流量限速
下面是操作流程：
一、先确认内核是否支持 BBR（关键一步）
在 VPS 上执行：
uname -r

只要内核版本是 4.9 及以上，就可以直接开启。
Ubuntu 18.04 常见情况：
4.15.x ✅（默认支持 BBR）
 
5.x / 6.x ✅
 
如果是 4.15.xxx-generic，无需升级内核。
二、一步开启 BBR（推荐做法）
1️⃣ 写入 sysctl 参数
执行：
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf

2️⃣ 立即生效
sysctl -p

三、验证 BBR 是否真的生效（一定要做）
1️⃣ 查看拥塞控制算法
sysctl net.ipv4.tcp_congestion_control

正确结果应为：
net.ipv4.tcp_congestion_control = bbr
 
2️⃣ 确认 BBR 模块已加载
lsmod | grep bbr

看到类似输出即正常：
tcp_bbr

四、针对小内存的额外优化建议（很重要）
如果内存非常紧张，建议顺手做以下 不增加风险的优化：
1️⃣ 调低 TCP 缓冲区（防止吃内存）
cat >> /etc/sysctl.conf << EOF
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 65536 4194304
net.core.rmem_max = 4194304
net.core.wmem_max = 4194304
EOF

然后：
sysctl -p

2️⃣ 建议开启 swap（如果没开）
小内存不开 swap 很容易 OOM，尤其跑代理或 Web 服务。
fallocate -l 512M /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
 
