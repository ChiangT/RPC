# Remote Procedure Calls

@Author *ChiangT ➢ https://github.com/ChiangT*
@Date *2022/8/24*

*关于RPC的原理与实现*

------

## 1. Five Pieces of Program

当一个远程调用发起时，涉及到五个部分：user（服务调用方）、user-stub（调用方本地存根）、RPCRuntime（RPC通信者）、server-stub（服务端本地存根）、server（服务端）。

### Consumer

服务调用方的业务：提供所需的接口名与方法、从自己的本地存根中获取执行的结果。

### Provider

服务提供方的业务：提供服务，为自己的本地存根提供方法的具体实现。

### Stub

本地存根的业务：进行类型和参数的转化，防止由于CP双方地址空间隔离引起问题。

- 服务调用方的stub负责解析consumer的函数调用，并按照相应协议进行序列化，打包成可传输的信息交给RPCRuntime；同时将RPCRuntime返回的数据包反序列化成consumer所需的结果，并传递给consumer
- 服务提供方的stub负责将RPCRuntime传来的请求包进行转化，以找到provider处对应的函数；当函数执行完毕后，stub会将执行结果序列化、打包，最后发给RPCRuntime

### RPCRuntime

RPC通信者负责数据包的重传、确认、路由和加密等，consumer与provider各有一个RPCRuntime实例，可靠地将存根传递的数据包传输到另一端。

------

## 2. Four Stages

RPC调用过程可分为四个阶段，分别是服务暴露的过程、服务发现的过程、服务引用的过程、方法调用的过程。

### 服务暴露

服务暴露的过程发生在provider端，具体可分为两种暴露方式：暴露到本地、暴露到远程。

- 暴露到本地：这种方式下RPCRuntime会绑定且监听机器A对应服务的本地端口，若另一台机器B想调用该服务，则需要显式地指定该服务的网络地址；一旦机器A的服务地址变动，B的远程调用就会失败

- 暴露到远程：这种方式首先也是在本地绑定端口，然后provider在服务启动时将地址、端口、服务需暴露的接口信息等注册到注册中心Registry，并与Registry保持心跳保活；若provider端某个节点异常下线，Registry通过保活检查可以发现该情况并将该节点的信息移除，防止consumer把请求发给该下线节点，另外Registry的配置地址一般不会变动，也就不容易产生上一种方式存在的问题

### 服务发现

服务发现的过程发生在consumer端，也就是寻址的过程。当consumer需要调用某个服务时，其首先需要知道提供该服务的provider的地址与端口。服务发现有两种方式：直连式和注册中心式，分别对应provider的两种服务暴露方式。

- 直连式：若provider端的服务仅暴露到本地，则consumer只能通过直连式实现服务发现，这就要求consumer必须实时掌握provider的地址与端口信息
- 注册中心式：若provider端的服务暴露到远程，则consumer可从Registry处获得provider的地址与端口

### 服务引用

当服务发现完成后，consumer会通过负载均衡策略选择其中一个provider进行服务引用。具体过程即为CP双方建立连接，以及在consumer端创建接口的代理。其中，建立连接也就是CP两端RPCRuntime建立连接的过程。

### 方法调用

当服务引用完成后，consumer即可进行方法的调用，具体的方法调用过程如图所示。

<img src="/Users/jtao/Library/Application Support/typora-user-images/截屏2022-08-29 下午10.15.24.png" alt="截屏2022-08-29 下午10.15.24" style="zoom:150%;" />

------

## I/O Model

Java对于I/O模型的封装可分为BIO、NIO和AIO三类。

### BIO : Blocking I/O

BIO的交互方式为同步阻塞方式，即：当一个Java线程在进行read或者write操作时，r/w操作未完成之前，线程会一直处于阻塞状态。

### NIO : Non-blocking I/O

### AIO : Asynchronous I/O



------

## Principle

若要实现调用远程方法像调用本地方法一样简单，首先需要解决如下问题：

1. 如何获取可用的远程服务器——服务注册与发现
2. 如何表示数据——序列化与反序列化
3. 如何传递数据——网络通信
4. 服务端如何确定并调用目标方法——调用方法映射

RPC的整体逻辑架构为：Server启动时将自己的服务节点信息注册到Registry，Client调用远程方法时会从Registry处订阅可用的服务节点信息，通过该服务节点来调用远程方法；当Client订阅的服务节点信息失效时，Registry会向Client发送相应的通知，防止Client继续使用已失效的服务节点来调用远程方法。

Client远程调用方法的具体过程为：

1. Client模块代理所有远程方法的调用
2. 将目标服务、目标方法、调用目标方法的参数等必要信息序列化
3. 序列化之后的数据包进一步压缩，压缩后的数据包通过网络通信传输到目标服务节点
4. 服务节点将接收到的数据包进行解压
5. 解压后的数据包反序列化成目标服务、目标方法、目标方法的调用参数
6. 通过Server代理调用目标方法获取结果，结果同样需要序列化、压缩然后回传给Client

------

## Implementation details

