
ocserv可以很好的做为Cisco ASA WEBVPN的开源替代解决方案，近期在实际的生产环境部署情况中表现非常优秀，以下是一些优化参数：

## 安装

### Ubuntu初始安装&配置

#### 开启路由转发

```shell
sudo vi /etc/sysctl.conf
net.ipv4.ip_forward=1
```
### 开启BBR

```shell
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
sysctl -p
```

### 安装ocsrv

当前最新版本1.16 `https://ocserv.gitlab.io/www/download.html`

```shell
# Required
apt-get install -y libgnutls28-dev libev-dev
# Optional functionality and testing
apt get install -y libpam0g-dev liblz4-dev libseccomp-dev \
	libreadline-dev libnl-route-3-dev libkrb5-dev libradcli-dev \
	libcurl4-gnutls-dev libcjose-dev libjansson-dev libprotobuf-c-dev \
	libtalloc-dev libhttp-parser-dev protobuf-c-compiler gperf \
	nuttcp lcov libuid-wrapper libpam-wrapper libnss-wrapper \
	libsocket-wrapper gss-ntlmssp haproxy iputils-ping freeradius \
	gawk gnutls-bin iproute2 yajl-tools tcpdump

```

可参照 README.md 编译安装

### 配置

 **网上大部分的教程都是，按照在VPS环境部署FQ使用的场景，不适合在实际企业网环境里部署**

需要在ISR路由器中映射ocserv的服务端口，默认是tcp 443,udp 443

在下面的拓扑中，ocserv Server无需启用 **iptables NAT**，核心三层交换机可以写一条往ocserv 客户端的静态路由

例如 **ip route 10.10.14.0 255.255.255.0 xx.xx.xx.xx 注:X为ocserv IP地址**

这样内网用户可以可以直接访问 AnyConnect Client用户，而且AnyConnect用户也是以VPN地址访问资源。

 ![ocserv拓扑图](./images/ocserv/2022-06-18%2014_05_11-Cisco%20Packet%20Tracer.png "ocserv拓扑图")


**在实际部署中发现启用DTLS的时候，AnyConnect的传输速度始终无法提高，在中国移动宽带下，最大传输速度大约在3Mbps，可以通过组配置文件对传输带宽高的用户禁用DTLS，通过TLS端口传输数据，配合Linux BBR可以跑慢ocserv 服务器带宽。**

ocpasswd添加新用户并加入特定组

```shell
ocpasswd -c /etc/ocserv/ocserv.passwd abc -g administrator
```

 ### ocserv集成 Active Directroy

 1.官方解决方案

 [How to setup ocserv for Microsoft AD Authentication](https://ocserv.gitlab.io/www/recipes-ocserv-ad-authentication.html)

 2. 可以借助Windows Radius服务器来解决
 
