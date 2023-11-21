# k8s-pod-traffic-deduplication
针对k8s单个node中的东西向流量进行去重

## 0.背景问题
<img width="539" alt="image" src="https://github.com/yanqiaoyu/k8s-pod-traffic-deduplication/assets/19269618/f3ea1c37-17b9-4551-919c-4e9a32f7ffaa">

上图是一个在k8s的同个node中，两个pod在calico方案下进行通信的大致流程

现在我们有如下这样一个场景：我们需要在每一个pod中起一个抓包进程，把业务流量抓一份上传至某个平台进行分析

如果我们采用的是sidecar方案，那么大致的拓扑如下

<img width="542" alt="image" src="https://github.com/yanqiaoyu/k8s-pod-traffic-deduplication/assets/19269618/f60436c6-b7b0-4eb7-9eb0-0a1d6a830055">

显然，我们会遇到一个问题：单节点内，东西向流量会重复

具体啥意思呢？举个例子：

1.Pod1 与 Pod2现在需要通信

2.由于calico网络方案的特性，他们之间的通信不会过一个网关类的网卡(比如说calico方案的tunl0)，而是直接两个Pod通信

3.那么流量其实只会在Pod1和Pod2的网卡上流动

4.如果此时我们在Pod1和Pod2上都拉起了一个抓包进程，那么Pod1与Pod2之间的交互包，我们会抓到2份，在Pod1抓到1份，在Pod2抓到1份，流量就重复了

## 1.解决方案

### 方案一 在抓流量的时候就去重
这个方案，是我在跟S部门的研发主管聊的时候，他给的一个方案：
1. 在每个Pod中，抓取所有的南北向流量
2. 在每个Pod中，仅抓取东西向流量中的入站(RX)或者出站(TX)方向的流量

最终汇总的时候，可以保证东西向和南北向的流量，都只有一份

问题：
1. 如何才能区分出东西向和南北向的流量呢？IP？
2. 如何区分TX和RX方向的流量？IP？MAC地址？

### 方案二 在采集端的Node去重
1. 在每个Pod中，抓取所有的流量(东西，南北)
2. 每个Pod抓取完成后，在Node节点上落盘一次
3. Node上起一个watchdog，专门用来针对每个Pod所落盘的流量文件做一次合并，然后去重
4. 处理完成后，发送到master汇总，再发送到数据分析中心

问题：
1. 两个Pod所落盘的流量文件如何做对比去重？会不会破坏同一个session中的报文？
2. 去重的方案是啥？目前我能想到的是参考交换机的去重原理，针对TCP头和某些标志位算一次checksum，然后对比

### 方案三 在master节点汇总的时候去重
1. 在每个Pod中，抓取所有的流量(东西，南北)
2. 每个Pod抓取完成后，不落盘，直接发送到master节点
3. master节点完成合并去重，落盘，发送到数据分析中心

问题：
同上


## 2.采取的方案(暂定)

1. 在每一个Pod中起一个抓包进程抓包（关键词：sidecar，gopacket）
2. 抓包的时候不进行去重，全部抓下来，落盘到Node的某个位置（关键词：gopacket）
3. 在Node上起一个进程，针对所有pod抓下来的包进行合并，去重（关键词：joincap，布隆过滤器）
4. 完成后，推送到某个分析中心

## 3.实践

### 3.1 模拟一个生产环境

![image](https://github.com/yanqiaoyu/k8s-pod-traffic-deduplication/assets/19269618/35a6f263-d345-4fec-85b9-ff40eec5f13e)

我搭建了一个模拟某个客户场景的测试环境，具体拓扑如上图




