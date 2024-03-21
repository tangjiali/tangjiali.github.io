---
title: MQTT实践：Vertx MQTT Client未设置maxMessageSize导致客户端断开连接问题
date: 2024-03-21 08:55:58 +0800
categories: [编程, MQTT]
tags:
  - MQTT
  - Vertx
  - MqttClient
  - MaxMessageSize
---

## 问题描述

```java
// 连接配置
MqttClientOptions options = new MqttClientOptions()
            .setClientId("test_9527")
            .setUsername("admin")
            .setPassword("admin")
            .setCleanSession(true)
            .setKeepAliveInterval(20)
            .setAutoKeepAlive(true)
            .setWillQoS(2);

// 创建客户端
MqttClient client = MqttClient.create(Vertx.vertx(), options);

// 发起连接
client.connect(1883, "127.0.0.1", res -> {
    if(!res.succeeded()){
        return;
    }
	
    // 订阅主题
    client.subscribe("$share/test/data/upload/+", 1, sub -> {
        if(!sub.succeeded()){
            System.out.println("订阅失败");
        }
    });
    
    // 消息处理器
    client.publishHandler(msg -> {
        log.info("topic={}, msg={}", msg.topicName(), msg.payload().toString(StandardCharsets.UTF_8));
    });

    // 连接断开处理器
    client.closeHandler(event -> {
        System.out.println("客户端已关闭");
    });
});
```

如上是昨天为验证线上MQTT消息订阅代码而编写的一段Demo代码，这里使用了`Vertx`的MQTT客户端库，创建客户端并订阅主题，处理订阅的消息。

但是这段代码遇到一个问题，MQTT客户端在连接成功后，很快就断开了连接。但之前使用`eclipse paho java mqtt client`库时，并没有出现类似情况。确认过无论是Broker的`host`、`username`、`password`均没有问题，网络也稳定，只好另想办法解决。

## 问题定位

为了找出连接断开的规律，我在连接断开的处理器中加上了重连的逻辑：

```java
// 连接断开处理器
client.closeHandler(event -> {
    System.out.println("客户端已关闭");
    
    // 重新发起连接
    reconnect(client);
});
```

观察日志，发现MQTT客户端还是会断开，但断开前发送消息，客户端都可以正常消费到。客户端连接断开的规律是连接发起15s断开。起初我以为是Vertx MQTT客户端库有什么默认设置导致15s后自动断开连接，但搜索引擎以`Vertx MQTT 15s`没有找到任何结果。

> 注：
>
> 这里后来发现是因为订阅的主题有生产者每15s生产一条消息，该消息触发了一段逻辑，导致连接断开。

经过网上各种查找资料未果，只好自己想办法排查。因为问题的现象是客户端自动断开连接，而在代码中恰好有断开连接时会触发的处理器，于是在连接断开处理器的首行代码打上了断点，以便溯源触发断连的原因。

![image-20240321093904131](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403210939260.png)

以`Debug`模式重新执行代码，连接断开后，执行停在了断点位置，可以看到断开连接前的栈帧情况：

![image-20240321094136983](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403210941028.png)

顺着栈帧逐一排查，重点关注可能涉及断开连接的位置，然后就发现了这样一处：

![image-20240321094635145](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403210946231.png)

从方法名可以看出，这里在处理消息，但向上看一行，就会发现这里触发了一个异常。既然有异常，那自然是要看一看的：

![image-20240321094844809](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403210948861.png)

于是得到了这样的提示消息：

```
io.netty.handler.codec.TooLongFrameException: too large message: 119073 bytes
```

这是在提示消息过大，有了这么具体的提示，于是又把这段异常提示作为关键词进行搜索，果然找到了问题的根因。

https://github.com/vert-x3/vertx-mqtt/issues/191

## 根因分析

从上面的`issue`可以看到，问题的根本原因是在发起MQTT客户端连接时，使用的配置`MqttClientOptions`没有设置消息最大长度(`setMaxMessageSize`)，于是`MqttClientImpl`类的`initChannel`方法使用了默认的 MqttDecoder 的构造方法，而默认的构造方法设置了 `MqttDecoder `的 `maxBytesInMessage` 值为 8092，大小超过此限制的消息会处理失败。核心代码如下：

```java
private void initChannel(NetSocketInternal sock) {

    ChannelPipeline pipeline = sock.channelHandlerContext().pipeline();

    pipeline.addBefore("handler", "mqttEncoder", MqttEncoder.INSTANCE);

    // MqttClientOptions的maxMessageSize默认值是-1
    if (this.options.getMaxMessageSize() > 0) {
      pipeline.addBefore("handler", "mqttDecoder", new MqttDecoder(this.options.getMaxMessageSize()));
    } else {
      // max message size not set, so the default from Netty MQTT codec is used
      pipeline.addBefore("handler", "mqttDecoder", new MqttDecoder());
    }
    
	// 省略无关代码...
}
```

可以看到`maxMessageSize`是否设置的不同就在于构造`MqttDecoder`时调用的构造方法：![image-20240321101310476](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403211014962.png)

显然如果没有设置`maxMessageSzie`，`MqttDecoder`的`maxBytesInMessage`默认就被设置为`8092`。于是在解码消息时，可以看到如下代码：

![image-20240321101454904](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403211014959.png)

正是这里触发了消息过长的异常，从而导致客户端自动断开。

## 解决方案

解决方法也比较简单，既然是消息长度超过了默认的`8092`，那就配置一个更长的可接收的消息长度：

```java
MqttClientOptions options = new MqttClientOptions();
// 这里极端了，如果可以评估最大长度，设置一个略大的即可
// 暂未验证maxMessageSize过大会不会导致内存浪费问题
options.setMaxMessageSize(Integer.MAX_VALUE);
```

当然，这里还有一个问题，程序发生了异常，但是我们在日志中看不到任何信息，也无从捕获异常，导致我们根本不知道程序发生了异常。实际上`Vertx MQTT Client`是提供了异常处理器的，我们在使用`Vertx MQTT Client`时，还是应该实现该异常处理器，以便及时发现问题。

```java

client.exceptionHandler(event -> {
	log.error("MQTT客户端消费时发生了异常", event);
});
```

![image-20240321102910947](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403211029009.png)