通过iptables实现IP端口数据包转发服务配置

VPS会经常使用到端口转发，以解决不同网络之间互联互通问题。例如我们的laxA-QN机房对联通更好，laxB-C3机房对电信更好。
这时如果有一台VPS用作中转，这样就能达到稳定且快速的效果。

这里，我们介绍一个使用iptables来进行中转的教程。使用iptables的好处就是不用额外装东西，且同时支持tcp及udp。


关于CentOS 7系统：需要删除firewalld装回iptables.
```bash
# 安装命令(已默认安装)：
systemctl stop firewalld.service
systemctl disable firewalld.service
yum install iptables-services -y
systemctl enable iptables.service
```
 

第一步：开启系统的转发功能
首先，先确认服务器是否已开启转发，运行：
```bash
sysctl net.ipv4.ip_forward
# 如果已经启动则显示
> net.ipv4.ip_forward = 1
# 如果没有启动则显示(请按照下面步骤进行开启)
> net.ipv4.ip_forward = 0


# CentOS 6/Debian/Ubuntu 开启方式：
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# CentOS 7 开启方式：
echo "net.ipv4.ip_forward = 1" >> /usr/lib/sysctl.d/cloudiplc.conf
sysctl -p /usr/lib/sysctl.d/cloudiplc.conf

# 第二步： 加入iptables规则
iptables -t nat -A PREROUTING -p tcp --dport [端口号] -j DNAT --to-destination [目标IP]
iptables -t nat -A PREROUTING -p udp --dport [端口号] -j DNAT --to-destination [目标IP]
iptables -t nat -A POSTROUTING -p tcp -d [目标IP] --dport [端口号] -j SNAT --to-source [本地服务器IP]
iptables -t nat -A POSTROUTING -p udp -d [目标IP] --dport [端口号] -j SNAT --to-source [本地服务器IP]
* 如服务器是内网IP(例如NAT-VPS),请用ifconfig或ip addr确认走公网流量网卡的内网IP(本地服务器IP).
 

# 第三步：重启iptables使规则生效
# CentOS使用：
service iptables save
service iptables restart

# Debian/Ubuntu使用：
iptables-save > /etc/iptables.up.rules
iptables-restore < /etc/iptables.up.rules

 

# 扩展需求：
# CentOS修改：/etc/sysconfig/iptables
# Debian/Ubuntu修改：/etc/iptables.up.rules

# 同端口号配置方案：（使用本地服务器的10000端口来转发目标IP为1.1.1.1的10000端口）
-A PREROUTING -p tcp -m tcp --dport 10000 -j DNAT --to-destination 1.1.1.1
-A PREROUTING -p udp -m udp --dport 10000 -j DNAT --to-destination 1.1.1.1
-A POSTROUTING -d 1.1.1.1 -p tcp -m tcp --dport 10000 -j SNAT --to-source [本地服务器IP]
-A POSTROUTING -d 1.1.1.1 -p udp -m udp --dport 10000 -j SNAT --to-source [本地服务器IP]


# 非同端口号配置方案：（使用本地服务器的60000端口来转发目标IP为1.1.1.1的50000端口）
-A PREROUTING -p tcp -m tcp --dport 60000 -j DNAT --to-destination 1.1.1.1:50000
-A PREROUTING -p udp -m udp --dport 60000 -j DNAT --to-destination 1.1.1.1:50000
-A POSTROUTING -d 1.1.1.1 -p tcp -m tcp --dport 50000 -j SNAT --to-source [本地服务器IP]
-A POSTROUTING -d 1.1.1.1 -p udp -m udp --dport 50000 -j SNAT --to-source [本地服务器IP]


# 多端口转发配置方案：(将本地服务器的10000~65535转发至目标IP为1.1.1.1的10000~65535端口)
-A PREROUTING -p tcp -m tcp --dport 10000:65535 -j DNAT --to-destination 1.1.1.1
-A PREROUTING -p udp -m udp --dport 10000:65535 -j DNAT --to-destination 1.1.1.1
-A POSTROUTING -d 1.1.1.1 -p tcp -m tcp --dport 10000:65535 -j SNAT --to-source [本地服务器IP]
-A POSTROUTING -d 1.1.1.1 -p udp -m udp --dport 10000:65535 -j SNAT --to-source [本地服务器IP]


# 修改后重启生效：
# CentOS6：
service iptables restart

# CentOS7:
systemctl restart iptables.service


# Debian/Ubuntu：
iptables-restore < /etc/iptables.up.rules



# 查看目前正在NAT的规则：
iptables -t nat -nL

# 查看目前iptables的规则：
iptables -nL --line-number

 