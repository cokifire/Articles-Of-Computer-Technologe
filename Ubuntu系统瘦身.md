在 Ubuntu磁盘空间紧张通常来自 日志、缓存、旧内核、Snap、Docker/容器残留 等。
一、先确认空间被谁占了（必做）
df -h
查看总体使用情况。
du -h --max-depth=1 / | sort -h
快速定位哪个目录占空间（重点关注 /var、/usr、/home）。
二、最立竿见影的瘦身操作（强烈推荐）
1️⃣ 清理 APT 缓存（几乎必做）
sudo apt clean
sudo apt autoclean
 在小盘 VPS 上，/var/cache/apt/archives 经常能占几百 MB。
2️⃣ 删除不再需要的软件包
sudo apt autoremove --purge -y
特别是旧依赖、旧内核。
3️⃣ 限制 / 清理 systemd 日志（非常关键）
查看日志占用：
journalctl --disk-usage
只保留最近 3 天日志：
sudo journalctl --vacuum-time=3d
或限制日志最大 50MB：
sudo journalctl --vacuum-size=50M
长期建议（防止再次爆盘）：
sudo nano /etc/systemd/journald.conf
修改或添加：
SystemMaxUse=50M
SystemMaxFileSize=10M
然后：
sudo systemctl restart systemd-journald
三、常见“吃空间大户”专项处理
4️⃣ /var/log 手动清理
sudo du -h /var/log | sort -h
可安全清空的文件（不是删目录）：
sudo truncate -s 0 /var/log/*.log
sudo truncate -s 0 /var/log/*/*.log
5 btmp吃硬盘大户
/var/log/btmp 是 系统安全登录失败日志，用于记录 所有失败的登录尝试，包括：
 SSH 登录失败（最常见）
 
 su / sudo 失败
 
 本地终端登录失败
 
在公网 VPS 上，这个文件 经常被 SSH 暴力破解打爆。
✅ 正确清空（不删除文件本身）
sudo truncate -s 0 /var/log/btmp

确认：
ls -lh /var/log/btmp

❌ 不推荐直接 rm（原因）
虽然也能用：
sudo rm /var/log/btmp

但某些系统服务期望该文件存在，可能会被重新创建并产生权限问题。
防止 btmp 再次爆炸（非常关键）
1️⃣ 通过 logrotate 限制大小（强烈建议）
编辑配置：
sudo nano /etc/logrotate.d/btmp

内容建议改为：
/var/log/btmp {
    monthly
    rotate 1
    size 10M
    missingok
    notifempty
    compress
}

手动测试：
sudo logrotate -f /etc/logrotate.d/btmp

2️⃣ 根本原因：SSH 暴力破解（强烈建议处理）
✅ 改 SSH 端口（立刻减少 90% 垃圾）
sudo nano /etc/ssh/sshd_config


Port 22222

然后：
sudo systemctl restart ssh

✅ 禁用 root 密码登录（强烈）
PermitRootLogin no
PasswordAuthentication no

仅使用 SSH Key 登录。
6️⃣ Docker / 容器用户（如果有）
查看占用：
docker system df
一键清理无用资源：
docker system prune -a -f
sudo apt remove linux-image-5.15.xxx

五、长期防爆盘配置（强烈建议）
9️⃣ 开启 logrotate（一般默认有）
检查：
ls /etc/logrotate.d/
如没有：
sudo apt install logrotate -y
🔟 临时文件自动清理
sudo systemctl enable systemd-tmpfiles-clean.timer
sudo systemctl start systemd-tmpfiles-clean.timer
