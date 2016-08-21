---
layout: post
title: Docker Embedded DNS
categories: [docker]
---

因为工作需要，去学习了下Docker的embedded DNS. 这个功能似乎是1.10才加进来的，用来对Docker自带的overlay网络提供DNS服务（容器）发现。我学习的是Docker1.10.3版本，对应的libnetwork是release/v0.7。这个版本的Embedded DNS仅支持IPv4，后续的1.11版本还会支持IPv6.
# Container Network Model（CNM）
Docker的容器网络模型如下图所示：
![CNM from github.com/docker/libnetwork]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/cnm.jpg" | prepend: site.baseurl }})
其中
* 图中少了一个关键组件：`NetworkController`，他是一个Docker Daemon唯一的，用来管理Docker的所有网络环境，跟Docker Daemon打交到；
* 每个Sandbox对应一个Container；
* 每个Network对应了一个网络环境（logical connectivity zone），他背后对应一个network driver，一些关键过程是调用其api；
* 每个Endpoint对应一个Sandbox到Network的逻辑链接（logical connection）.

# How it works
怎么工作的，其实可以从两个角度来看
#### 从Container的角度
![Docker Network Embedded DNS]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/network_on_embeded_dns.png" | prepend: site.baseurl }})
1. Sandbox每次通过一个Endpoint去Join一个Network都会去发布一把这个ep，`sb.populateNetworkResources(ep)`. 其中回去判断一把这个Network是否需要resolver，即Embedded DNS. 如果是用户定义的Network则会enable embedder DNS功能`sb.startResolver()`。anyway，Sandbox都会将这个ep的信息保存到NetworkController的一个Map解构中`networkController.svcDB`。他保存了每个Network中的每个ep的name、alias和分配的IP的对应关系。是跨host的name resolution的根据。（`libnetwork/sandbox.go`）
2. 在`sb.startResolver( )`方法中会做embedded DNS的初始化和启动工作：
  1) `sb.rebuildDNS()`更新/etc/resolv.conf文件，将servername指向127.0.0.11。（**127.0.0.0/8网段都是loopback地址**）
![resolv.conf in container]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/resolv_conf.png" | prepend: site.baseurl }})
  2) `sb.resolver.SetExtServers(sb.extDNS)`设置外部DNS与host一致，当embedded DNS不能resolve name的时候就delegate到extDNS。
  3) `sb.osSbox.InvokeFunc(sb.resolver.SetupFunc())`在Container环境中申请tcp和udp两个*随机*端口，姑且命其为tcp_port和udp_port。
  4) `sb.resolver.Start()`启动embedded DNS：
     * `r.setupIPTable()`添加iptables规则将`127.0.0.11:53`的name resolution query通过DNAT转发到`127.0.0.11:tcp_port`或`127.0.0.11:udp_port`（DNS query默认是udp，效率高）。将出去的query通过SNAT转换回`127.0.0.11:53`。
     * 调用`github.com/miekg/dns`包的api启动两个DNS server分别监听tcp_port和udp_port。
![iptables and port info]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/iptables_rules.png" | prepend: site.baseurl }})
3. 当DNS query来的时候，resolver会去handle：`r.ServeDNS( )`。resolver会在方法`r.handleIPQuery( )`里去resolve name。他会delegate给Sandbox`sb.ResolveName( )`，然后是去NetworkController里的`SvcDB`里查找：。没找到就delegate到ExtDNS。

#### 从Docker Daemon的角度
![Docker Network Embedded DNS Process \(zoom-in-able\)]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/embeded_dns_process.png" | prepend: site.baseurl }})
其中紫色部分为上一节关于embedded DNS的启动过程。（*图片可通过浏览器放大。。。*）
这部分是整个环境的搭建流程：
1. 在Daemo被创建的时候`NewDaemon( )`会初始化网络环境`d.initNetworkController( )`，其中会创建NetworkController`libnetwork.New( )`。此时，会在NetworkController里开启一个loop`c.startWatch( )`，一直监听`watch`和`unwatch`两个channel。这两个channel分别接受**被创建的Endpoint实例**和**被删除的Endpoint实例**然后分别调用`c.processEndpointCreate( )`和`c.processEndpointDelete( )`分别去维护NetworkController里的SvcDB里的ep name/alias和IP的对应关系。
2. 当调用`n.CreateEndpoint( )`或`ep.Join( )`时，会往NetworkController的`watch`channel里放入当前的ep，从而完成一个DNS record的注册
3. 当调用`ep.Delete( )`时，会往NetworkController的`unwatch`channel里放入当前ep，以完成这个DNS record的注销。

整个embedded DNS的环境就是这个样子。