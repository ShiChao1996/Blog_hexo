---
title: TCP 超时重传
tags: tcp
author:
nick: Lovae

subtitle: 理解TCP协议第二部分，TCP基于定时器重传和基于信息的重传。
cover: https://image.littlechao.top/20180315023638000003.jpg
date: 2018-3-20
categories: network protocol
---
##TCP超时重传

> TCP 提供可靠数据传输服务，为保证传输正确性，TCP 重传其认为已经丢失的包。TCP 有两套重传机制，一是基于定时器（超时），二是基于确认信息的构成（快速重传）。

###基于计时器的重传

#### 简单的超时重传

![](https://image.littlechao.top/20180315023638000003.jpg)

图中黑色那条就是因为定时器超时仍没有收到 ACK，所以引起了发送方超时重传。实际上 TCP 有两个阈值来决定如何重传同一个报文段：一是愿意重传的次数 R1、二是应该放弃当前连接的时机 R2。R1 和 R2 的值应分别至少为 3 次和 100 秒，如果超过任何一个但还没能重传成功，会放弃该连接。当然这两个值是可以设置的，在不同系统里默认值也不同。

那么如何设定一个合适的超时的值呢？假设 TCP 工作在静态环境中，这很容易，但真实网络环境不断变化，需要根据当前状态来设定合适的值。

#### 超时时间 RTO

RTO（retransmission timeout）一般是根据RTT（round trip time）也就是往返时间来设置的。若 RTO 小于   RTT，则会造成很多不必要的重传；若 RTO 远大于 RTT，则会降低整体网络利用率，RTO 是保证 TCP 性能的关键。并且不同连接的RTT不相同，同一个连接不同时间的 RTT 也不相同，所以 RTO 的设置一直都是研究热点。

所以凭我们的直觉，RTO 应该比 RTT 稍大：

​									**RTO=RTT+Δt**

那么，RTT 怎么算呢：

​							**SRTT=(1−α)×SRTT+α×RTTnew**

SRTT(smooth RTT)，RTTnew 是新测量的值。如上，为了防止 RTT 抖动太大，给了一个权值 **a** ，也叫平滑因子。a 的值建议在 10%~20%。举个例子，当前 RTTs=200ms，RTTs=200ms，最新一次测量的 RTT=800ms，RTT=800ms，那么更新后的 RTTs=200×0.875+800×0.125=275ms，RTTs=200×0.875+800×0.125=275ms.



**Δt**如何得到呢？RFC 2988 规定：

​								**RTO=SRTT+4×RTTD**

因此，按照上面的定义，**Δt=4×RTTD**. 而 RTTD 计算公式如下：

​							**RTTD=(1−β)×RTTD+β×|SRTT−RTTnew|**

实际上，RTTD 就是一个均值偏差，它就相当于某个样本到其总体平均值的距离。这就好比你的成绩与你班级平均成绩差了多少。RFC 推荐 β=0.25。

##### 退避指数

根据前面的公式，我们可以得到 RTO。一旦超过 RTO 还没收到 ACK，就会引起发送方重传。但如果重传后还是没有在 RTO 时间内收到 ACK，这时候会认为是网络拥堵，会引发 TCP 拥塞控制行为，使 RTO 翻倍。则第 n 次重传的 RTOn 值为：

​								**RTOn=2^(n−1)×RTO1**

下图是一个例子：

![图片来源：http://blog.csdn.net/q1007729991/article/details/70196099](http://img.blog.csdn.net/20170422182458581?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcTEwMDc3Mjk5OTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如上，在时间为0.22531时开始第一次重传，此后重传时间开始指数增大，（在尝试了8次后停下，说明其 R2 的值可能为8）。

#### 带时间戳的 RTT 测量

前面说了 RTO 的公式，它和 RTT 有关，那么每一次的 RTT 是如何得到的呢？在之前 TCP 连接管理的时候讲过，TCP有一个 TSOPT (timestamp option) 选项，它包含两个时间戳值。它允许发送者在报文中带上一个32比特的时间戳值（TSV），然后接收方 将收到的值原封不动的填入 ACK 报文段中 TSOPT 选项的第二部分，时间戳回显字段（TSER）。发送方收到 ACK 以后，将当前时间戳减去 TSOPT 选项的 TSER 就可得到精确的RTT值。

> 但是这里有很微妙的细节：接收方在收到数据包后，并不是立即发送 ACK，通常会延时“一小会儿”，多等待几个数据包后返回一个累积 ACK。此时接收方将确认时间最近的报文段的 TSV 填入 TSER 发送给发送方。

##### 重传二义性与 Karn 算法

还有另一个重要的细节，如果测量 RTT 的样本出现了超时重传，但是我们收到了 ACK 时无法分辨是对哪一次的确认，这时候 RTT 的值可能是不正确的。

因此，Karn 算法规定：**此时不更新 RTTnew 的值**。并且如果发生再次重传，则采用退避后的 RTO 的值，直到发送成功，退避指数重新设定为 1 。

##### 丢包和失序的情况

假设有三个数据包依次发送，1号和3号先到达，2号数据包由于网络因素最后到达。接收方收到3号时，会发送一个1号的冗余 ACK，然后2号到达，此时会发送一个3号的累积 ACK 表明这三个到达。在这个例子里，3号 ACK 并没有立即返回，发送方收到3号的 ACK 后，根据其 TSER 计算此时的 RTT，就会导致发送方过高的估计 RTT，降低重传积极性，使得 RTO 相应增大，当然这在失序时是有好处的，因为过分积极会导致大量的伪重传。

#### 伪超时与重传

如下图，在发送第四个 ACK 后出现延迟高峰，导致发送方在 RTO 时间内没有收到 5 ~ 8 的 ACK，于是发生重传，然后之前的 ACK 到达，于是又依次发送 6 ~ 8，就导致了不必要的重传。可以用 Eifel 算法来解决（略）。

<img src="https://image.littlechao.top/20180315075511000005.jpg" style="height:400px">

#### 目的度量

从前面可以看出，TCP 可以学习链路特征，如 RTT、SRTT 等，但一旦连接关闭，这些信息就会丢失。即使相同的接收方与发送方建立新的连接，也必须从头开始“学习”。较新的 TCP 实现维护了这些值，在 Linux 中可以通过如下命令查看：

```Shell
ip route show cache [ip]
```

###基于确认信息的重传

####  快速重传

在大多数情况下，计时器超时并触发重传是不必要的，也不是期望的，因为 RTO 通常是大于 RTT（约2倍或更大），因此基于计时器的重传会导致网络利用率降低。

首先我们要知道，接收方收到失序报文段时，应立即生成确认信息（重复 ACK），以便发送方尽快、高效地填补空缺。而发送方在收到重复 ACK 时，无法判断是由于数据包丢失还是仅仅因为延迟，所以发送方等待一定数目的重复 ACK （重复 ACK 阈值，dupthresh），这时可以认为是数据包丢失，即便还未超时，也立即发送丢失的分组。

所以快速重传概括如下：TCP 发送方在观测到至少 dupthresh ( 通常是 3 ) 个重复 ACK，立即重传，而不必得到计时器超时。当然也可以同时发送新的数据。

示例如下：![](https://image.littlechao.top/20180315024222000004.jpg)

#### 包失序与包重复

##### 失序

当然快速重传也会造成一些问题。在轻微失序的情况下(左图)，不会有什么影响。但在严重失序时(右图)，4号数据包延迟到达，接收方发送 4 个冗余 ACK ，让发送方认为 4 号分组丢失，造成伪快速重传。

![](https://image.littlechao.top/20180315092324000007.jpg)

##### 重复

尽管可能性较小，但 IP 协议也可能将单个包传输多次。假如 IP 协议将一个包传输了 4 次，然后发送方接收到 3 个冗余 ACK ，也会让发送方以为分组丢失，导致伪快速重传。

#### 带选择确认的重传

在上一篇文章中提到过 TCP 的 SACK 选项，它通过若干个 SACK 块来帮助发送方知道接收方有哪些空缺，可以减少不必要的重传。

##### 接收端的 SACK 行为

接收端在 TCP 连接建立期间收到 SACK 选项即可生成 SACK。通常来说，当收到失序报文段，接收方就会生成 SACK。

第一个 SACK 块包含的应该是**最近收到的**报文段的序列号范围。由于 SACK 选项空间有限，应尽可能向发送方提供最新信息。其余的 SACK 按先后顺序依次排列，也就是说，该 ACK 报文段除了包含最新接收的序列号信息，还应重复之前的 SACK 信息。这是因为 ACK 报文段是没有重发机制的，可能会丢失，重复提高了其鲁棒性。

##### 发送端的 SACK 行为

发送方应该充分利用 SACK 信息来进行重传，称为**选择性重传**。发送方记录累积 ACK 信息和 SACK 信息，当接收到相应序列号范围内的 ACK 时，在其重传缓存中标记该报文段重传成功。


