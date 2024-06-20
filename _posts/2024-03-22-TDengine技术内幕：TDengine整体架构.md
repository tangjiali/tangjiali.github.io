---
title: TDengine技术内幕：TDengine整体架构
categories: [编程, TDengine]
date: 2024-03-22 10:21:02 +0800
---

## 集群与基本逻辑单元

TDengine的设计基于单节点、单软件系统的不可靠性，基于任何单台计算机的算力及存储能力都无法处理海量数据的假设进行。因为TDengine按照分布式高可靠架构进行设计，支持水平扩展，从而避免服务器故障或软件错误时影响系统的可用性及可靠性。同时，TDengine通过节点虚拟化及负载均衡技术，高效利用异构及群众的计算和存储资源降低硬件投入。

### 主要逻辑单元

下图为TDengine官网提供的分布式架构的逻辑结构图：

![TDengine Database 架构示意图](https://raw.githubusercontent.com/tangjiali/note_asserts/master/齐简笔记/202403221022958.webp)

在TDengine的设计中，一个完整的TDengine系统运行在一个或多个物理节点，逻辑上包含数据节点（dnode）、TDengine应用驱动（taosc）以及应用（app）。TDengine系统中包含一个或多个数据节点，这些数据节点组成一个集群（cluster）。应用通过taosc的API与TDengine集群进行交互。

#### 物理节点（pnode）



#### 数据节点（dnode）

#### 虚拟节点（vnode）

#### 管理节点（mnode）

#### 计算节点（qnode）

#### 流计算节点（snode）

#### 虚拟节点组（VGroup）

#### 应用驱动（Taosc）

### 节点通讯