---
layout: post
title:  "How to read OpenStack code part 5 - neutron architecture"
date:   2017-07-12 08:43:59
author: Mingwei Li
categories: openstack
---


今天这一章节非常重要。我们知道neutron是一个非常复杂的系统，由很多组件构成。研究这样一个复杂的系统，正确的顺序应该是现在宏观上对其整体结构有所了解，然后再由针对性的对其组件进行深入了解。本章要做的事情就是介绍neutron 宏观上的架构。

首先看一下下图：



    人 - - >  Neutron Server - - > Plugin - -> message queue - -> Agent
                                  (Extension)
                                       
                                       |
                                       |
                                       
                                      MySQL database


因为是markdown 编辑，所以没办法把图画的太复杂，不过其实也够了。上面就是Neutron的大致结构。

首先，是人发起API请求给Neutron Server。这个过程没什么好说的，就是HTTP restful请求

然后，Neutron Server 接受到请求后，会根据route规则把请求转发给相应的Plugin。这里的route规则，也就是route map是由之前介绍过的route package生成的

Plugin接受到请求后主要做两件事，在DB中创建对象和调用Agent做响应配置。比如创建一个subnet请求，plugin会现在db创建subnet的数据对象，然后调用agent在openvswitch或者linuxbridge创建具体的配置信息。Plugin和db交互用的是sqlalchemy orm package，跟agent交互是通过message queue。


由此可见Plugin是Neutron的核心。不过从图中还可以看到一个叫extension的东西，这里的extension跟我们之前在stevedore中提到的extension不是一回事儿，但在neutron中也非常重要，下面我们大概了解一下plugin 和 extension。

# Plugin 发展历史

Neutron是所谓pluggable结构的，就是说它的很多组件应该可以像积木一样随意插拔使用。

Plugin 和 driver（plugin的底层结构）在neutron社区中被称作third-party code。Kilo版本之前，这些third-party code虽然可以由第三方组织或个人开发，但还是包含在Neutron的代码树中，统一管理在Neutron的代码库中。

Kilo版本中Plugin 和 driver等代码进入了decomposition阶段。这一阶段大部分Plugin和driver的代码把主要的逻辑剥离出来存入了独立的repository。

到了Liberty版本，Plugin等代码基本就可以独立存在于Neutron的代码库之外了。

所以后面的文章我们会尝试编写一个独立的python plugin。所谓独立，即是说该plugin可以由我们自己独立管理，安装和发布。

# Core plugin 和 Service Plugin

Neutron早期的功能很有限，只是支持network/subnet/port/subnet-pool等资源。所以当时的Plugin主要就是管理这些资源（subnet-pool资源是后来引入的），这些Plugin叫做Core Plugin。

Core Plugin主要涉及的都是二层的技术，比如在linuxbridge/openvswitch上配置一些二层网络。但二层上的具体实现技术有很多种，除了linuxbridge，openvswitch之外还有如cisco自己的交换机，huawei，juniper，edison等。各个厂商为了让neutron能够在自己的产品上配置二层网络开发了很多自己的core plugin，所以就出现了很多不同的core plugin。

不过你现在去安装部署neutron会发现大部分时候用的core plugin都是一个叫ml2的core plugin。这是因为社区发现，没必要为每种二层技术开发一个plugin。core plugin要做两件事：

    1.在数据库中创建网络对象。这部分工作不论用哪种二层技术（openvswitch还是linuxbridge）区别都不打，所以单独提炼出来一个type driver来做这件事儿
    2.用具体的二层技术如openvswitch或者linuxbridge来配置二层网络。这部分工作依据二层技术的不同会有较大不同，所以交给mechanism  driver来做

现在不同的二层技术只需要实现具体的mechnisam driver即可。

再后来随着neutron的发展，它可以支持更丰富的内容，如创建router，firewall，loadbalance等。这部分工作在neutron中叫做service。这些service neutron也和之前的资源一样用plugin来管理。不过这部分plugin不叫core plugin叫做service plugin。代码通常放在neutron/services下面，如l3_router。

所以，neutron中的plugin主要分为core plugin 和 service plugin。core plugin用来管理核心资源network/subnet/port/subnet-pool，而service plugin用来管理高层的服务如router。

不过，从neutron的代码中可以发现，其实neutron社区是想把这两种plugin合并成一种的。主要是把core plugin变成 service plugin的一种。neutron/manage.py中有这么一段注释很清楚的阐述了这一发展趋势：

        # core plugin as a part of plugin collection simplifies
        # checking extensions
        # TODO(enikanorov): make core plugin the same as
        # the rest of service plugins




# extension

上面主要再聊plugin，那么什么是extension呢？extension主要用于丰富和扩展现有的API功能，有时新增的功能也用extension来实现，这些新功能不会放入正式的API列表，只有在测试使用过一段时间后，才会放入正是API列表，并且这些extension也会慢慢转为plugin。

目前有下面三种extension

>* Resources extension introduce a new "entity" to the API. 所谓entity就是neutron中的资源，核心的资源有network/subnet/port/subnet-pool，当你想引入一个新的资源的时候，可以先用extension来实现
>* Action extensions tend to be "verbs" that act on a resource.举个例子，tiger是一种资源，但是eat是动作，neutron中通常的做法是用action extension来处理这个动作
>* Request extensions allow one to add new values to existing requests objects. 比如port本来是个很稳定的资源，API也很稳定。但是现在你想在创建PORT的时候添加一个属性，叫description或者area，那么可以通过extension来实现。

下一篇文章我们将看一下core plugin以及extension。