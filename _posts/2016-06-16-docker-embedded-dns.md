---
layout: post
title: Docker Embedded DNS
categories: [docker]
tags: [docker, embedded dns]
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

#### 什么情况下会使用embedded DNS

方法`Daemon.connectToNetwork()`是被调用来将一个container连接到某个网络的。

``` golang
func (daemon *Daemon) connectToNetwork(container *container.Container, idOrName string, endpointConfig *networktypes.EndpointSettings, updateSettings bool) (err error) {
	if endpointConfig == nil {
		endpointConfig = &networktypes.EndpointSettings{}
	}
	n, err := daemon.updateNetworkConfig(container, idOrName, endpointConfig, updateSettings)
	if err != nil {
		return err
	}
	if n == nil {
		return nil
	}

	controller := daemon.netController

	sb := daemon.getNetworkSandbox(container)
	createOptions, err := container.BuildCreateEndpointOptions(n, endpointConfig, sb)  // <--------
	if err != nil {
		return err
	}

	endpointName := strings.TrimPrefix(container.Name, "/")
	ep, err := n.CreateEndpoint(endpointName, createOptions...)
	if err != nil {
		return err
	}
  ......
```

其中`BuildCreateEndpointOptions`方法返回了创建一个endpoint所需要的option。而此方法中的如下代码直接指定了是否要使用embedded DNS。

``` golang
  if !containertypes.NetworkMode(n.Name()).IsUserDefined() {  // using embedded DNS if not user defined network mode
    createOptions = append(createOptions, libnetwork.CreateOptionDisableResolution())
  }
```

在package `github.com/docker/engine-api/types/container`中定义的方法`IsUserDefined()`会判断网络名不为`default`, `bridge`, `host`, `none`, `container`时，则认为是用户定义网络，进而enable了embedded DNS。

``` golang
// IsDefault indicates whether container uses the default network stack.
func (n NetworkMode) IsDefault() bool {
	return n == "default"
}

// NetworkName returns the name of the network stack.
func (n NetworkMode) NetworkName() string {
	if n.IsBridge() {
		return "bridge"
	} else if n.IsHost() {
		return "host"
	} else if n.IsContainer() {
		return "container"
	} else if n.IsNone() {
		return "none"
	} else if n.IsDefault() {
		return "default"
	} else if n.IsUserDefined() {
		return n.UserDefined()
	}
	return ""
}

......

// IsUserDefined indicates user-created network
func (n NetworkMode) IsUserDefined() bool {
	return !n.IsDefault() && !n.IsBridge() && !n.IsHost() && !n.IsNone() && !n.IsContainer()
}
```

而整个embedded DNS的生命周期，其实可以从两个角度来看

#### 从Container的角度
![Docker Network Embedded DNS]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/network_on_embeded_dns.png" | prepend: site.baseurl }})
1. Sandbox每次通过一个Endpoint去Join一个Network都会去发布一把这个ep，`sb.populateNetworkResources(ep)`. 其中回去判断一把这个Network是否需要resolver，即Embedded DNS. 如果是用户定义的Network则会enable embedder DNS功能`sb.startResolver()`。anyway，Sandbox都会将这个ep的信息保存到NetworkController的一个Map解构中`networkController.svcDB`。他保存了每个Network中的每个ep的name、alias和分配的IP的对应关系。是跨host的name resolution的根据。（`libnetwork/sandbox.go`）

2. 在`sb.startResolver( )`方法中会做embedded DNS的初始化和启动工作：

  1) `sb.rebuildDNS()`首先会**记录**初始时容器内部的/etc/resolv.conf文件的内容，然后更新/etc/resolv.conf文件，将servername指向127.0.0.11。（**127.0.0.0/8网段都是loopback地址**）
![resolv.conf in container]({{ "/img/posts/2016-06-16-docker-embeded-dns.md/resolv_conf.png" | prepend: site.baseurl }})

  2) `sb.resolver.SetExtServers(sb.extDNS)`将之前读取的容器初始/etc/resolv.conf文件的nameserver作为embedded DNS的recursive DNS，当embedded DNS不能resolve name的时候就delegate到extDNS。**注意**`这也是为什么--nameserver参数还能工作的原因`，虽然容器内的/etc/resolv.conf文件显示的nameserver还是127.0.0.11

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