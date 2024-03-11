---
title: MQTT基础：MQTT的QoS
date: 2024-03-08 14:15:01 +0800
categories: [编程, MQTT]
tags:
  - MQTT
  - QoS
  - 最多一次
  - 精确一次
  - 至少一次
image:
  path: https://assets.emqx.com/images/b934a6d38d5fcd548e234bc7e691c0f4.png?imageMogr2/thumbnail/1520x684
  width: 100%
---

# 理解QoS

## 什么是QoS

`QoS`是`Quality of Service`的缩写，即服务质量。通常情况下使用MQTT协议的设备都是运行在网络受限的环境下，仅依靠底层TCP协议并不能完全保证消息的可靠到达。为此MQTT提供了`QoS`机制，它设计了多种消息交互机制来提供不同的服务质量，从而满足用户在不同场景下对消息可靠性的需求。

MQTT定义了三个`QoS`等级：

- `QoS 0`：最多交付一次；
- `QoS 1`：至少交付一次；
- `QoS 2`：精确交付一次。

其中，`QoS 0`级别由于最多只会投递一次消息，如果客户端消费失败，可能出现消息丢失；`QoS 1`则为了保证客户端能够收到消息，可能出现消息重复；`QoS 2`可以做到精确投递一次消息，做到既不会丢失消息也不会重复。`QoS`等级越高，消息可靠性就越高，也意味着传输复杂程度越高。

> **注：**
>
> 需要明确的是，MQTT的设计中并没有提供直接的机制来确保订阅者一定会接收到消息。QoS 1和QoS 2是确保消息发布者的消息能到达消息代理服务器；但不能保证消息代理服务器的消息一定到达订阅者。比如订阅者在消息发送后离线，或者其他原因无法接收消息。

## QoS的降级

正常情况下，`QoS等级`由发布者在发布消息时指定，大部分情况下Broker向订阅者转发消息时会维持原始`QoS`不变。如下：

```java
// 参数 2 即表示QoS等级为2
client.publish("topic", "hello, mqtt".getBytes(StandardCharsets.UTF_8), 2, false);
```

此时订阅者订阅此消息，`QoS`依然为2：

```java
MqttSubscription[] subscriptions = new MqttSubscription[] {new MqttSubscription("topic")};
client.subscribe(subscriptions);
```

但是有时Broker也可以根据订阅者的要求，在转发消息时对消息的`QoS`等级进行降级。比如：

```java
MqttSubscription[] subscriptions = new MqttSubscription[] {new MqttSubscription("topic", 1)};
client.subscribe(subscriptions);
```

同样的发布者发布`QoS`为2的消息，由于客户端声明此主题下`QoS`等级为1，Broker将在转发消息到此订阅者时，将所有`QoS`等级为`QoS 2`的消息降级至`QoS 1`，而本身已经是`QoS 0`或`QoS 1`的消息等级仍按原`QoS`等级转发。

![MQTT QoS 降级](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403081733966.png)

# QoS 0 —— 最多交付一次

`QoS 0`是最低的`QoS`等级，`QoS 0`的消息即发即弃，不需要等待确认，不需要存储和重传。因此对于接收方来说，永远也不会接收到重复的`QoS 0`等级消息。

![MQTT QoS 0](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403081739563.png)

当指定`QoS 0`等级发送消息时，消息的可靠性完全依赖于底层TCP协议。但是TCP协议只能在连接稳定不关闭连接的情况下才能保证消息的可靠性，一旦出现连接关闭、重置，仍有可能丢失当前处于网络链路或操作系统底层缓冲区中的消息。因此，当消息在Broker转发出去以后，未到达客户端消费之前，比如消息在网卡传输阶段，`QoS 0`等级下可能出现消息丢失的情况。

# QoS 1——至少交付一次

`QoS 0`等级下消息可能丢失在某些场景下是不能接受的，因此为保证消息到达，`QoS 1`加入了应答与重传机制，发送方只有在收到接收方的 PUBACK 报文以后，才能认为消息投递成功，在此之前，发送方需要存储该 PUBLISH 报文以便下次重传。

`QoS 1` 需要在 PUBLISH 报文中设置 Packet ID，而作为响应的 PUBACK 报文，则会使用与 PUBLISH 报文相同的 Packet ID，以便发送方收到后删除正确的 PUBLISH 报文缓存。

![MQTT QoS 1](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403082307205.png)

## 为什么QoS 1的消息会重复

为什么`QoS 1`会出现消息重复呢？发送方只有在收到`PUBACK`报文后才认为消息投递成功，反之则重新投递消息。发送方没有能收到`PUBACK`的情况有两种：

- `PUBLISH`报文没有到达接收方；
- `PUBLISH`报文到达接收方，但接收方的`PUBACK`报文尚未到达发送方。

对于前一种情况，其实不存在消息重复的问题，因为接收方还没有接收到过消息。但后一种情况接收方已经收到过该`PUBLISH`报文，发送方因未能收到`PUBACK`报文而重新投递消息，就会导致接收方重复消费消息。

![MQTT QoS 1 重复消息](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403092146834.png)

## DUP = 1的两种情况

消息在重复传递时，`PUBLISH`报文中的`DUP`标志位会被置为1，以表示这是一条重传到报文。但是接收方在接收到报文检测到`DUP`为1时，绝不草率的认为自己已经处理过该消息，还是要把这条消息当作一条全新的消息处理。

这是因为对于接收方来说，`DUP`为1也有两种情况：

![MQTT QoS 1 重复消息](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403092155557.png)

第一种情况发送方是由于没有接收到`PUBACK`报文而重传了`PUBLISH`报文，对于接收方来说两次接收到的`PUBLISH`报文的`Packet ID`是一样的，且第二次收到`PUBLISH`报文的`DUP`标志为1，这确实是重复的消息。

而第二种情况是发送方其实投递了两条消息，第一条消息的`PUBLISH`报文已经完成投递，既发送方收到了接收方的`PUBACK`报文。此时这条`PUBLISH`报文的`Packet ID`又可以重新利用，发送使用该`Packet ID`发送了全新的`PUBLISH`报文，但这一次接收方没有收到报文，发送方就会重传该`PUBLISH`报文，且设置`DUP`为1。这种情况下接收方确实收到两条`Packet ID`一样的`PUBLISH`报文，且第二条报文的`DUP`标志为1，但这两条报文是不同的消息。

由于我们无法区分这两种情况，所以只能让接收方将这些 PUBLISH 报文都当作全新的消息来处理。因此当我们使用 QoS 1 时，消息的重复在协议层面上是无法避免的。

甚至在比较极端的情况下，例如 Broker 从发布方收到了重复的 PUBLISH 报文，而在将这些报文转发给订阅方的过程中，再次发生重传，这将导致订阅方最终收到更多的重复消息。

在下图表示的例子中，虽然发布者的本意只是发布一条消息，但对接收方来说，最终却收到了三条相同的消息：

![MQTT QoS 1 重复消息](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403092211517.png)

以上，就是 QoS 1 保证消息到达带来的副作用。

# QoS 2——精确交付一次

`QoS 2`解决了`QoS 0`和`QoS 1`的消息丢失、消息重复的问题，但显然这一切都是有代价的。`QoS 2`消息投递时需要发送方和接收方进行至少两次的请求/响应流程，因此无论在交付复杂度还是性能开销，较之`QoS 0`和`QoS 1`都更高。

![MQTT QoS 2](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403092216359.png)

上图展示了`QoS 2`的处理流程：

1. 发送方首先将`PUBLISH`报文存储并发送到接收方，与`QoS 1`的等待接收方的`PUBACK`不同，`QoS 2`等级的消息需要接收方返回`PUBREC`报文；
2. 发送方接收到接收方的`PUBREC`报文后，就认为接收方已经收到`PUBLISH`报文，发送方则不会再重传报文，也不能再重传报文。此时发送方即删除本地存储的该条`PUBLISH`报文，然后向接收方再发送一个`PUBREL`报文，通知对端自己准备将本次使用的 Packet ID 标记为可用了。与`PUBLISH`报文一样，发送方需要确保`PUBREL`报文到达接收方，所以也需要一个响应报文，同时`PUBREL`报文需要存储下来，以便后续重传；
3. 接收方收到`PUBREL`报文，即确认后续不会再重传该`PUBLISH`报文，它就会给发送方传递一个`PUBCOMP`的报文以表示自己已经准备好将该`PUBLISH`报文使用的`Packet ID`用于新的消息；
4. 发送方再接收到`PUBCOMP`报文时也就意味着这一次的消息投递完成了，后续再使用相同的`Packet ID`发送报文消息，接收方也知道这是一条新消息了。

## 为什么`QoS 2`消息不会重复？

首先`QoS 2`保证消息不丢失的原理和`QoS 1`是一样的，无需赘述。

而`QoS 2`中新增的`PUBREL`和`PUBCOMP`报文通信流程，即是用来确保消息不会重复。

# 不同 QoS 的适用场景和注意事项

#### QoS 0

QoS 0 的缺点是可能会丢失消息，消息丢失的频率依赖于你所处的网络环境，并且可能使你错过断开连接期间的消息，不过优点是投递的效率较高。

所以我们通常选择使用 QoS 0 传输一些高频且不那么重要的数据，比如传感器数据，周期性更新，即使遗漏几个周期的数据也可以接受。

#### QoS 1

QoS 1 可以保证消息到达，所以适合传输一些较为重要的数据，比如下达关键指令、更新重要的有实时性要求的状态等。

但因为 QoS 1 还可能会导致消息重复，所以当我们选择使用 QoS 1 时，还需要能够处理消息的重复，或者能够允许消息的重复。

在我们决定使用 QoS 1 并且不对其进行去重处理之前，我们需要先了解，允许消息的重复，可能意味着什么。

如果我们不对 QoS 1 进行去重处理，我们可能会遭遇这种情况，发布方以 1、2 的顺序发布消息，但最终订阅方接收到的消息顺序可能是 1、2、1、2。如果 1 表示开灯指令，2 表示关灯指令，我想大部分用户都不会接受自己仅仅进行了开灯然后关灯的操作，结果灯在开和关的状态来回变化。

![MQTT QoS](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403102039658.png)

#### QoS 2

QoS 2 既可以保证消息到达，也可以保证消息不会重复，但传输成本最高。如果我们不愿意自行实现去重方案，并且能够接受 QoS 2 带来的额外开销，那么 QoS 2 将是一个合适的选择。通常我们会在金融、航空等行业场景下会更多地见到 QoS 2 的使用。

# 关于 MQTT QoS 的 Q&A

## 如何为 QoS 1 消息去重？

在我们介绍 QoS 1 的时候讲到，QoS 1 消息的重复在协议层面上是无法避免的。所以如果我们想要对 QoS 1 消息进行去重，只能从业务层面入手。

一个比较常用且简单的方法是，在每个 PUBLISH 报文的 Payload 中都带上一个时间戳或者一个单调递增的计数，这样上层业务就可以根据当前收到消息中的时间戳或计数是否大于自己上一次接收的消息中的时间戳或计数来判断这是否是一个新消息。

## 何时向后分发 QoS 2 消息？

我们已经了解到，QoS 2 的流程是非常长的，为了不影响消息的实时性，我们可以在第一次收到 PUBLISH 报文时，就启动消息的向后分发。当然一旦开始向后分发，后续收到在 PUBREL 报文之前到达的 PUBLISH 报文，都不能再重复分发操作，以免消息重复。

## 不同 QoS 的性能有差距么？

以 EMQX 为例，在相同的硬件配置下进行点对点通信，通常 QoS 0 与 QoS 1 能够达到的吞吐比较接近，不过 QoS 1 的 CPU 占用会略高于 QoS 0，负载较高时，QoS 1 的消息延迟也会进一步增加。而 QoS 2 能够达到的吞吐一般仅为 QoS 0、1 的一半左右。
