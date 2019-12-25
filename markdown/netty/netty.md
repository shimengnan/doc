# Netty

## 选择netty 的理由

- NIO 类库 API 复杂，需要熟练掌握Selector，ServerSocketChannel，SocketChannel，ByteBuffer 等
- 需要额外的技能铺垫。需要掌握多线程编程（涉及Reactor 模式）
- 可靠性能力补齐，工作量和难度大。（断联重连，网络闪断，半包读写，失败缓存，网络拥塞，异常码流等问题。
- JDK NIO 的BUG 如 epoll bug ，导致Selector 空轮训，最终导致CPU 100%

## 构建Netty 应用步骤

### NIO Server 端

```java
    public void connect(int port) throws InterruptedException {
        //1.配置NIO 线程组
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
          	//2创建NIO server 端辅助启动类
            ServerBootstrap serverBootstrap = new ServerBootstrap();
			//2.1设置NIO 线程组
            serverBootstrap.group(bossGroup, workerGroup)
            //2.2指定Channel 
            .channel(NioServerSocketChannel.class)
            .option(ChannelOption.SO_BACKLOG, 100)
            .handler(new LoggingHandler(LogLevel.DEBUG))
            //2.3指定 socketChannel 用于新的连接接入时的操作。
            .childHandler(new ChannelInitializer<SocketChannel>() {
                @Override
                protected void initChannel(SocketChannel socketChannel) throws
                    Exception {
                    socketChannel.pipeline().addLast(new FixedLengthFrameDecoder(5));
                    socketChannel.pipeline().addLast(new StringDecoder());
                    socketChannel.pipeline().addLast(new TelnetServerHandler());
                }
            });
            //绑定监听地址，并返回异步操作对象
            ChannelFuture channelFuture = serverBootstrap.bind(port).sync();
            //依靠异步操作对象 channelFuture 调用关闭方法 并等待到 channel 关闭，main 继续执行。
            channelFuture.channel().closeFuture().sync();
        }finally {
            //优雅的关闭NIO线程组，释放资源
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
```

### NIO Client 端

```java
    public void connect(int port,String host) throws Exception {
        //创建NIO 线程组
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            //创建NIO client 端辅助启动类
            Bootstrap bootstrap = new Bootstrap();
            // 指定线程组
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {
                            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes());
                            socketChannel.pipeline().addLast(new DelimiterBasedFrameDecoder(2048,delimiter));
                            socketChannel.pipeline().addLast(new StringDecoder());
                            socketChannel.pipeline().addLast(new EchoClientHandler());
                        }
                    });
            //发起异步连接操作
            ChannelFuture channelFuture = bootstrap.connect(host, port).sync();
            //等待客户端链路关闭
            channelFuture.channel().closeFuture().sync();
        }finally {
            group.shutdownGracefully();
        }
    }
```

## TCP 粘包/拆包

### TCP 粘包/拆包

TCP 是个流协议，流是一串没有界限的数据，它们是连接一片的，没有界限划分。TCP 底层不知道上层业务数据的具体含义，它会根据TCP 缓冲区 的实际情况进行划分，所以事实上，一个完整的数据包可能会被TCP拆分成多个包进行发送，也有可能把多个小数据包，拼装成一个大数据包发送。

​		客户端分别发送D1 和 D2 数据包到服务端，由于服务端读取的字节数是不确定的。粘包问题可能会有如下四种情况：	

1. 服务端分别收到两个独立的数据包，分别是D1 和D2，没有粘包与拆包；
2. 服务端一次收到了两个数据包，D1 和 D2 粘和到了一起，被称为TCP 粘包；
3. 服务端分别收到了两个数据包，第一次读取了完整的D1 数据包与 部分D2 数据包，第二次收到了剩余的D2数据包，这被称为 拆包；
4. 服务端分别收到了两个数据包，第一次读取了部分的D1数据包，第二次读取了剩余的D1 数据包 与完整的D2 数据包，这被称为 拆包；
5. 如果服务端的TCP 接收滑窗非常小，而数据包D1 与 D2 比较大，很可能会发生 D1 与 D2 分别被拆分成多个数据包，发生多次拆包。

### TCP 粘包/拆包发生的原因

1. 应用程序write 的字节数大小大于套接字接口发送缓冲区大小；
2. 进行MSS 大小的TCP 分段
3. 以太网帧的payload大于MTU进行IP 分片

### TCP 粘包问题的解决策略

由于底层的TCP无法理解上层的业务数据，所以在底层是无法保证数据包不被拆分和重组的，这个问题只能通过上层的应用协议栈设计来解决，根据业内主流协议的解决方案，可以归纳如下：

- 消息定长，例如每个报文的大小固定为200字节，如果不够，空位补空格补齐；
- 在包尾增加回车换行符进行分割，如FTP 协议；
- 将消息分为消息头与消息体，消息头中包含消息总长度的字段，通常设计思路为消息头的第一个字段使用int32来表示消息的总长度；
- 更复杂的应用层协议；

### Netty 解决TCP 粘包/拆包

LineBasedFrameDecoder 				换行符粘拆包解析器

DelimiterBasedFrameDecoder		分隔符粘拆包解析器

FixedLengthFrameDecoder			  固定长度粘拆包解析器



LengthFieldBasedFrameDecoder	粘包拆包解析器

LengthFieldPrepender					   包体长度前置器



## 编解码技术

### Java 序列化的缺点

- 无法跨语言
- 序列化后的码流太大
-  序列化性能低

### 主流的编解码框架

Protobuf

https://github.com/protocolbuffers/protobuf

Thrift 

https://github.com/apache/thrift

MessagePack

https://msgpack.org/

FlatBuffers

https://github.com/google/flatbuffers



## 私有协议开发

### Java 应用中 跨节点通信方式

- RMI 远程通信服务
- 通过Java 的Socket  + Java 序列化方式跨节点调用
- 开源RPC 框架,进行远程服务调用,Dubbo,Thrift等
- 利用标准公有协议进行跨节点调用.如RestFull + JSON

私有协议没有标准定义,只要是能够跨进程,跨主机 数据交换的非标准协议,都可以成为私有协议.

### 私有协议主要功能

- 基于Netty 的NIO 通信框架,提供高性能的异步通信能力
- 提供消息编解码框架,实现POJO 的 序列化与反序列化
- 安全认证机制-IP 白名单 SSL 等
- 链路的有效性校验机制
- 链路的断连重连机制

### 通信模型

- Netty 协议客户端发送握手请求消息,携带节点ID 等有效身份认证信息
- Netty 协议栈服务端对握手请求消息进行合法性校验,包括节点ID 有效性校验,节点重复登录校验,IP 合法性校验,校验通过后,返回登录成功的握手消息
- 链路建立成功之后,客户端发送业务消息
- 链路建立成功之后,服务端发送心跳消息
- 链路建立成功之后,客户端发送心跳消息
- 链路建立成功之后,服务端发送业务消息
- 服务端退出,关闭连接,客户端感知对方关闭连接后,关闭客户端连接

注意:Netty 协议通信双方链路建立成功之后,双方可以进行全双工通信,无论客户端还是服务端,都可以主动发送请求消息给对方.

### 消息定义

#### 消息头 Header

| 名称       | 类型               | 长度 | 描述                                                         |
| ---------- | ------------------ | ---- | ------------------------------------------------------------ |
| crcCode    | int                | 32   | Netty 消息校验码主要有三部分内容组成:   crcCode = 0xABEF + 主版本号+次版本号;1)0xABEF 固定值,表明消息时netty 协议消息,两个字节;2)主版本号:1~255 1个字节;3)此版本号 1~255 一个字节; |
| length     | int                | 32   | 整体消息长度,包括消息头与消息体                              |
| sessionId  | long               | 64   | 集群节点内全局唯一,由会话生成器生成                          |
| type       | byte               | 8    | 消息类别:0 业务请求;1业务响应;2业务ONE WAY请求响应消息;3握手请求;4握手响应;5心跳请求;6心跳响应 |
| priority   | byte               | 8    | 优先级 0~255                                                 |
| attachment | Map<String,Object> | 变长 | 扩展消息头                                                   |



#### 消息定义 NettyMessage

| 名称   | 类型   | 长度 | 描述            |
| ------ | ------ | ---- | --------------- |
| header | Header | 变长 | 消息头          |
| body   | Object | 变长 | 消息体,请求内容 |

常用类

EventLoopGroup 

NioEventLoopGroup

ServerBootStrap

BootStrap

ChannelInitializer

ChannelHandlerContext

SocketChannel

NioSocketChannel

ChannelFuture

ByteBuf

Unpooled

DelimiterBasedFrameDecoder

FixedLengthFrameDecoder

## 参考 

https://www.jianshu.com/p/128ddc36e713

Netty权威指南