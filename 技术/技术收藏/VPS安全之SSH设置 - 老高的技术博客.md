VPS安全之SSH设置 - 老高的技术博客

SSH是管理VPS的重要途径，所以SSH经常会受到攻击，所以我们需要将SSH**武装**起来，保护我们VPS的安全。

SSH服务的配置文件位于`/etc/ssh/sshd_config`，我们的安全设置都是围绕此文件展开，所以修改前最好先备份一次，以免出现无法登陆的情况。

修改完不要忘了执行

```
service sshd restart
```

## 如何打开

```

vim /etc/ssh/sshd_config
```

## 修改端口

## 禁止ROOT用户登陆

```

PermitRootLogin no
```

## 限制其他用户登陆

```

AllowUsers fsmythe bnice swilson
DenyUsers jhacker joebadguy jripper
```

## 使用 Chroot SSHD 将 SFTP 用户局限于其自己的主目录：

```

ChrootDirectory /home/%u
X11Forwarding no
AllowTcpForwarding no
```

## 登陆IP限制

```

ALL: 192.168.200.09        
```

## 禁止空密码登陆

```

PermitEmptyPasswords no
```

禁用用户的 .rhosts 文件：

> .rhosts 文件通常许可 UNIX 系统的网络访问权限。.rhosts 文件列出可以访问远程计算机的计算机名及关联的登录名。在正确配置了 .rhosts 文件的远程计算机上运行 rcp、rexec 或 rsh 命令时，您不必提供远程计算机的登录和

```

IgnoreRhosts yes
```

## 禁用基于主机的身份验证

```

HostbasedAuthentication no
```

## 删除不安全文件

从系统上删除 rlogin 和 rsh 二进制程序，并将它们替代为 SSH 的一个 symlink：

```

/usr/bin/rsh


```

## 设置自动掉线

```

ClientAliveInterval 600        
ClientAliveCountMax 0
```

## 指纹登陆

生成4096位密钥

```
ssh-keygen -t rsa -b 4096
```

将公钥拷贝至服务器对应用户的.ssh下，重命名为authorized_keys

```
scp -P xxxxx ~/.ssh/id_rsa.pub server:/root/.ssh/authorized_keys
```

如果已经存在authorized\_keys，需要将公钥追加至authorized\_keys

```
scp -P xxxxx ~/.ssh/id_rsa.pub server:/root/.ssh/tmp.pub



cat /root/.ssh/tmp.pub >> /root/.ssh/authorized_keys

```

## 禁止使用密码登陆

```

PasswordAuthentication no
```

## 报错

WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

删除`~/.ssh/known_hosts`文件

```
rm ~/.ssh/known_hosts
```

## 检查登录日志

如果你的服务器一直很正常，那也可能不正常的表现，最好的办法就是定期查询ssh的登录日志，手动发现系统的异常！

```


LogLevel DEBUG


tail -100 /var/log/secure


who /var/log/wtmp

last
```

## fail2ban

这个小巧的软件可以代替你做很多事情，以暴力破解ssh密码为例，当我们安装fail2ban后，经过合理的配置，我们可以自动屏蔽某个攻击IP的所有请求。

项目地址：[https://github.com/fail2ban/fail2ban](https://github.com/fail2ban/fail2ban)

需求:

Python2 >= 2.6 or Python >= 3.2 or PyPy

```



git clone https://github.com/fail2ban/fail2ban.git
cd fail2ban

python setup.py install

cp files/redhat-initd /etc/init.d/fail2ban

chmod 755 /etc/init.d/fail2ban

chkconfig --add fail2ban
chkconfig --level 35 fail2ban on


vim /etc/fail2ban/jail.conf



[ssh-iptables]
enabled  = true
filter   = sshd
action   = iptables[name=SSH, port=ssh, protocol=tcp]
           sendmail-whois[name=SSH, dest=root, sender=fail2ban@example.com, sendername="Fail2Ban"]
logpath  = /var/log/secure
maxretry = 2
[ssh-ddos]
enabled = true
filter  = sshd-ddos
action  = iptables[name=ssh-ddos, port=ssh,sftp protocol=tcp,udp]
logpath  = /var/log/messages
maxretry = 2


mkdir -p /var/run/fail2ban



service fail2ban restart


service fail2ban status






```

## 禁止IP访问

```
vim /etc/hosts.deny


all:1.1.1.1


service network restart
```

Reference:

[http://www.ibm.com/developerworks/cn/aix/library/au-sshsecurity/](https://www.ibm.com/developerworks/cn/aix/library/au-sshsecurity/)  
[http://zhidao.baidu.com/link?url=lk0vycK63DsC8x3mO2FkSoPXMIXCPmm9vYkrt35pZ_Ans5KsyRXPRs6SPMC63FVxYj30uGOvueFnmSt5eMwMQK](https://zhidao.baidu.com/link?url=lk0vycK63DsC8x3mO2FkSoPXMIXCPmm9vYkrt35pZ_Ans5KsyRXPRs6SPMC63FVxYj30uGOvueFnmSt5eMwMQK)