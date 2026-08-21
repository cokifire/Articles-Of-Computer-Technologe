很多人希望把旧 Android 手机变成一个长期运行的 AI 自动化节点。理论上，通过 Linux 环境可以在手机上运行 OpenClaw，但实际部署过程中会遇到不少坑。本文记录了一次完整的安装过程以及解决方法。
很多人第一次看到 OpenClaw，会以为它只是一个普通 Node 项目，但真正到了安卓手机环境，事情完全不是这样。
因为 Android 并不是标准 Linux，它只是一个带 Linux 内核的移动系统。
 所以在手机上安装 OpenClaw，最大的挑战不是命令本身，而是 环境兼容层。
很多人第一步都会直接在 Termux 里执行：
npm install -g openclaw

结果通常会遇到：
 native module 编译失败
 
 glibc 缺失
 
 node-gyp 报错
 
 ABI 不兼容
 
原因是：
Termux 用的是：Android bionic libc
而 OpenClaw 的很多依赖默认按：glibc Linux 编译。
所以结论很明确：不要直接在 Termux 原生层部署 OpenClaw。
比较稳定的架构是：
Termux
→ proot-distro
→ Ubuntu
→ Node.js
→ OpenClaw
也就是说，需要在 Android 上先创建一个 Linux 用户空间，再在里面部署 OpenClaw。
1， 先安装 Ubuntu：
pkg install proot-distro
proot-distro install ubuntu
proot-distro login ubuntu

这一步看似简单，但很多人第一次会忽略：Termux 镜像源慢。如果不换镜像，在国内环境下载 Ubuntu 会非常慢。
termux-change-repo
这是 Temux 官方提供的交互式工具，运行后会显示可用的镜像列表，你可以用方向键选择并切换到国内的地区。
2，安装 Node.js
OpenClaw 依赖 Node.js，因此需要先安装。
更新系统：
apt update
apt upgrade

安装基础编译工具：
apt install build-essential curl git python3

然后安装 Node.js：
curl -fsSL https://deb.nodesource.com/setup_22.x | bash
apt install nodejs

确认版本：
node -v
npm -v
3， 安装 OpenClaw
建议安装社区中文版：
# 社区中文适配版（对国内网络更友好）
npm install -g openclaw-cn@latest

如果网络慢，可以换 npm 镜像：
npm config set registry https://registry.npmmirror.com

然后启动：
openclaw-cn onboard

至此 OpenClaw 基本可以运行。
实际安装过程中遇到的坑
下面是整个安装过程中最常见的几个问题。
1：Termux 版本问题
如果从 Google Play 安装 Termux，经常会出现：
 软件源无法更新
 
 包依赖错误
 
正确来源是：
 F-Droid 官网
 
 官方 GitHub release
2：OpenClaw 不能直接安装插件
例如安装飞书插件时执行：
openclaw plugins install @openclaw/lark-connector

会报错：
npm error 404 Not Found

原因是：
很多 OpenClaw 插件 并没有发布到 npm 仓库，而是直接放在 GitHub 仓库中。
因此需要手动安装插件，而不是使用 plugins install。
3：插件已经存在但默认 disabled
例如插件列表里可以看到：
feishu   disabled

插件一直启动不成功，只是还没有编译，安装typescript编译
npm install -g typescript
npm run build
4：Android 杀后台
Android 系统会自动关闭后台进程，导致 OpenClaw 停止运行。
需要在系统设置里：
关闭 Termux 的
 电池优化
 
 后台限制
 
否则几分钟后进程就会被系统杀掉。
5：浏览器类 skill 在手机上容易失败
很多人一上来把所有 skill 全开：
browser: true
python: true
file: true
search: true
shell: true

手机环境里最容易出问题的是：
browser

因为它依赖浏览器运行时：
 chromium
 
 playwright
 
安卓 Ubuntu 下极不稳定。
最稳建议：
先只开：
file
python
shell
search
