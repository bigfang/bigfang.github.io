---
title: 过上了在线看1080p的幸福生活
date: 2017-08-08 15:30:24
tags:
  - tips
  - kcptun
  - linux
categories:
  - 折腾
---

### kcptun和BBR

心血来潮体验了一把kcptun，从此过上了在线看1080p无压力的幸福生活。

然后又试了一把BBR，生活质量同样也达到了1080p。

kcptun安装配置并不复杂，不过调参挺麻烦的；BBR需要内核版本>=4.9，用`uname -r`查一下，配置也还是蛮简单的：

```shell
sudo modprobe tcp_bbr
sudo sysctl net.core.default_qdisc=fq
sudo sysctl net.ipv4.tcp_congestion_control=bbr
```

稍稍解释一下这些命令:

`modprobe tcp_bbr`载入名为tcp_bbr的内核模块，可以使用命令`lsmod | grep bbr`来确认是否载入成功。

`sysctl net.core.default_qdisc=fq`用来配置队列规则(queuing discipline)算法为`fq`，`qdisc`的算法跟流量控制密切相关，目前默认的是`fq_codel`，和`fq`的一个区别是没有实现一个名为`pacing`的特性，而BBR正好需要这个特性才能正常工作。如果开启了BBR而该选项依然用默认的`fq_codel`，会导致丢包率增加。在未来的linux 4.13中，`fq_codel`也将支持`pacing`。也就是说4.13之后，这条命令就可以不用执行了。

运行`sysctl net.ipv4.tcp_available_congestion_control`可以查看可用的拥塞控制算法，用`sysctl net.ipv4.tcp_congestion_control=bbr`选择使用BBR

要注意的是，这些命令改变的是运行中的系统，也就是说重启之后这些配置就没有了。你会发现，`sysctl`命令后面跟着的那些长长的参数，跟`/proc/sys`目录有着神奇的对应关系，至于为什么，那就是另一个故事了。


### kcptun对比BBR

* kcptun是双端优化，需要安装一个客户端；BBR是单端优化，只需要服务端开启
* kcptun以带宽换网速，据说带宽消耗还是蛮可观的；BBR似乎不会特别消耗带宽
* kcptun若要达到最优效果，需要根据网络状况进行配置，配置项复杂又专业；BBR没啥配置，开启即可
* 我个人感觉，kcptun比BBR稍微快了那么一点点
* 两者同时使用，使我产生了生活质量又上升了一点点的错觉
