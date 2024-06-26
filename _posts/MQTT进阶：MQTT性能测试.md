---
title: MQTT进阶：MQTT性能测试
date: 2024-03-25 10:46:53 +0800
categories: [编程, MQTT]
tags:
- MQTT
- emqx-bench
- 性能
- 压测
---

## 下载emqtt-bench

[ZIP包下载地址](https://www.emqx.com/zh/downloads-and-install?product=emqtt-bench&version=0.4.18&os=macOS13&oslabel=macOS+13)

```shell
-- 下载压缩包
wget https://www.emqx.com/zh/downloads/emqtt-bench/0.4.18/emqtt-bench-0.4.18-macos13-amd64-quic.zip

-- 解压获取emqtt-bench
unzip emqtt-bench-0.4.18-macos13-amd64-quic.zip

-- 加入到环境变量
vi ~/.zshrc

	export PATH=~/tools/bin/:$PATH
```

## emqtt-bench命令

emqx-bench共有三个子命令：

- `pub`，用于创建大量客户端执行发布消息的操作；
- `sub`，用于创建大量客户端执行订阅主题，并接受消息的操作；
- `conn`，用于创建大量的连接。

通过如下命令可以查看每个命令的相关参数：

```shell
./emqtt_bench [command] --help

如
./emqtt_bench pub --help
./emqtt_bench sub --help
./emqtt_bench conn --help
```

下面是三个命令涉及的参数清单：

| 参数              | 简写 | 可选值     | 默认值     | 说明                                                         |
| ----------------- | ---- | ---------- | ---------- | ------------------------------------------------------------ |
| --host            | -h   | -          | localhost  | 要连接的 MQTT 服务器地址                                     |
| --port            | -p   | -          | 1883       | MQTT 服务端口                                                |
| --version         | -V   | 3 4 5      | 5          | 使用的 MQTT 协议版本                                         |
| --count           | -c   | -          | 200        | 客户端总数                                                   |
| --startnumber     | -n   | -          | 0          | 客户端数量起始值                                             |
| --interval        | -i   | -          | 10         | 每间隔多少时间创建一个客户端；单位：毫秒                     |
| --interval_of_msg | -I   | -          | 1000       | 每间隔多少时间发送一个消息                                   |
| --username        | -u   | -          | 无；非必选 | 客户端用户名                                                 |
| --password        | -P   | -          | 无；非必选 | 客户端密码                                                   |
| --topic           | -t   | -          | 无；必选   | 发布的主题；支持站位符： `%c`：表示 ClientId `%u`：表示 Username `%i`：表示客户端的序列数 |
| --size            | -s   | -          | 256        | 消息 Payload 的大小；单位：字节                              |
| --qos             | -q   | -          | 0          | QoS 等级                                                     |
| --retain          | -r   | true false | false      | 消息是否设置 Retain 标志                                     |
| --keepalive       | -k   | -          | 300        | 客户端心跳时间                                               |
| --clean           | -C   | true false | true       | 是否以清除会话的方式建立连接                                 |
| --ssl             | -S   | true false | false      | 是否启用 SSL                                                 |
| --certfile        | -    | -          | 无         | 客户端 SSL 证书                                              |
| --keyfile         | -    | -          | 无         | 客户端 SSL 秘钥文件                                          |
| --ws              | -    | true false | false      | 是否以 Websocket 的方式建立连接                              |
| --ifaddr          | -    | -          | 无         | 指定客户端连接使用的本地网卡                                 |

## 使用说明

### 发布

启动 10 个连接，分别每秒向主题 `t` 发送 100 条 Qos0 消息，其中每个消息体的大小为 `16` 字节大小：

```sh
./emqtt_bench pub -t t -h emqx-server -s 16 -q 0 -c 10 -I 10
```

### 订阅

启动 500 个连接，每个都以 Qos0 订阅 `t` 主题：

```sh
./emqtt_bench sub -t t -h emqx-server -c 500
```

### 连接

启动1000个连接：

```sh
./emqtt_bench conn -h emqx-server -c 1000
```

### ssl连接

单向证书：

```bash
./emqtt_bench sub -c 100 -i 10 -t bench/%i -p 8883 -S
./emqtt_bench pub -c 100 -I 10 -t bench/%i -p 8883 -s 256 -S
```

双向证书：

```sh
./emqtt_bench sub -c 100 -i 10 -t bench/%i -p 8883 --certfile path/to/client-cert.pem --keyfile path/to/client-key.pem
./emqtt_bench pub -c 100 -i 10 -t bench/%i -s 256 -p 8883 --certfile path/to/client-cert.pem --keyfile path/to/client-key.pem
```

## 场景压测

### 连接数压测

```sh
emqtt_bench -h 100.153.32.81 -p 30099 -u admin -P 1qaz2wsx -c 50000
```



### 吞吐量压测

