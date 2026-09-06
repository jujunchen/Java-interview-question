# Netty 面试题集

> 说明：共 36 题，按面试官常问顺序分七组。每题给「考点 + 答题要点」，⭐ 标记为本岗位（设备接入 / 高并发 / 断线重连）高频必背题。
> 建议练法：先看题干自问自答，卡住再看要点；要点里加粗的词是面试官想听到的关键词。

---

## 一、架构与线程模型（基础必考）

**1. Netty 是什么？相比 JDK 原生 NIO 解决了哪些问题？** ⭐
考点：对框架价值的理解，不是背 API。
要点：

- 基于 JDK NIO 的**异步事件驱动**网络框架，统一封装 TCP/UDP/HTTP/WebSocket/MQTT 等协议。
- 解决原生 NIO 四大痛点：① **API 复杂**（Selector/Channel/Buffer 样板代码多）；② **臭名昭著的 epoll 空轮询 bug**（Netty 检测到空轮询超阈值后重建 Selector）；③ **粘包拆包需自己处理**；④ **可靠性与性能需自己打磨**（Netty 内置内存池、零拷贝、成熟线程模型）。
- 生态背书：Dubbo、RocketMQ、gRPC-java、Elasticsearch、Spring WebFlux 底层都用 Netty。

**2. 讲讲 Reactor 三种模式，Netty 默认用哪种？** ⭐
考点：线程模型的核心概念。
要点：

- **单 Reactor 单线程**：一个线程包办 accept + read + decode + 业务 + encode + write。优点无锁简单，缺点无法利用多核、业务耗时阻塞 IO。Redis 6.0 前是此类。
- **单 Reactor 多线程**：一个线程负责 accept，IO 读写交给线程组，业务丢给独立线程池。缺点：单个 accept 线程在高并发新建连接时成瓶颈。
- **主从 Reactor 多线程**：**mainReactor（bossGroup）只负责 accept**，接受后注册到 **subReactor（workerGroup）** 处理 IO 读写，业务再交给业务线程池。**Netty 服务端默认就是这个模型**（`new NioEventLoopGroup(1)` + `new NioEventLoopGroup()`）。
- 加分句：Netty 也支持退化为单线程模型（只传一个 EventLoopGroup），客户端场景常用。

**3. EventLoopGroup、EventLoop、Channel、ChannelPipeline、ChannelHandler 是什么关系？** ⭐
考点：Netty 的对象模型，几乎必问。
要点（一句话串起来）：

- 一个 **EventLoopGroup** 包含多个 **EventLoop**；一个 **EventLoop = 一个线程 + 一个 Selector + 一个任务队列**。
- 一个 Channel **终生绑定**一个 EventLoop（注册时确定，不可变），一个 EventLoop 可服务多个 Channel → 所以 **Channel 的所有 IO 事件天然串行**，handler 内访问成员变量**无需加锁**。
- 每个 Channel 有一个 **ChannelPipeline**，是 **ChannelHandlerContext 组成的双向链表**；pipeline 里挂一串 **ChannelHandler**（入站/出站）。
- 事件传播：**入站事件从 head 往 tail**（按 addLast 顺序），**出站事件从 tail 往 head**（逆序）。这是「解码器要放前面、编码器位置靠后」的原因。

**4. 为什么 Netty 是「串行无锁化」设计？在 handler 里执行耗时操作会怎样，怎么解决？** ⭐
考点：真实生产踩过坑的人才答得好。
要点：

- 设计动机：多线程竞争同一个 Channel 会带来**锁开销 + 上下文切换 + 状态一致性**问题；串行化后 handler 内无需同步，性能与安全性都更好。
- 风险：EventLoop 线程被阻塞 → **该 EventLoop 上所有 Channel 的读写全部停摆**（不是只影响一个连接），表现为一堆设备同时掉线/心跳超时。
- 三种解法：
  1. **耗时业务丢给业务线程池**（自建 ThreadPoolExecutor，配合 `ctx.executor()` 之外的线程执行，注意回写时用 `channel.writeAndFlush` 是线程安全的）；
  2. `pipeline.addLast(businessEventLoopGroup, handler)` —— **给指定 handler 单独指定 EventExecutorGroup**，Netty 自动做线程隔离；
  3. 从架构上隔离：接入层只做协议解析与转发，业务在下游（如 Kafka 消费者）异步处理。**这是 IoT 接入网关的标准做法**。
- 反例必提：**绝对不要在 handler 里做同步 RPC、同步查库、sleep、大文件读写**。

**5. Netty 的「零拷贝」体现在哪几个地方？**
考点：概念区分（OS 零拷贝 vs 用户态零拷贝）。
要点：

- **CompositeByteBuf**：把多个 ByteBuf 逻辑聚合成一个（如 header + body），**不做内存复制**，替代 read 到临时数组再合并。
- **slice() / duplicate() / retainedSlice()**：共享底层内存，只是独立的读写指针视图。
- **FileRegion + FileChannel.transferTo**：文件传输直接走 OS 的 **sendfile**，避免「内核态→用户态→内核态」的往返拷贝。
- **堆外内存 DirectByteBuf**：socket 读写时省去 JDK 内部「堆内→堆外」的一次拷贝（JDK NIO 要求传给系统调用的必须是直接内存）。
- **wrap(byte[])**：`Unpooled.wrappedBuffer` 包装已有数组，不复制。
- 注意区分：零拷贝不等于「零内存分配」，也说清它和 Kafka 那种 sendfile 零拷贝是同一类思想。

**6. Netty 的 FastThreadLocal 比 JDK ThreadLocal 好在哪？**
考点：细节题，能答出加分。
要点：JDK ThreadLocal 底层是**开放地址法哈希表**，冲突时**线性探测**，容量大时性能退化；FastThreadLocal 配合 `FastThreadLocalThread` 使用**数组下标直接寻址**，**O(1)** 访问且无冲突；同时支持 `onRemoval` 回调与 `removeAll` 批量清理，避免 EventLoop 线程复用带来的脏数据/内存泄漏。
坑点：普通 Thread 使用 FastThreadLocal 会退化为 JDK ThreadLocal 实现，**没有性能收益**。

**7. EventLoopGroup 的线程数默认是多少？怎么设置合理？**
考点：调优基础。
要点：

- 默认 = **CPU 核数 × 2**（`NettyRuntime.availableProcessors() * 2`），最小为 16（`io.netty.eventLoopThreads` 可覆盖）。
- bossGroup：**通常 1 个线程足够**（单机单端口 accept）；多端口监听或极高新建连接速率时按端口数设置。
- workerGroup：CPU 密集型按核数，IO 密集型可 2×核数起，**最终以压测为准**。
- 加分句：线程数不是越多越好，过多 EventLoop 会增加上下文切换与内存占用（每个 EventLoop 一个 Selector + 队列）。

---

## 二、ByteBuf 与内存管理（区分高级与中级）

**8. ByteBuf 和 JDK NIO 的 ByteBuffer 有什么区别？** ⭐
要点：

- **读写双指针分离**：`readerIndex` / `writerIndex`，写完直接读，**不需要 flip()**；ByteBuffer 单指针必须 flip，忘记 flip 是经典 bug。
- **动态扩容**：写入超过 capacity 自动扩容（到 maxCapacity）；ByteBuffer 容量固定，超出抛 `BufferOverflowException`。
- **引用计数管理生命周期**：`retain()` / `release()`，池化内存靠它归还。
- 模式丰富：heap / direct、pooled / unpooled 四种组合。
- 提供大量便捷方法：`readBytes`、`readInt`、`markReaderIndex/resetReaderIndex`、`nioBuffer()` 与 NIO 互转。

**9. 讲讲 Netty 的内存池化机制。**
考点：能否讲清 jemalloc 思想。
要点：

- 目标：**避免频繁 malloc/free 与 GC 压力**，减少堆外内存分配的系统调用开销。
- 层次结构（Netty 4.1）：**PoolArena**（分 heap / direct，多个 arena 分散多线程竞争）→ **PoolChunkList** → **PoolChunk**（默认 **16MB**，内部用**伙伴算法 + 完全二叉树**管理页）→ **Page**（默认 **8KB**）→ **SubPage**（小于 8KB 的切分）。
- 规格分类：**Small**（< 一个 Page，Netty 4.1.45+ 已把 tiny 合并进 small）、**Normal**（Page ~ Chunk 之间，走伙伴算法）、**Huge**（超过 Chunk，**不池化**，直接分配释放）。
- 线程绑定：**PoolThreadCache** 为每个线程缓存最近释放的内存块，**同线程复用无需加锁**，这是高性能关键。
- 实用命令：`PooledByteBufAllocator.DEFAULT.metric()` 可看使用率；`-Dio.netty.allocator.type=pooled`（默认已是 pooled）。

**10. Netty 的内存泄漏怎么产生？怎么排查？** ⭐
考点：线上排障能力，高级岗必问。
要点：

- 根因：`ByteBuf` 是 `ReferenceCounted`，**入站消息在 pipeline 末尾由 Netty 自动 release；一旦你在中间 handler 消费了消息却没有 release（或没有传递给下一个 handler），引用计数不归零 → 内存泄漏**。堆外内存泄漏还会导致 **OOM: Direct buffer memory** 或进程 RSS 暴涨被 OOM Kill。
- 常见泄漏场景：
  1. 继承 `ChannelInboundHandlerAdapter` 的 `channelRead` 里**没调用 `ReferenceCountUtil.release(msg)` 也没 `ctx.fireChannelRead(msg)`**；
  2. `ByteBuf buf = ctx.alloc().buffer()` 写完**忘记 release**，或异常分支没释放；
  3. `slice()/duplicate()` 后 retain 与 release 次数不匹配；
  4. 把 ByteBuf 塞进队列/缓存长期持有却忘记释放。
- 排查手段：**`ResourceLeakDetector`** 四级 —— `DISABLED` / **`SIMPLE`（默认，抽样约 1%）** / `ADVANCED`（抽样并打印访问轨迹）/ `PARANOID`（100% 检测，仅测试环境用）。启动参数 `-Dio.netty.leakDetection.level=PARANOID` 在测试环境复现，日志关键字 `LEAK: ByteBuf.release() was not called`。
- 最佳实践：**优先继承 `SimpleChannelInboundHandler<T>`**，它在 `channelRead0` 返回后**自动 release**；写代码遵循「**谁最后持有，谁负责释放**」。

**11. 堆内存 ByteBuf 与堆外内存 ByteBuf 怎么选？**
要点：

- **堆外（direct）**：socket 读写**少一次拷贝**，速度快；缺点：分配/释放成本高（依赖内存池弥补）、不受 GC 直接管理、**泄漏难排查**、访问需先转堆内数组（`hasArray()` 为 false）。
- **堆内（heap）**：分配快、GC 自动回收、`hasArray()` 可直接操作字节数组方便编解码；缺点：网络 IO 时 JDK 会**临时拷到堆外**。
- 结论：**网络 IO 传输用 direct + pooled；纯业务内存计算/编解码处理可用 heap**。Netty 默认 `ByteBufAllocator.DEFAULT` 是 pooled direct。

---

## 三、编解码与协议设计（IoT 岗核心考区）

**12. TCP 为什么会粘包/拆包？Netty 有哪些解决方案？** ⭐⭐
考点：JD 里「设备协议解析」的直接对应题。
要点：

- 根因：**TCP 是面向字节流的协议，没有消息边界**。发送端 Nagle 算法可能合并小包；接收端内核缓冲区一次读取不定长数据；MSS 限制导致大包被切分。
- 五种通用解法：① **定长**（每条消息固定长度，不足补齐）；② **分隔符**（如 `\r\n`）；③ **长度字段 + 消息体**（**工业界最主流**）；④ 应用层自定义协议头带长度；⑤ 换用有边界的协议（如 HTTP chunk、MQTT 变长头）。
- Netty 对应解码器：`FixedLengthFrameDecoder`、`LineBasedFrameDecoder`、`DelimiterBasedFrameDecoder`、**`LengthFieldBasedFrameDecoder`**、`ProtobufVarint32FrameDecoder`。
- 加分句：分隔符方案在**二进制协议**中不可用（payload 里可能出现分隔符字节），所以 IoT 设备私有协议基本都用长度字段方案。

**13. LengthFieldBasedFrameDecoder 的五个参数分别是什么含义？举个例子。** ⭐⭐
考点：能不能真写过协议解码，一问就知。
要点：

- 构造参数：`maxFrameLength`、`lengthFieldOffset`、`lengthFieldLength`、`lengthAdjustment`、`initialBytesToStrip`。
  - **maxFrameLength**：单帧最大长度，超出抛 `TooLongFrameException` —— **必须设置，防止恶意/异常长度字段导致 OOM**。
  - **lengthFieldOffset**：长度字段在帧中的**起始偏移**（前面有几个字节）。
  - **lengthFieldLength**：长度字段**自身占几个字节**（1/2/4/8）。
  - **lengthAdjustment**：修正值 = **长度字段之后到消息体结束还有多少字节** − 长度字段的值。当长度字段只表示 payload 长度、但后面还有校验位时用它补齐。
  - **initialBytesToStrip**：解出完整帧后**跳过前几个字节**（通常用来剥掉协议头）。
- 举例（协议：`magic(2B) | length(4B) | payload | crc(2B)`，length 只表示 payload 长度）：
  `lengthFieldOffset = 2`，`lengthFieldLength = 4`，`lengthAdjustment = 2`（因为长度字段后还有 crc 2 字节），`initialBytesToStrip = 0`（想保留头给业务 handler 解析 cmd）。
- 若 `length` 表示「长度字段之后的全部字节数」，则 `lengthAdjustment = 0`。
- 面试小技巧：**当场画一个字节图**标出偏移，比干说更容易被认可。

**14. ByteToMessageDecoder 和 MessageToByteEncoder 有什么区别？各有什么坑？**
要点：

- `ByteToMessageDecoder`：入站，**ByteBuf → Java 对象**。内部维护 **cumulation（累积缓冲区）**，因为 TCP 流可能半包，需要跨多次 `channelRead` 累积字节；`decode` 中**向 out 添加元素即向下游传递**，添加完可 return 等待更多数据。
- 坑 1：**decode 里不能调用 `ctx.fireChannelRead`**（父类统一做），否则重复传递。
- 坑 2：**父类已管理累积缓冲区，不要在 decode 里 release 入参 ByteBuf**。
- 坑 3：**ByteToMessageDecoder 是有状态的，绝不能标 `@Sharable` 共享实例**。
- `MessageToByteEncoder<T>`：出站，**Java 对象 → ByteBuf**，泛型自动做类型匹配，**父类负责 release 入参消息**；自身通常**无状态，可以 `@Sharable`**。

**15. 哪些 handler 可以加 @Sharable？哪些不行？为什么？**
要点：

- **可以**：无状态 handler —— `ChannelInboundHandlerAdapter` 子类（仅转发）、`MessageToByteEncoder` 子类（无累积状态）、纯业务逻辑 handler。标 `@Sharable` 后**全局单例**，节省对象创建。
- **不可以**：**有状态 handler** —— `ByteToMessageDecoder`（cumulation 缓冲区）、`LengthFieldBasedFrameDecoder`、`IdleStateHandler`（内部维护每连接的读写字节计数与定时任务）、`SslHandler`、`HttpObjectAggregator`（聚合缓冲）。
- 后果：**共享有状态 handler 会导致多个连接的数据串到一起（脏数据/协议错乱）**，这类 bug 极难复现与定位 —— 说出来能体现生产经验。

**16. 让你为一个 IoT 设备接入服务设计 TCP 私有协议，你怎么设计？** ⭐⭐
考点：开放设计题，本岗位最可能出现。
要点（给出一份可讲 3 分钟的协议头）：

```
| magic(2B) | version(1B) | msgType/cmd(1B) | seq(4B) | deviceId长度(1B) | deviceId(N) | payloadLen(4B) | payload(JSON/Protobuf) | crc16(2B) |
```

设计理由逐条讲：

- **magic 魔数**：快速识别非法连接（端口扫描、非本协议客户端），不匹配直接关连接，**保护服务端**。
- **version**：为设备固件升级留兼容余地（老固件长期在线，协议必须能演进）。
- **cmd/msgType**：区分 注册鉴权 / 心跳 / 遥测上报 / 指令下发 / 指令响应 / 事件告警。
- **seq 序列号**：**实现指令下发与响应的异步匹配**，同时支持业务幂等去重（重传场景）。
- **deviceId**：设备唯一标识，接入后用于会话路由（也可放在鉴权阶段绑定到 Channel 的 Attribute 中，避免每帧都带）。
- **payloadLen**：配合 `LengthFieldBasedFrameDecoder` 解决粘包。
- **crc16**：链路完整性校验，弱网/串口透传场景必要。
- 编码选型：**Protobuf**（体积小、跨语言、schema 演进）优于 JSON；若设备端资源受限可考虑 CBOR。
- 安全：**首帧鉴权**（deviceId + secret 签名或一机一密证书），鉴权未通过前不注册业务 handler；生产建议 TLS（`SslHandler`）。
- 加分：提到**协议要能被 Netty 解码器链自然拆分**：`LengthFieldBasedFrameDecoder` → `MagicCheckHandler` → `ProtocolDecoder` → `AuthHandler` → `BusinessHandler`。

**17. Netty 内置支持哪些常用协议编解码器？**
要点：HTTP（`HttpServerCodec`、`HttpObjectAggregator`）、WebSocket（`WebSocketServerProtocolHandler`）、**MQTT**（`MqttDecoder`/`MqttEncoder`，`netty-codec-mqtt` 模块）、Protobuf、JSON（`JsonSerializer`/自定义）、SSL（`SslHandler`）、压缩（`ZlibCodecFactory`、`SnappyFrameDecoder`）、Base64、RTSP/DNS/HAProxy/SMTP 等。
**加分句**：`netty-codec-mqtt` 让 Netty 完全可以自研 MQTT Broker 的接入层，这也是 JetLinks 等国产物联网平台的做法（JD 加分项提到 JetLinks，这里可以主动接上）。

---

## 四、心跳、断线重连与连接管理（JD 明确要求，⭐重点组）

**18. Netty 怎么做心跳检测？IdleStateHandler 的三个时间参数怎么理解？** ⭐⭐
要点：

- `IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime, unit)`：分别检测**读空闲**（多久没收到对端数据）、**写空闲**（多久没向对端写数据）、**读写都空闲**；触发后向 pipeline 发送 `IdleStateEvent`，你在 `userEventTriggered` 里处理。
- 实现方式：内部用 **EventLoop 的 `schedule` 定时任务**，每次读写刷新 `lastReadTime/lastWriteTime`，**每连接独立计时**（所以它是有状态、不可 `@Sharable` 的 handler）。
- 典型策略：
  - **服务端**：设 `readerIdleTime`，读空闲 N 次（一般 2~3 次）未收到心跳 → 判定设备离线，**主动关闭连接并更新设备状态为 offline**。用计数器而非一次就关，避免弱网抖动误杀。
  - **客户端**：设 `writerIdleTime`，写空闲就发一次 PING，保活并探测链路。
- **应用层心跳 vs TCP keepalive**：TCP keepalive 默认 **2 小时**才开始探测（可调 `SO_KEEPALIVE` + 系统参数），**无法及时发现半开连接**，且中间 NAT/防火墙会静默丢弃空闲连接；应用层心跳周期可控（如 30s/60s）、能携带业务状态、能跨中间件感知，**生产必须用应用层心跳**。
- 半开连接（half-open）：一端已宕机/断网但未发 FIN，另一端仍认为连接有效 —— **这是设备「假在线」的根因**，必须靠读空闲超时清理。

**19. 客户端（设备端 SDK）断线重连怎么实现？** ⭐⭐
要点（JD 原文「断线重连」，务必能完整答）：

- 触发点：`channelInactive`（连接断开）或 `exceptionCaught` 中主动关连接后。

- 核心代码思路：
  
  ```java
  ctx.channel().eventLoop().schedule(() -> bootstrap.connect(host, port), delay, MILLISECONDS);
  ```
  
  —— **复用原 EventLoop 调度**，不需要额外线程池。

- **指数退避 + 随机抖动**：delay = min(base × 2^n, maxDelay) + random(0, jitter)。理由：设备批量掉线（如机房断网恢复、broker 重启）时，**若所有设备同时以固定间隔重连，会造成连接风暴打垮服务端**（惊群）。这是资深候选人和初级候选人的分水岭。

- 重连后必须做的四件事：① **重新鉴权注册**（新 Channel 是全新会话）；② **恢复订阅/topic**；③ **重传未确认数据**（本地缓存 + seq 去重）；④ **重置退避计数**。

- 防重复连接：用 `AtomicBoolean` 或连接状态机（CONNECTING/CONNECTED/RECONNECTING）保证同一时刻只有一个重连任务；**Bootstrap 可复用，但要确保 handler 是无状态或每次新建 pipeline**。

- 加分句：服务端侧对「重连风暴」的保护手段 —— **接入层限流（令牌桶）、连接数上限、握手队列削峰、返回 503 让设备退避**。

**20. 服务端如何管理海量设备连接（会话管理）？** ⭐⭐
要点：

- **会话映射**：`ConcurrentHashMap<String deviceId, Channel>`（或 `AttributeKey<Session>` 挂在 Channel 上存业务数据，`channel.attr(KEY).set(session)`）。鉴权成功后写入，`channelInactive` 中移除。
- **反向索引**：`Channel → deviceId`（用 `AttributeKey`），保证断开时能定位设备。
- **分组广播**：`ChannelGroup` + `DefaultChannelGroup(GlobalEventExecutor.INSTANCE)`，`group.writeAndFlush(msg)` **自动遍历组内 Channel，且 Channel 关闭时自动从组移除**，天然防内存泄漏 —— 用于「给某个区域所有设备下发指令」。
- **跨节点下发**：单机 map 只在**集群内单节点有效**。多接入节点时，指令下发需要**路由层**：用 Redis 记录 `deviceId → nodeId`，下发时先查路由再投递到目标节点（RPC 或 MQ 定向 topic）。**这是 IoT 接入网关集群化的核心难点，主动讲出来很加分。**
- **资源保护**：连接数上限（超出拒绝并关闭）、单连接发送速率限制、**空闲连接主动回收**、握手超时（`ConnectTimeoutHandler` / `ReadTimeoutHandler`）。
- **优雅下线**：节点重启前先摘掉注册中心流量 → 停止 accept → 通知设备重连到其他节点 → 等待在途消息处理完 → `shutdownGracefully`。

**21. Netty 中如何做流量整形与背压（写入过慢）？** ⭐
要点：

- **问题现象**：`channel.writeAndFlush` 太快而**对端消费慢（TCP 滑动窗口满）**，数据堆积在 Netty 的 **ChannelOutboundBuffer**，**堆外/堆内内存暴涨直至 OOM**。
- **检测**：`channel.isWritable()` —— 当待写字节超过**高水位线**返回 false；`channelUnwritable` / `channelWritabilityChanged` 回调感知状态变化。
- **配置**：`ChannelOption.WRITE_BUFFER_WATER_MARK`，`new WriteBufferWaterMark(low, high)`，**默认 32KB / 64KB**。低水位恢复可写，高水位停止可写。
- **应对**：① 业务侧**判断 isWritable，不可写时暂停生产或丢帧**（遥测数据可降级丢弃，指令数据必须缓存重发）；② `ChannelOutboundBuffer` 满时**限流上游**（如 Kafka 消费者暂停 poll）；③ 用 `FlushConsolidationHandler` **合并多次 flush**，减少系统调用（高频小报文场景收益明显）。
- **流量整形 handler**：`GlobalTrafficShapingHandler`（全局）、`ChannelTrafficShapingHandler`（单连接）、`GlobalChannelTrafficShapingHandler`，可配置读写速率上限，**防止个别设备打满带宽**。
- `writeAndFlush` vs `write` + `flush`：前者每条消息一次系统调用；后者可**批量 write 后统一 flush**，高吞吐场景更高效。

**22. Netty 服务端/客户端要做优雅关闭，怎么做？**
要点：`group.shutdownGracefully(quietPeriod, timeout, unit)` —— **默认静默期 2s、超时 15s**。静默期内**拒绝新任务**，若期间有新任务提交则重置静默期；超时后强制关闭。返回 `Future`，可 `syncUninterruptibly()` 或注册回调确保关闭完成后再退出 JVM（配合 Spring 的 `@PreDestroy` / `SmartLifecycle`）。
注意：关闭前应**先停止 accept 新连接、通知在途请求、flush 未发送数据**，避免设备侧看到异常断连。

---

## 五、参数调优与高并发（高级岗区分度）

**23. Netty 常用的 ChannelOption 参数有哪些？各自作用？** ⭐
要点（能说出 6 个以上算合格）：

- **SO_BACKLOG**：TCP 全连接队列大小（accept queue），**Netty 默认 1024**；受内核 `net.core.somaxconn` 上限约束，**两者取小**。高并发新建连接时调大可减少连接被丢弃（`SYN flood` 场景另说）。
- **SO_REUSEADDR**：地址复用，服务端重启时避免 `Address already in use`（TIME_WAIT 占用）；Linux 下配合 `SO_REUSEPORT` 可多进程/多线程绑同端口做负载均衡。
- **TCP_NODELAY**：**关闭 Nagle 算法**，禁用「攒小包再发」，**降低延迟**。IoT 指令下发、心跳这类小包场景**必须开启**（Linux 默认开启 Nagle）。
- **SO_KEEPALIVE**：开启 TCP 层保活探测（默认 2 小时起探），**只能作为应用层心跳的兜底，不能替代**。
- **SO_RCVBUF / SO_SNDBUF**：内核收发缓冲区大小，一般交给系统自动调优（Linux 有 buffer auto-tuning，**显式设置反而会禁用自动调优**），大流量场景才手动调。
- **SO_LINGER**：控制 close 行为，设为 0 会发 RST 直接丢弃未发送数据（慎用）。
- **ALLOCATOR**：指定 ByteBuf 分配器（pooled/unpooled）。
- **RCVBUF_ALLOCATOR**：接收缓冲区分配器，`AdaptiveRecvByteBufAllocator` 自适应调整每次读取大小（默认）。
- **WRITE_BUFFER_WATER_MARK**：写缓冲水位（见第 21 题）。
- **CONNECT_TIMEOUT_MILLIS**：客户端连接超时（默认 30s，生产建议改小到 3~5s）。
- 补充：`Bootstrap.option()` 作用于 **ServerSocketChannel（accept 相关）**，`childOption()` 作用于**每个新建的 SocketChannel** —— **这个区别常被拿来考细节**。

**24. EpollEventLoopGroup 比 NioEventLoopGroup 好在哪？**
要点：

- **仅 Linux 可用**（`netty-transport-native-epoll`，需带 classifier 的 native 依赖，如 `linux-x86_64`）。
- 优势：① 支持 **边缘触发（ET）** 模式，减少 epoll 唤醒次数；② 支持 **SO_REUSEPORT**，多进程绑同端口，内核层负载均衡，**缓解单 accept 线程瓶颈**；③ 支持更多 Linux 专属参数（TCP_CORK、TCP_FASTOPEN、`EpollChannelOption`）；④ **性能比 NIO 高（官方与社区压测通常 10%+ 提升）**，且**规避了 JDK NIO 的 epoll 空轮询 bug**（Netty NIO 版本靠重建 Selector 绕过）。
- 结论：生产 Linux 环境**优先 Epoll**，本地开发用 Nio 做兼容（可通过反射检测 `Epoll.isAvailable()` 自动切换）。

**25. 百万级长连接场景，Netty 服务要注意什么？** ⭐
考点：架构视野题，IoT 平台必问。
要点：

- **内存是第一瓶颈**：每连接的 Channel + pipeline + 读写缓冲区 + 会话对象。假设每连接 20KB，100 万连接就是 **20GB** → 必须**精简 pipeline（handler 数量最小化）、共享 `@Sharable` handler、会话对象瘦身、用堆外内存 + 内存池**。
- **文件句柄**：`ulimit -n` / `fs.file-max` 必须放大（百万级），否则 `Too many open files`。
- **内核参数**：`net.core.somaxconn`、`net.ipv4.tcp_max_syn_backlog`、`net.ipv4.ip_local_port_range`（客户端侧）、`net.ipv4.tcp_tw_reuse`、`net.core.netdev_max_backlog`。
- **GC 压力**：海量长生命周期对象 → **对象晋升老年代 → Full GC 停顿影响心跳** → 用 G1/ZGC、控制堆大小、**优先堆外内存**、避免每连接创建大对象。
- **单机上限现实值**：单机 10 万~50 万连接较务实，**超过就水平扩展**（接入层无状态化 + 四层负载均衡 LVS/DNS 轮询 + 会话路由表）。
- **连接风暴防护**：批量掉线重连时的限流、握手排队、退避引导（见第 19 题）。
- **监控**：连接数、读写空闲数、ChannelOutboundBuffer 堆积、EventLoop 任务队列长度（`executor().pendingTasks()`）、GC 停顿、句柄数 —— **有监控才敢说支撑百万连接**。

**26. Netty 的定时任务是怎么实现的？HashedWheelTimer（时间轮）了解吗？**
要点：

- **EventLoop 内置调度**：`eventLoop.schedule()` / `scheduleAtFixedRate()`，底层是 **优先级队列（按执行时间排序）**，精度较高，**任务在 EventLoop 线程执行 → 不能放耗时逻辑**。
- **HashedWheelTimer（时间轮）**：环形数组（默认 **512 个槽 tickPerWheel**），每槽一个任务链表，指针每 **tickDuration（默认 100ms）** 前进一格，执行到期任务。
  - 优点：**添加/取消任务 O(1)**，海量定时任务（如**每连接一个心跳超时定时器**）下远优于优先级队列的 O(log n)；单线程运行，内存与 CPU 开销可控。
  - 缺点：**精度受 tickDuration 限制（最坏误差一个 tick）**；任务在**时间轮线程串行执行，耗时任务会拖累后续任务** → 必须丢业务线程池。
  - 使用：`new HashedWheelTimer(threadFactory, 100, MILLISECONDS, 512)`，**全局共享一个实例，不要每次 new**（每个实例一个线程）。
- 选型结论：**少量高精度任务用 EventLoop schedule；海量连接级超时检测用时间轮**（Netty 的 `IdleStateHandler` 早期版本即用时间轮思路，现版本用 EventLoop schedule）。

---

## 六、场景设计题（IoT 岗压轴，占分最重）

**27. 设计一个支撑 10 万台设备的接入网关，讲讲整体架构。** ⭐⭐
考点：JD 第 2、3 条的综合题，几乎一定会考。
要点（分层讲，边讲边画）：

1. **接入层（Netty）**：`主从 Reactor` + `LengthFieldBasedFrameDecoder`/`MqttDecoder` 解协议 → **鉴权 Handler（一机一密/证书，未过鉴权 5s 内关闭）** → 会话注册（deviceId↔Channel + Redis 路由表 `deviceId → gatewayNodeId`）→ **接入层不做业务，只做协议转换与转发**。
2. **上行链路**：解码后的遥测/事件消息 → **Kafka**（按 deviceId hash 分区，保证单设备有序）→ 下游清洗消费者 → **时序库（TDengine/InfluxDB）** + **规则引擎/告警计算**。理由：**削峰、解耦、可重放**，设备数据洪峰（如整点集中上报）不会打垮存储。
3. **下行链路**：业务系统下发指令 → 指令服务查 Redis 路由 → 定向投递到目标网关节点（RPC 或 MQ 定向队列）→ 网关从会话 map 拿 Channel → `writeAndFlush`，并**登记 seq + Promise 等待响应，超时告警重试**。
4. **可靠性**：应用层心跳（`IdleStateHandler`）+ 掉线判定 → 更新**设备影子**在线状态；断线重连用指数退避（见 19 题）；写背压用 `isWritable` + 水位（见 21 题）。
5. **水平扩展**：接入节点**无状态**（会话状态在 Redis）、LVS/DNS 分流、节点上下线时优雅摘流；**监控**连接数、消息 lag、EventLoop 队列长度。
6. **加分**：主动说明「如果规模没那么大，直接基于 **EMQX** 做接入 + 规则引擎桥接 Kafka，比自研 Netty broker 更划算；自研 Netty 接入层适合**私有二进制协议 + 深度定制**的场景」—— **体现技术选型判断力，比一味炫技更能打动面试官**。

**28. 设备指令下发后如何拿到同步响应？超时怎么处理？** ⭐
要点：

- **难点**：Netty 是异步的，`writeAndFlush` 只保证写成功（进内核缓冲），**不代表设备已执行**。
- 方案：**请求-响应关联表** `ConcurrentHashMap<String seq, CompletableFuture<Response>>`（或 Netty `DefaultPromise`）。
  1. 下发前生成唯一 seq（建议 `deviceId + 自增 + 时间戳`），创建 Future 放入 map，设置超时；
  2. `channel.writeAndFlush(cmd)` 并挂 `ChannelFutureListener`：写失败立即 complete 异常并清理 map；
  3. 设备响应帧解码后按 seq 从 map 取出 Future 并 `complete(response)`；
  4. **超时兜底**：用 `future.orTimeout(n, SECONDS)`（或 `HashedWheelTimer` 注册延时任务）→ 超时后 `completeExceptionally` **并从 map 移除**（**不移除就是内存泄漏**）；
  5. 业务侧根据失败原因决定重试（重试需带**相同 seq 保证幂等**，设备侧去重）。
- 加分：区分「**写成功**」与「**设备执行成功**」两个 ack；指令要设计**幂等键**，防止网络重传导致设备重复动作（如重复开锁）。

**29. 如何感知设备上下线并保证状态准确？** ⭐
要点：

- **上线**：TCP 连接建立 ≠ 设备可用，**必须以鉴权成功为上线标志**（避免半连接/扫描器误报）。
- **下线**：三条路径 —— ① `channelInactive`（正常 FIN/RST，最准）；② **读空闲超时**（假死/断网/掉电，`IdleStateHandler` 兜底）；③ 主动踢下线（鉴权失效、重复登录挤下线）。
- **抖动抑制**：状态变更**延迟确认**（连续 2~3 个心跳周期无响应才置 offline），或结合**重连窗口**（断开后 N 秒内重连视为未离线），避免弱网导致状态频繁跳变刷爆告警与业务。
- **重复登录处理**：同 deviceId 新连接进来时，**旧 Channel 主动关闭并清理会话**，防止一个设备两条连接导致指令下发到死连接。
- 落地：**状态写入 Redis + 变更事件发 Kafka**，业务侧（告警、看板、设备影子）订阅消费；**避免在接入层直接写库**（阻塞 IO 线程）。

**30. 如何在 Netty 中集成 Spring Boot？生命周期怎么管理？**
要点：

- 服务端启动：实现 `SmartLifecycle`（或用 `@PostConstruct` + `@PreDestroy`），在 `start()` 里绑定端口、`stop()` 里 `shutdownGracefully`；**端口绑定建议同步等待**（`sync()`）以便启动失败时快速暴露。
- Handler 注入：**pipeline 里需要注入 Spring Bean（如业务 Service）时，handler 不能用无参 new**；做法：把 `ApplicationContext` 或目标 Bean 通过构造函数传入 handler，**或用 `ChannelInitializer` 持有 Spring Bean 引用**（注意 `@Sharable` 与有状态 handler 的区别，见第 15 题）。
- 常见坑：Netty 线程与 Spring 事务/ThreadLocal 上下文（如 `RequestContextHolder`、MDC 日志链路）**不互通** → 需要在提交业务任务时**手动传递上下文**（日志 traceId 尤其容易丢）。

---

## 七、快问快答（查漏补缺）

**31. pipeline 里 handler 的执行顺序规则？**
入站：head → tail（addLast 的先后顺序）；出站：tail → head（逆序）。所以**解码器要放在业务 handler 之前，编码器要放在业务 handler 之后（即更靠 tail）**。

**32. exceptionCaught 的作用与传播规则？**
异常沿 pipeline **向后（入站方向）传播**，直到有 handler 覆写处理；若到 tail 仍无人处理，Netty 打印 `An exceptionCaught() event was fired, and it reached at the tail of the pipeline. It usually means the last handler in the pipeline did not handle the exception.` 的 WARN 日志。**生产必须在业务 handler 或 pipeline 末尾统一捕获并记录、必要时关闭连接**。
注意：`exceptionCaught` 只捕获 pipeline 内异常；**EventLoop 线程外的异步异常捕获不到**。

**33. ctx.writeAndFlush 与 channel.writeAndFlush 的区别？**
`ctx.writeAndFlush` **从当前 handler 位置开始向前（出站方向）传播**，只经过它之后的出站 handler；`channel.writeAndFlush` **从 pipeline 尾部（tail）开始**，经过所有出站 handler。**编码器位置不同时结果会不一样，是常见 bug 来源**。

**34. Netty 怎么解决 JDK NIO 的 epoll 空轮询 bug？**
`NioEventLoop.run()` 中统计**空轮询次数**，超过阈值（默认 **512**，`SELECTOR_AUTO_REBUILD_THRESHOLD`）就**重建一个新 Selector，把原有 Channel 迁移过去**，替换旧 Selector。属于「绕过」而非根治。

**35. 什么是 ChannelHandlerContext？它和 Channel、pipeline 的关系？**
每个 handler 加入 pipeline 时被包装成一个 **ChannelHandlerContext**，它是 **handler 在 pipeline 中的「位置句柄」**，持有前驱/后继引用，提供 `fireXxx`（入站传播）、`write/flush`（出站传播）、`alloc()`、`executor()`、`channel()`、`attr()` 等方法。**ctx 是链表节点，channel 是整条链的宿主**。

**36. Netty 的性能瓶颈通常出现在哪里？你实际调优过什么？**
参考答题框架（**务必替换成你自己的真实经历，不要背模板**）：

- 定位手段：`top -H` + jstack 看 EventLoop 线程 CPU；Arthas `trace`/`watch` 定位慢 handler；`ChannelOutboundBuffer` 堆积量；GC 日志；Netty 自带的 `PooledByteBufAllocatorMetric`。
- 常见瓶颈：① **handler 里做了阻塞操作**（最常见）；② **频繁 flush 导致系统调用过多**（用 `FlushConsolidationHandler` 或批量 write）；③ **ByteBuf 泄漏导致 direct memory OOM**；④ **共享有状态 handler 造成数据错乱**；⑤ **水位设置不当导致 OOM 或吞吐下降**；⑥ **序列化开销大**（JSON → Protobuf）。
- **建议结合你简历中的真实素材**：例如「开放平台重构性能优化」「Arthas 集成监控使排查时间缩短 80%」「解决线上抢券卡死」——把这些经历用 Netty/高并发的语言重新讲一遍，比背八股更有说服力。

---

## 附：面试前 3 小时冲刺清单

1. 手画 **Netty 架构图**：EventLoopGroup → EventLoop → Selector → Channel → Pipeline → Handler，边画边说 3 分钟。
2. 手画 **协议字节图**，讲清 `LengthFieldBasedFrameDecoder` 五个参数取值。
3. 手写（或口述）**心跳 + 断线重连 + 指数退避**核心代码骨架。
4. 背熟三个数字：EventLoop 默认线程数 **2×核数**、内存池 Chunk **16MB**/Page **8KB**、写缓冲水位默认 **32KB/64KB**、`shutdownGracefully` 默认 **2s/15s**、空轮询阈值 **512**。
5. 准备一句诚实的兜底话术：「Netty 我在 XX 场景做过 YY，ZZ 没有生产深度使用，但原理和适用边界我清楚，上手很快。」**切忌编造项目细节，被追问三层就露馅。**
