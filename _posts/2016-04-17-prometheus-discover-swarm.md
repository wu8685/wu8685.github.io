---
layout: post
title: Prometheus对Swarm的服务发现插件
categories: [golang, docker]
---

最近由于项目需要，自己实现了Prometheus对Swarm的服务发现，以方便收集metrics。代码在[github](https://github.com/wu8685/prometheus)，`retrieval/discovery/swarm`。基于[Prometheus](https://github.com/prometheus/prometheus) 0.17.0和[Swarm](https://github.com/docker/swarm) v1.1.0。
## 系统需求
因为Swarm自身并没有为prometheus提供metrics的输出接口，所以需要在Swarm的每个master和node上跑一个CAdvisor。插件默认CAdvisor的`/metrics`entrypoint默认接口为8070（可配置）。
## 配置
```
- job_name: service-swarm
  swarm_sd_configs:
  - masters:
    - 'http://swarm.example.com:8080'

    refresh_interval: 1s
    metrics_port: '8060'
```
- **refresh_interval** 制定插件去收集metrics的时间间隔；
- **metrics** 制定CAdvisor的`/metircs`端口；
- label policy的相关配置和Kubernetes的一致。

## 原理
#### Prometheus服务发现插件接口
```
// prometheus/retrieval/targetmanager.go

// A TargetProvider provides information about target groups. It maintains a set
// of sources from which TargetGroups can originate. Whenever a target provider
// detects a potential change, it sends the TargetGroup through its provided channel.
//
// The TargetProvider does not have to guarantee that an actual change happened.
// It does guarantee that it sends the new TargetGroup whenever a change happens.
//
// Sources() is guaranteed to be called exactly once before each call to Run().
// On a call to Run() implementing types must send a valid target group for each of
// the sources they declared in the last call to Sources().
type TargetProvider interface {
// Sources returns the source identifiers the provider is currently aware of.
Sources() []string
// Run hands a channel to the target provider through which it can send
// updated target groups. The channel must be closed by the target provider
// if no more updates will be sent.
// On receiving from done Run must return.
Run(up chan<- config.TargetGroup, done <-chan struct{})
}
```
- Sources() []string，返回当前provider的标示。target manager将返回的string作为target group的ID
- Run(up chan<- config.TargetGroup, done <-chan struct{})，启动target provider。provider会将最新的target group信息输出到up这个channel中，以通知target manager。

```
// prometheus/config/config.go

// TargetGroup is a set of targets with a common label set.
type TargetGroup struct {
// Targets is a list of targets identified by a label set. Each target is
// uniquely identifiable in the group by its address label.
Targets []model.LabelSet
// Labels is a set of labels that is common across all targets in the group.
Labels model.LabelSet

// Source is an identifier that describes a group of targets.
Source string
}
```
- Targets，会被prometheus解析，用于获得target metrics的访问信息，会被添加到metrics记录上；
- Labels，普通的label，会被添加到metrics记录上；
- Source，等同于ID。

#### Swarm服务发现原理
##### Nodes
1.通过定时访问Swarm master的REST API`/info`，拿到cluster的最新信息。
```
// 访问swarm master的/info api 得到的response body

{
 "ID": "",
 "Containers": 16,
 "ContainersRunning": 10,
 "ContainersPaused": 0,
 "ContainersStopped": 6,
 "Images": 30,
 "Driver": "",
 "DriverStatus": null,
 "SystemStatus": [
  [
   "Role",
   "primary"
  ],
  [
   "Strategy",
   "spread"
  ],
  [
   "Filters",
   "health, port, dependency, affinity, constraint"
  ],
  [
   "Nodes",
   "2"
  ],
  [
   " hh-yun-k8s-128049.vclound.com",
   "10.199.128.49:2375"
  ],
  [
   " └ Status",
   "Healthy"
  ],
  [
   " └ Containers",
   "8"
  ],
  [
   " └ Reserved CPUs",
   "12 / 25"
  ],
  [
   " └ Reserved Memory",
   "3.75 GiB / 132 GiB"
  ],
  [
   " └ Labels",
   "executiondriver=native-0.2, kernelversion=3.10.0-229.4.2.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=devicemapper"
  ],
  [
   " └ Error",
   "(none)"
  ],
  [
   " └ UpdatedAt",
   "2016-04-05T09:22:57Z"
  ],
  [
   " hh-yun-k8s-128050.vclound.com",
   "10.199.128.50:2375"
  ],
  [
   " └ Status",
   "Healthy"
  ],
  [
   " └ Containers",
   "8"
  ],
  [
   " └ Reserved CPUs",
   "12 / 25"
  ],
  [
   " └ Reserved Memory",
   "3 GiB / 132 GiB"
  ],
  [
   " └ Labels",
   "executiondriver=native-0.2, kernelversion=3.10.0-229.4.2.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=devicemapper"
  ],
  [
   " └ Error",
   "(none)"
  ],
  [
   " └ UpdatedAt",
   "2016-04-05T09:23:29Z"
  ]
 ],
 "Plugins": {
  "Volume": null,
  "Network": null,
  "Authorization": null
 },
 "MemoryLimit": true,
 "SwapLimit": true,
 "CpuCfsPeriod": true,
 "CpuCfsQuota": true,
 "CPUShares": true,
 "CPUSet": true,
 "IPv4Forwarding": true,
 "BridgeNfIptables": true,
 "BridgeNfIp6tables": true,
 "Debug": false,
 "NFd": 0,
 "OomKillDisable": true,
 "NGoroutines": 0,
 "SystemTime": "2016-04-05T17:23:50.830465718+08:00",
 "ExecutionDriver": "",
 "LoggingDriver": "",
 "NEventsListener": 0,
 "KernelVersion": "3.10.0-229.4.2.el7.x86_64",
 "OperatingSystem": "linux",
 "OSType": "",
 "Architecture": "amd64",
 "IndexServerAddress": "",
 "RegistryConfig": null,
 "NCPU": 50,
 "MemTotal": 283536760012,
 "DockerRootDir": "",
 "HttpProxy": "",
 "HttpsProxy": "",
 "NoProxy": "",
 "Name": "hh-yun-k8s-128050.vclound.com",
 "Labels": null,
 "ExperimentalBuild": false,
 "ServerVersion": "",
 "ClusterStore": "",
 "ClusterAdvertise": ""
}
```
2.解析文本，提取出node的信息。将这些信息封装到target group中，通过channel通知target manager
```
func (d *Discovery) Run(up chan<- config.TargetGroup, done <-chan struct{}) {
	defer close(up)

	if tg := d.masterTargetGroup(); tg != nil {
		select {
		case <-done:
			return
		case up <- *tg:  // 将Swarm master的信息通知给target manager
		}
	}

	retryInterval := time.Duration(d.Conf.RefreshInterval)
	update := make(chan []*Node, 10)

	go d.watchNodes(update, done, retryInterval)  // 持续的从master拿node的信息，并放进update channel中

	for {
		select {
		case <-done:
			return
		case nodes := <-update:  // 一旦有更新，则处理
			d.updateNodes(nodes)  // 更新cache中的node信息
			tg := d.nodeTargetGroup()  // 返回封装了最新node信息的target group
			up <- *tg  // 通知target manager
		}
	}
}
```
##### Masters
Swarm master是静态的，通过prometheus配置文件提供。当provider启动后，直接将master信息通知到target manager（见Run方法代码）。
但是访问master获取node信息的时候，添加有rotation机制，以找到当前正在工作的master。
```
func (c *swarmClient) getNodeInfo() (*Info, error) {
	c.masterMu.Lock()
	defer c.masterMu.Unlock()

	for _, master := range c.masters {
		urlStr := fmt.Sprintf("%s/info", master.String())
		req, err := http.NewRequest("GET", urlStr, nil)
		if err != nil {
			return nil, err
		}

		var resp *http.Response
		if c.do != nil {
			// code for testing
			resp, err = c.do(req)
		} else {
			resp, err = c.client.Do(req)
		}
		if err == nil {
			return c.processNodeInfo(resp)
		}

		c.rotateMaster()  // rotate master
	}
	return nil, errors.New("No available master.")
}

func (c *swarmClient) rotateMaster() {
	if len(c.masters) > 1 {
		c.masters = append(c.masters[1:], c.masters[0])  // 换下一个master
	}
}
```
#### 一些想法
- Swarm master的`/info`返回的json文本，格式相当粗糙。直接就是无脑的把`docker -H X.X.X.X:2375 info`命令的输出给转换成了kv格式。所以才会出现` └ `这种符号。给文本解析带来不便；
- 除了REST API的方式，还可以考虑etcd的watch机制，或者Swarm自己的发现机制`docker/docker/pkg/discovery`.
- 通过文本解析的方式去获得node信息，是非常原始的方法，暴力且不可靠。但之所以选择它，主要是考虑到尽量不带入额外的第三方依赖。这样在以后更新Prometheus版本的时候会带来方便（毕竟才0.X.0版本）。