
一、问题现象
环境：
Oracle Cloud VPS
Ubuntu 22.04
Node.js
Vite 开发服务器，端口3000
React
程序启动正常，本地浏览器访问
http://vps公网IP:3000
一直无法打开。
二、第一步：确认程序是否正常
查看监听端口
sudo ss -tlnp
结果：
LISTEN 0.0.0.0:3000
users:(("node",pid=29326))
说明：
Node 已启动
已监听所有网卡
不是监听 localhost
查看 Node 进程
ps -ef | grep node
结果：
vite --host 0.0.0.0 --port 3000
说明：
Vite 配置正常。
三、确认 Ubuntu 防火墙
sudo ufw status
结果：
Status: inactive
说明：
UFW 未启用，不是ufw封堵了端口。
四、确认 Oracle Cloud 网络
查看 Primary VNIC
确认：
Public IP 已绑定
Private IP 正常
已绑定对应的 NSG
查看 Network Security Group
确认规则已经存在，如下：
Ingress

Source Type
CIDR

Source
0.0.0.0/0

Protocol
TCP

Destination Port
3000
Security List
确认 Subnet 已关联 Security List，且有如下 Ingress Rule (入站规则):
Ingress

Source Type
CIDR

Source
0.0.0.0/0

Protocol
TCP

Destination Port
3000
五、检查 iptables
查看：
sudo iptables -L -n -v
发现：
Chain INPUT

ACCEPT RELATED,ESTABLISHED

ACCEPT icmp

ACCEPT lo

ACCEPT tcp dpt:22

REJECT all reject-with icmp-host-prohibited
最关键的一条：
REJECT all
意味着：
除了：
SSH
localhost
已建立连接
其它全部拒绝。
在 Oracle Cloud 的 Ubuntu 镜像中，默认不推荐直接开启 ufw。
原因如下：
自带 iptables： Oracle 的镜像预装了一套非常严格的 iptables 规则。
容易失联： 如果你直接运行 sudo ufw enable，而没有先执行 sudo ufw allow ssh，你可能会立即被切断远程连接 (SSH)，再也连不上服务器。
规则冲突： ufw 本质上是 iptables 的一个前端。在 Oracle 环境下，手动开启 ufw 可能会与系统原有的 iptables 规则冲突，导致即便你在网页控制台开了端口，依然访问不了。

六、解决方法
插入一条允许 3000 端口的规则，且必须放在REJECT规则之前。
sudo iptables -I INPUT 5 -p tcp --dport 3000 -j ACCEPT
查看：
sudo iptables -L INPUT --line-numbers
应类似：
1 RELATED,ESTABLISHED

2 icmp

3 lo

4 ssh

5 ACCEPT tcp dpt:3000

6 REJECT all
重点：
3000 必须位于 REJECT 前。
七、永久保存规则
安装：
sudo apt install iptables-persistent
保存：
sudo netfilter-persistent save
或者：
sudo iptables-save > /etc/iptables/rules.v4
八、整个排查流程
浏览器打不开3000
          │
          ▼
ss -tlnp
          │
          ▼
3000是否监听？
          │
      是 ▼
curl 127.0.0.1
          │
          ▼
程序是否正常？
          │
      是 ▼
检查 NSG
          │
          ▼
检查 Security List
          │
          ▼
检查 iptables
          │
          ▼
发现 INPUT 最后一条

REJECT ALL

          │
          ▼
插入

ACCEPT tcp 3000

          │
          ▼
恢复正常
九、本次排查收获
应用层先排除：确认 Node/Vite 正常监听，避免一开始怀疑代码。
云平台与主机防火墙分层检查：Oracle NSG、Security List 放行并不代表 Linux 一定允许。
抓包是定位网络问题的关键：tcpdump 能区分“包没到”和“包到了但被主机拒绝”。
iptables 优先级决定最终结果：一条末尾的 REJECT all 足以让所有已放行的云端规则失效。
推荐建立固定排查顺序：程序监听 → 本机访问 → 云安全组 → 抓包 → 主机防火墙，能够快速缩小问题范围。
