使用Ocserv 手动搭建 Cisco AnyConnect VPN服务端

很早就有人想要我写这个 Cisco AnyConnect VPN 服务端的搭建教程了，不过我一直嫌麻烦没写，这段时间没什么东西写，那么就写个看看吧。

**Cisco AnyConnect VPN客户端教程：[Cisco AnyConnect VPN Windows/Android 平台客户端使用教程](https://doubibackup.com/hc5a0yn5.html)**

**一键安装脚本：[『原创』Ocserv 搭建 Cisco AnyConnect VPN服务端 一键脚本](https://doubibackup.com/nr2hjmg2.html)**

## 介绍一下

**首先介绍一下Ocserv也就是OpenConnect，即Cisco AnyConnect的兼容服务端。**

众所周知，Cisco在网络设备领域处于霸主地位，其开发的AnyConnect作为一种VPN协议，不是想封就能封的。封锁AnyConnect将对大量跨国公司的子公司与母公司的通讯造成灾难性后果。因此，虽然AnyConnect的握手特征很明显，但是依然可以正常使用。

而AnyConnect作为Cisco专有技术，其服务端只能运行在Cisco设备上，即如果没有购买Cisco相关设备，将无法使用AnyConnect服务端。而OpenConnect的出现解决了这一个问题，OpenConnect是一个开源项目，其目标是在相对廉价的linux设备上运行与AnyConnect协议兼容的服务端，以此来使用该协议而不需要购买Cisco专有设备。

**AnyConnect目前支持 Windows 7+ / Android / IOS / Mac ，其他设备没有客户端所以无法使用，例如 XP系统。**

## 教程说明

**本教程仅在 Debian 7 中测试，理论上 Debian 8 与 Ubuntu 步骤一样。**

我也是刚刚了解这个VPN，AnyConnect的优势就是相比其他的VPN协议（OpenVPN、PPTP、L2TP、IKEv2等），AnyConnect并不会被墙封，当然也不代表完全没有干扰，所以**如果只是玩游戏的话，用其他的VPN协议容易被墙干掉，这时候就可以使用AnyConnect，如果只是看网页看视频，那还是建议使用ShadowsocksR。**

AnyConnect的配置参数很多，大部分我都不明白什么意思，教程里的示例配置文件只是在默认的配置文件中简单改了改，大家可以自己了解后去修改。

**本教程仅写出了通过用户名和密码的方式来登陆链接AnyConnect VPN，通过自签SSL证书用于客户端与服务端直接的SSL安全加密。**

> **注意：**如果服务器同时安装了 **锐速(ServerSpeed/LotServer)**，那么可能会导致 AnyConnect 连接上后**无网络或者速度异常(慢)**，这时候请关闭锐速，**BBR加速无影响**。

## 安装步骤

### 检查PPP/TUN环境

首先要检查VPS的TUN是否开启(OpenVZ虚拟化的服务器很可能默认关闭)。

1.  cat /dev/net/tun
2.  \# 返回的必须是：
3.  cat:  /dev/net/tun:  File descriptor in bad state

如果返回内容不是指定的结果，请与VPS提供商联系开启TUN权限（一般控制面板有开关）。

### 安装依赖

首先为了确保依赖安装正常、完整，我们需要更换系统 软件包源为最新的稳定源 `jessie` （本步骤必做，否则很容易出错）。

默认下面的代码是 美国的镜像源，可以更换下面代码 `us.sources.list 中的 us` ，具体可以[看这里](https://github.com/ToyoDAdoubiBackup/doubi/tree/master/sources)。

1.  rm -rf /etc/apt/sources.list && wget -N --no-check-certificate -O "/etc/apt/sources.list"  "https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/sources/us.sources.list"

然后我们更新软件包列表，并开始安装依赖：

1.  apt-get install pkg-config build-essential libgnutls28-dev libwrap0-dev liblz4-dev libseccomp-dev libreadline-dev libnl-nf-3-dev libev-dev gnutls-bin -y

> **注意：**这个更换 镜像源的步骤，**Debian 8、Debian 9 不需要执行**，可以直接跳过，**Debian 7 必须执行！**

### 编译安装

然后我们新建一个文件夹，并下载和编译安装 ocserv，此处用的 ocserv 可能不是最新的，最新的请看：[ocserv](ftp://ftp.infradead.org/pub/ocserv)

1.  mkdir ocserv && cd ocserv
2.  wget ftp://ftp.infradead.org/pub/ocserv/ocserv-0.11.8.tar.xz
3.  tar -xJf ocserv-0.11.8.tar.xz && cd ocserv-0.11.8
4.  ./configure

`./configure` 命令执行后，过一会就会出现如下信息，正常情况下应该跟我的差不多。

[点击展开 查看更多](#)

然后我们继续安装：

1.  make
2.  make install
3.  \# 安装后的文件可以不删除，卸载的时候还需要用到。

最后我们新建一个配置文件的文件夹：

1.  mkdir /etc/ocserv
2.  wget -N --no-check-certificate -P "/etc/ocserv" https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/other/ocserv.conf

上面下载的是我根据 ocserv 默认配置文件稍微改了改以适应教程的配置文件。

如果你想要自己改配置文件可以去看源码默认自带的配置文件： `ocserv-0.11.8/doc/sample.config`

### 配置参数解释

因为这些参数很多我也不明白，所以只挑我清楚的解释一下，有错误欢迎指出或补充。

[点击展开 查看更多](#)

## 添加 VPN用户

假设你要添加的账号用户名为 `doubi` ，那么命令如下：

1.  ocpasswd -c /etc/ocserv/ocpasswd doubi

输入后，服务器会提示你输入两次密码（不会显示，盲输），如下：

1.  \# === 服务器输出示例 === #
2.  root@debian:~/ocserv/ocserv-0.11.8\# ocpasswd -c /etc/ocserv/ocpasswd doubi
3.  Enter password:
4.  Re-enter password:
5.  \# === 服务器输出示例 === #

### 删除 VPN用户

注意：删除/锁定/解锁 VPN用户都没有任何提示！

1.  ocpasswd -c /etc/ocserv/ocpasswd -d doubi

### 锁定 VPN用户

1.  ocpasswd -c /etc/ocserv/ocpasswd -l doubi

### 解锁 VPN用户

1.  ocpasswd -c /etc/ocserv/ocpasswd -u doubi

## 自签SSL证书

首先在当前目录新建一个文件夹（要养成不把文件乱扔的习惯）。

1.  mkdir ssl && cd ssl

### 生成 CA证书

然后用下面的命令代码（8行一起复制一起粘贴一起执行），其中的两个 `doubi` 可以随意改为其他内容，不影响。

1.  echo -e 'cn = "doubi"
2.  organization = "doubi"
3.  serial = 1
4.  expiration_days = 365
5.  ca
6.  signing_key
7.  cert\_signing\_key
8.  crl\_signing\_key'  > ca.tmpl

然后我们生成证书和密匙：

1.  certtool --generate-privkey --outfile ca-key.pem
2.  certtool --generate-self-signed  --load-privkey ca-key.pem --template ca.tmpl --outfile ca-cert.pem

### 生成 服务器证书

继续用下面的命令代码（6行一起复制一起粘贴一起执行），其中的 `1.1.1.1` 请改为你的服务器IP，而 `doubi` 可以随意改为其他内容，不影响。

1.  echo -e 'cn = "1.1.1.1"
2.  organization = "doubi"
3.  expiration_days = 365
4.  signing_key
5.  encryption_key
6.  tls\_www\_server'  > server.tmpl

然后我们生成证书和密匙：

1.  certtool --generate-privkey --outfile server-key.pem
2.  certtool --generate-certificate --load-privkey server-key.pem --load-ca-certificate ca-cert.pem --load-ca-privkey ca-key.pem --template server.tmpl --outfile server-cert.pem

* * *

最后我们在ocserv的目录中新建一个ssl文件夹用于存放证书。

1.  mkdir /etc/ocserv/ssl

把刚才生成的证书和密匙都移过去：

1.  mv ca-cert.pem /etc/ocserv/ssl/ca-cert.pem
2.  mv ca-key.pem /etc/ocserv/ssl/ca-key.pem
3.  mv server-cert.pem /etc/ocserv/ssl/server-cert.pem
4.  mv server-key.pem /etc/ocserv/ssl/server-key.pem

现在刚才在当前文件夹新建的 ssl 文件夹就没用了，你可以删除它： `cd .. && rm -rf ssl/`

## 安装服务

下载我写好的服务脚本并赋予执行权限：

1.  wget -N --no-check-certificate -O "/etc/init.d/ocserv" https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/other/ocserv_debian
2.  chmod +x /etc/init.d/ocserv

### 设置开机启动(可选)

1.  update-rc.d -f ocserv defaults

## 防火墙配置

首先我们打开 **防火墙的NAT：**

我们需要先查看我们的主网卡是什么：

1.  ifconfig

输出结果可能如下，那么我们的主网卡默认是 **eth0** ，如果你是 OpenVZ，那么主网卡默认是 **venet0** 。

如果是 Debian9 系统，则默认网卡名为 **ens3**，CentOS Ubuntu 最新版本的系统可能为 **enpXsX(X代表数字或字母)**。

1.  \# === 服务器输出示例 === #
2.  root@debian:~\# ifconfig
3.  eth0 Link encap:Ethernet  HWaddr xx:xx:00:00:a8:a0 
4.   inet addr:xxx.xxx.xxx.xxx Bcast:xxx.xxx.225.255  Mask:255.255.255.0
5.   inet6 addr: xxx::xxx:ff:fe00:xxx/64  Scope:Link
6.    ......

8.  lo Link encap:Local  Loopback  
9.   inet addr:127.0.0.1  Mask:255.0.0.0
10.  inet6 addr:  ::1/128  Scope:Host
11.   ......
12. \# === 服务器输出示例 === #

1.  iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
2.  \# 如果是 OpenVZ就执行下面这个：
3.  iptables -t nat -A POSTROUTING -o venet0 -j MASQUERADE

然后我们需要打开 **ipv4防火墙转发：**

1.  echo -e "net.ipv4.ip_forward=1"  >>  /etc/sysctl.conf && sysctl -p

最后，我们就需要**开放防火墙端口**了，教程中示例文件的**默认TCP/UDP端口都是 443**，如果你改了 那么改为其他端口即可。

1.  iptables -I INPUT -p tcp --dport 443  -j ACCEPT
2.  iptables -I INPUT -p udp --dport 443  -j ACCEPT
3.  \# 如果你以后要删除规则，那么把 -I 改成 -D 即可。
4.  iptables -D INPUT -p tcp --dport 443  -j ACCEPT
5.  iptables -D INPUT -p udp --dport 443  -j ACCEPT

### 配置防火墙开启启动读取规则

一般默认 iptbales 关机后并不会保存规则，这样开机后 防火墙规则也全都清空了，所以需要设置一下。

**Debian/Ubuntu 系统：**

1.  iptables-save >  /etc/iptables.up.rules
2.  echo -e '#!/bin/bash\\n/sbin/iptables-restore < /etc/iptables.up.rules'  >  /etc/network/if-pre-up.d/iptables
3.  chmod +x /etc/network/if-pre-up.d/iptables

以后需要保存防火墙规则只需要执行：

1.  iptables-save >  /etc/iptables.up.rules

## 最后测试

最后我们需要运行下面这个命令来测试 ocserv 是否没有问题了。

1.  ocserv -f -d 1
2.  \# 如果没有问题，那么我们就按 Ctrl + C 键退出。

VPS输出 正常情况如下：

[点击展开 查看更多](#)

## 使用说明

1.  /etc/init.d/ocserv start
2.  \# 启动 ocserv 
3.  /etc/init.d/ocserv stop
4.  \# 停止 ocserv 
5.  /etc/init.d/ocserv restart
6.  \# 重启 ocserv 
7.  /etc/init.d/ocserv status
8.  \# 查看 ocserv 运行状态 
9.  /etc/init.d/ocserv log
10. \# 查看 ocserv 运行日志
11. /etc/init.d/ocserv test
12. \# 测试 ocserv 配置文件是否正确

**配置文件：**/etc/ocserv/ocserv.conf

## 卸载方法

假设你一开始源码编译安装的目录是：/root/ocserv/ocserv-0.11.8 ，那么这么做：

1.  cd /root/ocserv/ocserv-0.11.8
2.  \# 进入源码编辑安装目录

4.  make uninstall
5.  \# 执行卸载命令

7.  cd ..  && cd ..
8.  rm -rf ocserv/
9.  \# 回到 /root 文件夹，删除 ocserv 源码自身

11. rm -rf /etc/ocserv/
12. \# 删除配置文件目录

如果你还下载了系统服务脚本并设置开机启动了，那么需要：

1.  rm -rf /etc/init.d/ocserv
2.  update-rc.d -f ocserv remove

## 其他说明

> **注意：**如果服务器同时安装了 **锐速(ServerSpeed/LotServer)**，那么可能会导致 AnyConnect 连接上后**无网络或者速度异常(慢)**，这时候请关闭锐速，**BBR加速无影响**。

### 运行优化说明

建议在运行 ocserv前，执行一下这个命令，作用是提高系统的文件符同时打开数量，对于TCP连接过多的时候系统默认的 1024 就会成为速度瓶颈。

1.  ulimit -n 51200

这个命令只有临时有效，重启后失效，如果想要永久有效，请执行：

1.  echo "\* soft nofile 51200
2.  \* hard nofile 51200"  >>  /etc/security/limits.conf

然后最后再执行一下`ulimit -n 51200` 即可。

### 提示 wget: command not found 的错误

这是你的系统精简的太干净了，wget都没有安装，所以需要安装wget。

1.  \# CentOS系统:
2.  yum install -y wget

4.  \# Debian/Ubuntu系统:
5.  apt-get install -y wget

* * *

**参考资料：**[https://blog.uuz.moe/2016/04/02/cisco-anyconnect-deployment/](https://blog.uuz.moe/2016/04/02/cisco-anyconnect-deployment/)  
[https://mark1998.com/anyconnect-on-debian-and-ubuntu/](https://mark1998.com/anyconnect-on-debian-and-ubuntu/)

**转载请超链接注明：**[逗比根据地](https://doubibackup.com/index.html) » [使用Ocserv 手动搭建 Cisco AnyConnect VPN服务端](https://doubibackup.com/k9994-k3.html)

**责任声明：**本站一切资源仅用作交流学习，请勿用作商业或违法行为！如造成任何后果，本站概不负责！