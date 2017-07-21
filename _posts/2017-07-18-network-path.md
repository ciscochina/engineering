---
layout: post
title:  "网络学习和CCIE认证"
date:   2017-07-18 08:43:59
author: Peng Xiao
categories: network
---

本文假设你没有任何网络基础，从头开始，打算在2-3年内达到某一个方向CCIE的水准（不是paper IE）, 以我个人的经历梳理的一个计划。对于每一个部分需要花的时间，不同的人会有不同的情况，要依情况而定。但是总体来讲，在不影响工作的前提下，稳步提升，2-3年是一个比较合理的时间。

我是2010年7月11号开始在思科实习，为期一年，2011年初考了CCNA，然后开始学习CCNP，但是并没有考CCNP的认证，主要时间用来学习OSPF和BGP。从2012年开始准备CCIE，2013年5月获取了CCIE SP的认证。

学习过程中一定要理论+实践，实践要占高的比重，另外如果能做一些笔记记录就更好了。我在那三年时间，大概写了70篇相关技术博客。下面我就按照从基础——>CCNA——>CCNP——>CCIE的顺序，总结一下大概的学习流程。

# 1. 基础篇

在基础篇里，基本不会涉及和接触到具体网络设备，而偏重于理论和概念，但是非常重要，这是学习后面所有知识的基础。

如果大学或者研究生课程学过类似 ``通信网基本概念与主体结构``[[^1]] 或者 ``计算机网络``, ``通信网基础``等课程，那么基础篇可以略过了，因为你已经掌握了**网络分层**，**TCP/IP协议**，**路由算法**的基本概念。（不太自信的话，也可以复习一遍）

### 1.1 TCP/IP基础

如何学习TCP/IP协议，可以参考知乎的一篇问答“如何学习tcp ip协议”[[^2]]. 总结一下基本有这么两点：

1. 理论：TCP,UDP,IP,ARP,ICMP,DHCP,DNS等.
2. 实践: 理论是枯燥的，实践是有趣的
    - 基础实践：使用wireshark抓包分析网页打开的过程，HTTP, TCP三次握手[http://web.engr.oregonstate.edu/~qassimy/index_files/WiresharkLab-HTTP-Ch2-Sol.pdf]()
    - 基础实践：使用熟悉的编程语言写socket小程序，并抓包分析. 例如：[http://www.binarytides.com/python-socket-programming-tutorial/]()
    - 进阶实践：使用scapy做各种发包实验 [https://github.com/secdev/scapy]()

#### 参考资料

1. 《计算机网络》[[^3]]
2. 《TCP/IP详解卷一：协议》[[^4]]

#### 工具

``wireshark``[[^5]]

### 1.2 路由协议基础

主要还是概念性的东西。
- 路由器的基本概念
- 静态路由和动态路由的概念（距离矢量算法和链路状态算法）。
- RIP/OSPF/BGP基本概念

# 2. CCNA篇

在基础篇的基础下，增加对路由器交换机设备的接触，可以以CCNA学习指南[[^6]]为参考资料，以Cisco Packet tracer[[^7]]为模拟器进行试验操作。
在操作上，基本上要达到：

- 交换机路由器telnet的配置
- 对于交换机路由器的接口IP配置
- VLAN的基本配置
- 路由协议RIP/EIGRP/OSPF的简单配置
- 基本的show命令
- 基本的网络排错能力（ping，traceroute，telnet，arp）

结果能在不背诵题库，全部理解的前提下通过CCNA的考试。（我个人大概花了小半年的时间在CCNA上，2010年低-2011年初，记录了一些博客[http://blog.sina.com.cn/s/articlelist_1263548705_10_1.html]()）

# 3. CCNP篇

我个人没有考过CCNP，个人感觉如果不考试，NP阶段的学习重点是各种路由协议，这个学习不是简单会配置（在入门篇已达到这个水平），而是要深入学习。
基本方式就是GNS3[[^8]]（或者IOL）加wireshark抓包分析，配置手册和相关的RFC也需要看看。此时的实验拓扑一般比较小，不太会出现IE那种复杂拓扑。

我下面以OSPF和BGP为例，其它协议RIPv2，EIGRP，ISIS类似。

## 3.1 OSPF

主要就是做实验做实验！可以参考OSPF命令行配置手册。
《Cisco OSPF Command and Configuration Handbook》[[^9]]

- OSPF的邻接怎么建立起来（抓包分析）
- OSPF的五种数据包类型，以及在建立邻接关系过程中各起到什么作用？
- OSPF的特殊区域类型
- OSPF的LA类型（抓包）
- OSPF Area

## 3.2 BGP

《Cisco BGP-4 Command and Configuration Handbook》[[^10]] ，RFC 4271[[^11]]

- BGP邻居建立过程（抓包分析）
- BGP Message类型（Open，Update，Keepalive，Notification）
- IBGP，RR，RR Client，EBGP的概念和配置
- 基本的BGP Route Policy

如果打算考CCNP，建议再看看题库，考试大纲，指南。个人不太建议CCNP，其实直接选一个方向深入进去，准备CCIE就可以的。

# 4. CCIE篇（SP方向）

CCIE的方向有很多，比较大众的Routing Switch方向和Service Provider方向，因为项目需要，我需要专攻BGP，所以CCIE选择了SP方向，

SP方向的重中之重是MPLS VPN。必看书籍为《MPLS和VPN体系结构》[[^12]]，以卷1为主，卷2为辅。另外还有一些组播和l2vpn的内容，建议不要一上来就做CCIE的lab题库，要先过知识点，每一个知识点都要动手做实验。CCIE我大概写了28篇相应的博客[http://blog.sina.com.cn/s/articlelist_1263548705_15_1.html]()，另外网络上资源也非常多，各种博客，视频也是学习的好工具。

这样到最后，再去做lab题库，会非常顺手，然后考前准备就是一个敲命令行的熟练度问题了.

## 4.1 试验环境的搭建

CCIE SP方向的考试，实验拓扑的设备基本都是IOS-XR, 很多配置和IOS语法略有不同。真正准备考试的拓扑可以使用XRv[[^13]]去搭建。XRv比较消耗系统资源，所以在平时知识点学习的时候，个人还是建议使用GNS3.
因为机制是一样的，只不过配置语法不同，关键是GNS3“即插即用”，搭建拓扑拖拖拽拽非常简单，更重要的是，通过wireshark抓包非常方便，这点非常重要。对于IOS-XR配置的训练可以从后期准备实验真题练习开始。

## 4.2 IGP

CCIE SP方向考试，IGP比较简单，基本就是IPv4+IPv6单区域的OSPF或者ISIS。在考试中，IGP是后面所有的配置的基础，特别是BGP。因为IBGP neighbor都是以Loopback0作为update-source，所以要保证一个BGP AS里所有的loopback0接口都能相互ping通（IPv4+IPv6）。要熟悉ping，扩展ping，traceroute等trouble shooting工具，因为错误在所难免，少配置一行，多一行很常见，所以要有基本的trouble shooting能力。

## 4.3 BGP

对于IBGP，EBGP，RR-client配置要非常熟悉。然后就是MPLS VPN的基本配置，PE，CE。最后是Inter-AS VPN那几种option，以及CSC。

trouble shooting还是一样的，各种ping，show命令一跳跳查找故障点。

## 4.4 组播

组播刚看概念会比较抽象一些，所以我做了很多实验，包括使用Python和一些软件自己建立组播源。大家可以参考[http://blog.sina.com.cn/s/articlelist_1263548705_15_1.html]()

## 4.5 其它

参考考试大纲[[^14]]。

# 5. 后续

学习网络知识并不是为了考一个CCIE认证，当拿到一个认证以后，后期还是要继续学习的，一是知识会遗忘，二是新技术层出不穷，要多多去接触，比如SDN，container等等。

# Reference

[^1]: 通信网基本概念与主体结构 [https://item.jd.com/10078500.html]()
[^2]: 如何学习TCP/IP协议？[https://www.zhihu.com/question/28943943]()
[^3]: 计算机网络 [https://book.douban.com/subject/2970300/]()
[^4]: 《TCP/IP详解卷一：协议》[https://book.douban.com/subject/1088054/]()
[^5]: WireShark [https://www.wireshark.org/]()
[^6]: CCNA学习指南 [https://book.douban.com/subject/2968802/]()
[^7]: Cisco Packet tracer [https://www.netacad.com/about-networking-academy/packet-tracer/]()
[^8]: GNS3 [https://www.gns3.com/]()
[^9]: Cisco OSPF Command and Configuration Handbook [http://www.ciscopress.com/store/cisco-ospf-command-and-configuration-handbook-9781587050718]()
[^10]: Cisco BGP-4 Command and Configuration Handbook [http://www.ciscopress.com/store/cisco-bgp-4-command-and-configuration-handbook-9781587050176]()
[^11]: RFC 4271 [https://tools.ietf.org/html/rfc4271]()
[^12]: MPLS和VPN体系结构 [https://book.douban.com/subject/4896791/]()
[^13]: Cisco XRv [https://docs.gns3.com/appliances/cisco-iosxrv.html]()
[^14]: CCIE Service Provider [http://www.cisco.com/c/en/us/training-events/training-certifications/certifications/expert/ccie-service-provider.html]()