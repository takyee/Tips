本指引是实现OpenVPN SSL site-to-site VPN，客户端站点使用 "client-config-dir ccd"实现个站点间身后不同网络的互通。

https://www.digitalocean.com/community/tutorials/how-to-set-up-and-configure-an-openvpn-server-on-ubuntu-20-04
https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/#scripting-and-environmental-variables

### Step 1 — Installing OpenVPN and Easy-RSA
```
sudo apt update
sudo apt install openvpn easy-rsa
```
建议下载下载最新版本easy-rsa https://github.com/OpenVPN/easy-rsa/releases

### Step 2 — Creating a PKI for OpenVPN
接下来是为openvpn 创建ssl证书
```
cd ~/easy-rsa
nano vars
```
添加以下到文件末尾
```
set_var EASYRSA_ALGO "ec"
set_var EASYRSA_DIGEST "sha512"
```
初始化 PKI
```
./easyrsa init-pki
````
接下来是床架CA证书和OPENVPN的私钥和密钥
可参考 easy-rsa --help

### Step 3 — Configuring OpenVPN Cryptographic Material
```
cd ~/easy-rsa
openvpn --genkey --secret ta.key
```
### Step 4 — 生成客户端证书

```
easy-rsa build-client-full 证书名字 nopass   #注意证书名字必须唯一"client-config ccd" 中的配置文件必须与证书名字一致
```
### Step 5 - OpenVPN服务器配置

server.conf
```
port 1194

# TCP or UDP server?
;proto tcp
proto udp

mode server
fast-io
sndbuf 512000
rcvbuf 512000
push "sndbuf 512000"
push "rcvbuf 512000"
txqueuelen 2000

# "dev tun" will create a routed IP tunnel,
# "dev tap" will create an ethernet tunnel.
# Use "dev tap0" if you are ethernet bridging
# and have precreated a tap0 virtual interface
# and bridged it with your ethernet interface.
# If you want to control access policies
# over the VPN, you must create firewall
# rules for the the TUN/TAP interface.
# On non-Windows systems, you can give
# an explicit unit number, such as tun0.
# On Windows, use "dev-node" for this.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

ca ./ssl-new/ca.crt   # easy-rsa 生成的ca
cert ./ssl-new/openvpn-gateway.crt   # easy-rsa 生成的openVPN 服务器证书
key ./ssl-new/openvpn-gateway.key  # This file should be kept secret easy-rsa 生成的openVPN 服务器证书

# Diffie hellman parameters.
# Generate your own with:
#   openssl dhparam -out dh2048.pem 2048
#dh dh2048.pem

dh none    # 使用了aes-256-gcm加密算法，测试可以none

# Network topology
# Should be subnet (addressing via IP)
# unless Windows clients v2.0.9 and lower have to
# be supported (then net30, i.e. a /30 per client)
# Defaults to net30 (not recommended)
topology subnet    # 默认是net30 OpenVPN不推荐使用

# Configure server mode and supply a VPN subnet
# for OpenVPN to draw client addresses from.
# The server will take 10.8.0.1 for itself,
# the rest will be made available to clients.
# Each client will be able to reach the server
# on 10.8.0.1. Comment this line out if you are
# ethernet bridging. See the man page for more info.
server 192.168.214.0 255.255.255.0    # OpenVPN 服务器端IP地址 

# Maintain a record of client <-> virtual IP address
# associations in this file.  If OpenVPN goes down or
# is restarted, reconnecting clients can be assigned
# the same virtual IP address from the pool that was
# previously assigned.
ifconfig-pool-persist ipp.txt    # ipp.txt文件用于记录客户端的IP地址，当下次重新连接时保持IP地址不变

# Configure server mode for ethernet bridging.
# You must first use your OS's bridging capability
# to bridge the TAP interface with the ethernet
# NIC interface.  Then you must manually set the
# IP/netmask on the bridge interface, here we
# assume 10.8.0.4/255.255.255.0.  Finally we
# must set aside an IP range in this subnet
# (start=10.8.0.50 end=10.8.0.100) to allocate
# to connecting clients.  Leave this line commented
# out unless you are ethernet bridging.
;server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100

# Configure server mode for ethernet bridging
# using a DHCP-proxy, where clients talk
# to the OpenVPN server-side DHCP server
# to receive their IP address allocation
# and DNS server addresses.  You must first use
# your OS's bridging capability to bridge the TAP
# interface with the ethernet NIC interface.
# Note: this mode only works on clients (such as
# Windows), where the client-side TAP adapter is
# bound to a DHCP client.
;server-bridge

# Push routes to the client to allow it
# to reach other private subnets behind
# the server.  Remember that these
# private subnets will also need
# to know to route the OpenVPN client
# address pool (10.8.0.0/255.255.255.0)
# back to the OpenVPN server.
;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"

# push route 的作用，是将OpenVPN 服务器端后面的子网路由通告给每一个openVPN客户端
push "route 172.18.44.0 255.255.255.0"
push "route 172.18.128.0 255.255.240.0"
push "route 172.18.224.0 255.255.240.0"
push "route 192.168.214.0 255.255.255.0"


# To assign specific IP addresses to specific
# clients or if a connecting client has a private
# subnet behind it that should also have VPN access,
# use the subdirectory "ccd" for client-specific
# configuration files (see man page for more info).

# EXAMPLE: Suppose the client
# having the certificate common name "Thelonious"
# also has a small subnet behind his connecting
# machine, such as 192.168.40.128/255.255.255.248.
# First, uncomment out these lines:

# client-config-dir 作用是将openVPN客户端身后路由，宣告给openVPN服务器，此处的条目是ccd目录下各个配置文件中iroute条目的汇总。
client-config-dir ccd
route 10.10.5.0 255.255.255.0
route 10.10.10.0 255.255.255.0
route 10.10.11.0 255.255.255.0
route 10.10.13.0 255.255.255.0
route 10.10.14.0 255.255.255.0
route 10.10.100.0 255.255.255.0
route 10.51.2.0 255.255.255.0
route 10.51.5.0 255.255.255.0
route 10.51.11.0 255.255.255.0
route 10.51.51.0 255.255.255.0
route 10.51.100.0 255.255.255.0
route 172.19.33.0 255.255.255.0
route 192.168.128.0 255.255.255.224
route 172.17.0.0 255.255.255.0
#route 10.254.254.0 255.255.255.240
#route 10.10.0.0 255.255.0.0
#SHH KD2 ON Aliyun(Shanghai)
#route 192.168.33.0 255.255.255.0


# Then create a file ccd/Thelonious with this line:
#   iroute 192.168.40.128 255.255.255.248
# This will allow Thelonious' private subnet to
# access the VPN.  This example will only work
# if you are routing, not bridging, i.e. you are
# using "dev tun" and "server" directives.

# EXAMPLE: Suppose you want to give
# Thelonious a fixed VPN IP address of 10.9.0.1.
# First uncomment out these lines:
;client-config-dir ccd
;route 10.9.0.0 255.255.255.252
# Then add this line to ccd/Thelonious:
#   ifconfig-push 10.9.0.1 10.9.0.2

# Suppose that you want to enable different
# firewall access policies for different groups
# of clients.  There are two methods:
# (1) Run multiple OpenVPN daemons, one for each
#     group, and firewall the TUN/TAP interface
#     for each group/daemon appropriately.
# (2) (Advanced) Create a script to dynamically
#     modify the firewall in response to access
#     from different clients.  See man
#     page for more info on learn-address script.
;learn-address ./script

# If enabled, this directive will configure
# all clients to redirect their default
# network gateway through the VPN, causing
# all IP traffic such as web browsing and
# and DNS lookups to go through the VPN
# (The OpenVPN server machine may need to NAT
# or bridge the TUN/TAP interface to the internet
# in order for this to work properly).
;push "redirect-gateway def1 bypass-dhcp"

# Certain Windows-specific network settings
# can be pushed to clients, such as DNS
# or WINS server addresses.  CAVEAT:
# http://openvpn.net/faq.html#dhcpcaveats
# The addresses below refer to the public
# DNS servers provided by opendns.com.
;push "dhcp-option DNS 208.67.222.222"
;push "dhcp-option DNS 208.67.220.220"

# Uncomment this directive to allow different
# clients to be able to "see" each other.
# By default, clients will only see the server.
# To force clients to only see the server, you
# will also need to appropriately firewall the
# server's TUN/TAP interface.
client-to-client    # 允许openVPN客户端之间互相通信

# Uncomment this directive if multiple clients
# might connect with the same certificate/key
# files or common names.  This is recommended
# only for testing purposes.  For production use,
# each client should have its own certificate/key
# pair.
#
# IF YOU HAVE NOT GENERATED INDIVIDUAL
# CERTIFICATE/KEY PAIRS FOR EACH CLIENT,
# EACH HAVING ITS OWN UNIQUE "COMMON NAME",
# UNCOMMENT THIS LINE OUT.
;duplicate-cn

# The keepalive directive causes ping-like
# messages to be sent back and forth over
# the link so that each side knows when
# the other side has gone down.
# Ping every 10 seconds, assume that remote
# peer is down if no ping received during
# a 120 second time period.
keepalive 10 120

# For extra security beyond that provided
# by SSL/TLS, create an "HMAC firewall"
# to help block DoS attacks and UDP port flooding.
#
# Generate with:
#   openvpn --genkey --secret ta.key
#
# The server and each client must have
# a copy of this key.
# The second parameter should be '0'
# on the server and '1' on the clients.

;tls-auth .ssl/ta.key 0 # This file is secret
tls-crypt ./ssl-new/ta.key    # 注释tls-auth，启用tls-crypt


# Select a cryptographic cipher.
# This config item must be copied to
# the client config file as well.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
;cipher AES-256-CBC
cipher AES-256-GCM    # AES-256-GCM 有更好的性能
#auth    # 默认使用SHA1，如果需要更高级别的验证算法可以需要注释，例如 auth SHA256

# Enable compression on the VPN link and push the
# option to the client (v2.4+ only, for earlier
# versions see below)
compress lz4-v2    # 启用压缩算法
push "compress lz4-v2"    #推送压缩算法给openVPN客户端

# For compression compatible with older clients use comp-lzo
# If you enable it here, you must also
# enable it in the client config file.
;comp-lzo

# The maximum number of concurrently connected
# clients we want to allow.
;max-clients 100

# It's a good idea to reduce the OpenVPN
# daemon's privileges after initialization.
#
# You can uncomment this out on
# non-Windows systems.
;user nobody
;group nobody

# The persist options will try to avoid
# accessing certain resources on restart
# that may no longer be accessible because
# of the privilege downgrade.
persist-key
persist-tun

# Output a short status file showing
# current connections, truncated
# and rewritten every minute.
status openvpn-status.log

# By default, log messages will go to the syslog (or
# on Windows, if running as a service, they will go to
# the "\Program Files\OpenVPN\log" directory).
# Use log or log-append to override this default.
# "log" will truncate the log file on OpenVPN startup,
# while "log-append" will append to it.  Use one
# or the other (but not both).
log         openvpn.log
log-append  openvpn.log

# Set the appropriate level of log
# file verbosity.
#
# 0 is silent, except for fatal errors
# 4 is reasonable for general usage
# 5 and 6 can help to debug connection problems
# 9 is extremely verbose
verb 0

# Silence repeating messages.  At most 20
# sequential messages of the same message
# category will be output to the log.
;mute 20

# Notify the client that when the server restarts so it
# can automatically reconnect.
explicit-exit-notify 1
```

### Step 6 - client-config-dir ccd 配置

```
$ sudo mkdir /etc/openvpn/server/ccd
$ cd /etc/openvpn/server/ccd
$ sudo editor "openvpn 客户端 sslz证书名字"
例如: iroute 10.10.0.0 255.255.0.0
```

### sTep 7 OpenVPN客户端配置
```

# Specify that we are a client and that we
# will be pulling certain config file directives
# from the server.
client

# Use the same setting as you are using on
# the server.
# On most systems, the VPN will not function
# unless you partially or fully disable
# the firewall for the TUN/TAP interface.
;dev tap
dev tun

# Windows needs the TAP-Win32 adapter name
# from the Network Connections panel
# if you have more than one.  On XP SP2,
# you may need to disable the firewall
# for the TAP adapter.
;dev-node MyTap

;proto tcp
proto udp


remote 39.108.19.163  1194    # openVPN 服务器地址,支持FQDN
;remote my-server-2 1194

# Choose a random host from the remote
# list for load-balancing.  Otherwise
# try hosts in the order specified.
;remote-random

# Keep trying indefinitely to resolve the
# host name of the OpenVPN server.  Very useful
# on machines which are not permanently connected
# to the internet such as laptops.
resolv-retry infinite

# Most clients don't need to bind to
# a specific local port number.
nobind

# Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup

# Try to preserve some state across restarts.
persist-key
persist-tun

# If you are connecting through an
# HTTP proxy to reach the actual OpenVPN
# server, put the proxy server/IP and
# port number here.  See the man page
# if your proxy server requires
# authentication.
;http-proxy-retry # retry on connection failures
;http-proxy [proxy server] [proxy port #]

# Wireless networks often produce a lot
# of duplicate packets.  Set this flag
# to silence duplicate packet warnings.
;mute-replay-warnings

# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.

# OpenVPN服务器端 easy-rsa 生成的客户端证书
ca /etc/openvpn/client/ssl/ca.crt
cert /etc/openvpn/client/ssl/2601.crt
key /etc/openvpn/client/ssl/2601.key

# Verify server certificate by checking that the
# certicate has the correct key usage set.
# This is an important precaution to protect against
# a potential attack discussed here:
#  http://openvpn.net/howto.html#mitm
#
# To use this feature, you will need to generate
# your server certificates with the keyUsage set to
#   digitalSignature, keyEncipherment
# and the extendedKeyUsage to
#   serverAuth
# EasyRSA can do this for you.
remote-cert-tls server

# If a tls-auth key is used on the server
# then every client must also have the key.
;tls-auth ta.key 1

tls-crypt /etc/openvpn/client/ssl/ta.key    # openVPN服务器端 easy-rsa生成的ta密钥

# Select a cryptographic cipher.
# If the cipher option is used on the server
# then you must also specify it here.
# Note that v2.4 client/server will automatically
# negotiate AES-256-GCM in TLS mode.
# See also the ncp-cipher option in the manpage
;cipher AES-256-CBC
cipher AES-256-GCM    # AES-256-GCM有更好的性能
#auth    # auth默认为sha1
key-direction 1

# Enable compression on the VPN link.
# Don't enable this unless it is also
# enabled in the server config file.
#comp-lzo


# Set log file verbosity.
verb 3

# Silence repeating messages
;mute 20

# 当OpenVPN连接成功后，也就是tun 网卡设备启用后执行以下脚本，执行的脚本可以传递 openVPN的环境变量
script-security 2
up /etc/openvpn/client/addRoute

$ sudo editor /etc/openvpn/client/addRoute
ip route add 192.168.1.0/24 via $route_vpn_gateway dev $dev    # $route_vpn_gatway $dev 为OpenVPN环境变量，更多变量可参考上面的连
$chmod +x addRoute    # 添加可执行权限


```
