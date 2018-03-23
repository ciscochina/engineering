---
layout: post
title: "租用 VPS 搭建 Shadowsocks"
date:   2018-03-23 
author: Meng Yu
categories: python
---

![enter image description here](http://owu6vks0s.bkt.clouddn.com/develo_2.png){:style="margin:auto;border:none;width:400px"}

由于个人的 PC 貌似加不到 Cisco 的域了，所以自己的电脑无法享受到 Anyconnect 的优质服务，加上平时又不喜欢走哪都拖着公司那个沉沉 T450 。那就索性自己搭一个吧，以下是对搭建过程的记录，有需要的同学可以参阅。

### 1. 搭建一个 VPS

VPS 的提供商有很多，例如：Vultr 、Bandwagon 、DigitalOcean 、 virmach ... 。     

我大概的搜了一下，Vultr 的口碑相对好于其他的，最便宜的虚机是2.5刀一个月，用于上上 Google 完全够用，但是经常处于售罄状态，次便宜的是5刀一个月，性价比很高，而且还支持 Aalipay，那就选它了。

#### 1.1 注册 Vultr 账号

[注册链接](https://www.vultr.com/?ref=7368626){:style="color:#409EFF"} 如果用我推荐的这个 link 注册，貌似会送我10刀，不过无所谓啦，哈哈哈。

#### 1.2 充值
因为在 Deploy 虚机的时候，账户里没钱的话会转到充值页面，所以这里先做充值。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/pay.png){:style="margin:0"}

有4中支付方式，这里我选的是支付宝，比较方便。

默认有5个面额，当然你也可以选 Other。

充值成功会跳转到 Billing 页面，可以查看账户余额。

#### 1.3 Deploy虚机

接下来就是部署虚机了，选左边导航的 Servers 。

首先选择设备节点。 

**推荐选择<span style="color:#409EFF">洛杉矶</span>节点，感觉速度不错，日本节点虽然更近，但是容易被墙。我第一个选的日本节点，SSH 上不去， Ping 都 Ping 不通。只好删掉重新建了一个洛杉矶的。** 

建完虚机发现 IP 被墙的话，删掉重建就好了，会拿到一个新 IP。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/node.png){:style="margin:0"}

其次选择操作系统，按照自己的喜好来吧，不满意之后也可以换。这里我选的 CentOS 7。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/os.png){:style="margin:0"}

再次选择虚机规格，不同规格价格不一样，5刀那个我个人完全够用。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/serve_size.png){:style="margin:0"}

最后选填一些其他信息，默认就好了。不过这里我填了 SSH Key, 为了登录方便。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/install_finish.png){:style="margin:0"}

安装完成的话状态那一项会显示 Running。

### 2. 安装配置 Shadowsocks

#### 2.1 SSH 到 VPS

选一款终端模拟软件， 比如： XShell 、 Putty 、 SecureCRT。

```
ssh root@<host>
```

登录到 VPS。


#### 2.2 安装 Server 端的 Shadowsocks

[teddysun@github](https://github.com/teddysun/shadowsocks_install ) 写了一个一键安装 Shadowsocks Server 的脚本。 

```
wget --no-check-certificate -O shadowsocks.sh https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks.sh
chmod +x shadowsocks.sh
./shadowsocks.sh 2>&1 | tee shadowsocks.log
```

安装过程中会提示选择加密方式，默认回车即可。

安装完成后，会有如下提示。

```
Congratulations, Shadowsocks-python server install completed!
Your Server IP        :your_server_ip
Your Server Port      :your_server_port
Your Password         :your_password
Your Encryption Method:your_encryption_method

Welcome to visit:https://teddysun.com/342.html
Enjoy it!

```

#### 2.2.1 配置 TFO

TCP 三次握手会造成一个 RTT 的延时，因此 TFO 的目标就是去除这个延时，在三次握手期间也能交换数据。优化链接速度。

首先打开 /etc/rc.local ：

```
vi /etc/rc.local
```

在文本最下面添加：

```
echo 3 > /proc/sys/net/ipv4/tcp_fastopen
```

其次打开 /etc/sysctl.conf ：

```
vi /etc/sysctl.conf
```

在文本最下面添加：

```
net.ipv4.tcp_fastopen = 3
```

最后，打开 Shadowsocks 的配置文件:

```
vi /etc/shadowsocks.json
```

把 `"fast_open"` 的 false 替换成 true。

如果想配置多用户的话。

```
{
    "server":"0.0.0.0",
#   "server_port": "8888" 
    "local_address":"127.0.0.1",
    "local_port":1080,
    "port_password":{
        "port_a": "password_1",
        "port_b": "password_2"
    },
    "timeout":300,
    "method":"aes-256-gcm",
    "fast_open":true
}
```
注释掉 `server_port`,  key 由 `password` 改为 `port_password`。

value 是一个 JSON， JSON 的 key 为用户登录的端口，value 为 用户登录的密码。即一个端口对应一个密码。

且配置了多用户之后，需要对新增的端口更改防火墙规则 (tcp 和 udp)。 替换 `<port>` 为自己新加的端口号。

```
iptables -I INPUT -m state --state NEW -m tcp -p tcp --dport <port> -j ACCEPT
iptables -I INPUT -m state --state NEW -m udp -p udp --dport <port> -j ACCEPT 
/etc/init.d/iptables save
/etc/init.d/iptables restart
```

最后重启 Shadowsocks。

```
/etc/init.d/shadowsocks restart
```
  
启动：/etc/init.d/shadowsocks start           
停止：/etc/init.d/shadowsocks stop      
重启：/etc/init.d/shadowsocks restart         
状态：/etc/init.d/shadowsocks status         

#### 2.3 安装客户端的 Shadowsocks 

[windows 版](https://github.com/shadowsocks/shadowsocks-windows/releases){:style="color:#409EFF"} | [Mac 版](https://sourceforge.net/projects/shadowsocksgui/){:style="color:#409EFF"} 

windows 版如下， 配置好 IP、端口、密码、加密方式，就全都搞定了。    

如果配置信息忘了就去 VPS 上 /etc/shadowsocks.json 目录下看配置文件吧。

![enter image description here](http://owu6vks0s.bkt.clouddn.com/ss_client.png){:style="margin:0; width:400px"}

Enjoy ~