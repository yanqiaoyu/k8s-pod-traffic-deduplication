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

### 方案二 在采集端的Node去重

### 方案三 在master节点汇总的时候去重
