---
title: MQTT基础：如何在Java中使用MQTT
date: 2024-03-08 10:37:17 +0800
categories: [编程, MQTT]
tags:
- MQTT
- 订阅
- 发布
- 消息
image:
  path: https://assets.emqx.com/images/2d6d078236de0738c257bc8604482a3f.png?imageMogr2/thumbnail/1520x684
  width: 100%
---

# 引入客户端

Java语言中使用最为广泛的MQTT客户端库是[Eclipse Paho Java Client](https://github.com/eclipse/paho.mqtt.java)，可使用Maven将其添加到工程依赖：

```xml
<dependencies>
   <dependency>
       <groupId>org.eclipse.paho</groupId>
       <artifactId>org.eclipse.paho.client.mqttv3</artifactId>
       <!-- 可更新为最新版本 -->
       <version>1.2.5</version>
   </dependency>
</dependencies>
```

# 创建MQTT连接

EMQX提供了[免费公共 MQTT 服务器](https://www.emqx.com/zh/mqtt/public-mqtt5-broker)，接入信息如下：

- Broker：**broker.emqx.io**（国内可使用**broker-cn.emqx.io**）
- TCP端口：**1883**
- SSL/TLS端口：**8883**
- 用户：**emqx**
- 密码：**public**

## 普通TCP连接

使用上面提供的MQTT服务器信息，通过如下代码即可连接上MQTT服务器：

```java
// 连接信息
String broker = "tcp://broker.emqx.io:1883";
// TLS/SSL
// String broker = "ssl://broker.emqx.io:8883";
String username = "emqx";
String password = "public";
String clientId = "publish_client";

// 创建客户端
MqttClient client = new MqttClient(broker, clientId, new MemoryPersistence());
MqttConnectionOptions options = new MqttConnectionOptions();
options.setUserName(username);
options.setPassword(password.getBytes(StandardCharsets.UTF_8));

// 发起连接
client.connect(options);
```

在上面的代码中，我们首先创建了`MqttClient`这样一个**同步调用客户端**， 使用阻塞方法通信；`MemoryPersistence`是`MqttClientPersistence`接口的一种实现，`MqttClientPersistence`表示数据持久存储，是用来保存传输过程中出站和入站消息，以支持指定的QoS交付，这里使用了内存保存出站、入站消息的实现方式。

`MqttConnectionOptions`是连接配置项，用于指定连接MQTT服务器的各种参数，常见的参数包括：

- `setUserName`：设置连接MQTT服务器的用户名；
- `setPassword`：设置连接MQTT服务器的密码；
- `setConnectionTimeout`：设置连接MQTT服务器时的超时时间，单位：秒，默认30秒；
- `setAutomaticReconnect`：设置连接丢失时是否自动重连，默认为自动重连；
- `setAutomaticReconnectDelay`：当设置连接丢失自动重连时生效，指定连接丢失后延时多长时间重连，单位：秒；
- `setKeepAliveInterval`：设置心跳间隔，客户端在达到指定间隔未收到消息时，可发起一个`ping`消息，服务端会确认该消息，从而确定服务器是否可用；
- `setSessionExpiryInterval`：设置会话过期时间，该配置决定了客户端在断开连接后，broker保留会话状态的最大时长。如果客户端打算在断开一段时间后连接到服务器，就需要设置较长的会话过期时间；
  - 默认情况下该过期时间设置为`null`，此时会话就不会过期；
  - 如果设置为`0`，会话就会在网络连接断开时立即过期。
- `setCleanStart`：设置客户端与服务器在重新启动或者重新连接时是否记住状态，如果设置为`false`，服务器将保持会话状态，直到 与客户端建立新连接，并将cleanStart标志设置为true。或者关闭网络连接后超过会话过期时间间隔，请参见`setSessionExpiryInterval`；如果设置为true，服务器会立即删除给定客户端的任何现有会话状态，并启用一个新会话。

# 发布消息

`MqttClient`实例创建完成并连接成功后，即可使用该客户端实例作为消息生产者发布消息。`MqttClient`有两个重载方法可以用来发布消息：

```java
// 指定主题、荷载、QoS、是否保留消息
public void publish(String topic, byte[] payload, int qos, boolean retained) throws MqttException, MqttPersistenceException;
// 指定主题、消息对象
public void publish(String topic, MqttMessage message) throws MqttException, MqttPersistenceException;
```

因此我们可以使用这两个API实现MQTT的消息发布：

```java
final String content = "hello, mqtt!";
client.publish("topic/hello", content.getBytes(StandardCharsets.UTF_8), 0, false);

// 或

final String content = "hello, mqtt!";
MqttMessage message = new MqttMessage(content.getBytes(StandardCharsets.UTF_8));
message.setQos(0);
message.setRetained(false);
message.setId(9527);
client.publish("topic/hello", message);
```

# 订阅消息

## IMqttMessageListener

`MqttClient`所实现的接口`IMqttClient`同时提供了4个重载方法用于订阅主题：

```java
public IMqttToken subscribe(String topicFilter, int qos) throws MqttException;
public IMqttToken subscribe(String[] topicFilters, int[] qos) throws MqttException;
// 慎用
public IMqttToken subscribe(String topicFilter, int qos, IMqttMessageListener messageListener) throws MqttException;
// 慎用
public IMqttToken subscribe(String[] topicFilters, int[] qos, IMqttMessageListener[] messageListeners) throws MqttException;
```

`MqttClient`在实现时，又补充了2个重载方法来订阅主题：

```java
public IMqttToken subscribe(MqttSubscription[] subscriptions) throws MqttException;
public IMqttToken subscribe(MqttSubscription[] subscriptions, IMqttMessageListener[] messageListeners) throws MqttException;
```

我们可以使用这些方法来订阅主题，并消费生产者发布的消息：

```java
MqttSubscription[] subscriptions = new MqttSubscription[] {new MqttSubscription("/data/upload/#", 0)};
IMqttMessageListener[] listeners = new IMqttMessageListener[] {(topic, message) -> System.out.println(new String(message.getPayload(), StandardCharsets.UTF_8))};
client.subscribe(subscriptions,  new IMqttMessageListener[]{});
```

这里的`MqttSubscription`表示要订阅的主题，同时可以设置订阅的QoS。`IMqttMessageListener`则用于处理接收到的消息。

### 订阅死循环

但是，`MqttClient`在实现这些方法的时候出现了问题，如下图：

![image-20240308134342337](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403081343495.png)

可以看到`MqttClient`实现`public IMqttToken subscribe(String[] topicFilters, int[] qos) throws MqttException`时出现了死循环，而`public IMqttToken subscribe(String topicFilter, int qos, IMqttMessageListener messageListener) throws MqttException`的实现又直接调用了这个死循环的实现，因此也会出现死循环。

我这里的版本是`1.2.5`,不确定其他版本是否存在此问题。

## MqttCallback

前面我们看到可以使用如下方式订阅消息：

```java
client.subscribe("data/upload/#", 0);
```

但是这时即便接收到订阅的消息，也没有处理的机会。此时就可以通过设置`Callback`回调来处理订阅到的消息：

```java
 client.setCallback(new MqttCallback() {
    @Override
    public void disconnected(MqttDisconnectResponse disconnectResponse) {
        System.out.println("连接断开");
    }

    @Override
    public void mqttErrorOccurred(MqttException exception) {
        System.out.println("发生异常");
    }

    @Override
    public void messageArrived(String topic, MqttMessage message) throws Exception {
        System.out.println("接收到消息" + message.getPayload());
    }

    @Override
    public void deliveryComplete(IMqttToken token) {
        System.out.println("消息发送完成");
    }

    @Override
    public void connectComplete(boolean reconnect, String serverURI) {
        System.out.println("连接完成");
    }

    @Override
    public void authPacketArrived(int reasonCode, MqttProperties properties) {
        System.out.println("认证包到达");
    }
});
```

`MqttCallback`定义了一些方法：

- `disconnected`：客户端与服务器断开连接时触发，可以做重连操作；
- `mqttErrorOccurred`：发生错误时触发，可以处理一些未知错误；
- `messageArrived`：订阅的消息到达时触发，消息处理的逻辑在这里；
- `deliveryComplete`：消息发送成功时触发，作为生产者时可以监听消息发送成功的事件；
- `connectComplete`：客户端连接上服务器时触发，可以做一些初始化逻辑；
- `authPacketArrived`：接收到认证消息包时触发。

使用`IMqttMessageListener`订阅处理消息时，只能在`MqttClient`调用`connect`方法后通过`subscribe`方法指定监听；而`MqttCallback`则需要在`connect`方法前设置。

如果同时设置了`IMqttMessageListener`和`MqttCallback`，则`MqttCallback`不会生效，只有`IMqttMessageListener`在处理订阅到的消息。



