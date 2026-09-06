# MQTT 面试题集

> 共 42 题，分八组。每题给「考点 + 答题要点」，⭐⭐ = 本岗位高频必背，⭐ = 建议掌握。
> 建议练法：先自问自答，卡住再看要点；加粗词是面试官想听到的关键词。
> 版本相关的具体配置项和数值会随 EMQX / 协议版本变化，正式引用前建议核对目标版本文档。

---

## 一、协议基础与报文结构

**1. MQTT 是什么？为什么适合物联网？** ⭐⭐
考点：一句话讲清协议定位。
要点：

- **MQ = MQ Telemetry Transport**（不是 Message Queue，这是常见误区），**轻量级的发布/订阅消息传输协议**，1999 年由 IBM 的 Andy Stanford-Clark 与 Arlen Nipper 设计，最初用于输油管道 SCADA 的卫星链路监控；现为 **OASIS 标准**。
- 适合 IoT 的四个原因：① **报文头最小仅 2 字节**（对比 HTTP 动辄几百字节头），**窄带宽、按流量计费**场景优势明显；② **发布/订阅解耦**，设备与消费者互不感知，天然适合一对多下发与海量上报；③ **三级 QoS** 适配不稳定网络（蜂窝、卫星、弱 WiFi）；④ **会话保持 + 遗嘱消息 + 保留消息**，为"设备可能随时掉线"这一前提做了内建设计。
- 一句话对比：**HTTP 是请求-响应，MQTT 是发布-订阅 + 长连接**；IoT 设备数量大、上行频繁、下行随时可能来，长连接推送模型比轮询合适。

**2. MQTT 控制报文的结构是怎样的？有哪些报文类型？** ⭐
考点：能不能讲出报文层面的细节。
要点：

- **三段式结构**：`固定报头（Fixed Header，必有）` + `可变报头（Variable Header，部分报文有）` + `有效载荷（Payload，部分报文有）`。
- **固定报头**：第 1 字节 = **报文类型（高 4 位）+ 标志位（低 4 位）**；后面是 **Remaining Length（剩余长度）**，用**可变字节整数**编码（每字节低 7 位有效、最高位为续位），最多 4 字节。
- **15 种控制报文**：`CONNECT`、`CONNACK`、`PUBLISH`、`PUBACK`、`PUBREC`、`PUBREL`、`PUBCOMP`、`SUBSCRIBE`、`SUBACK`、`UNSUBSCRIBE`、`UNSUBACK`、`PINGREQ`、`PINGRESP`、`DISCONNECT`、**`AUTH`（仅 MQTT 5.0，用于增强认证）**。
- 加分细节：**剩余长度最大可表示 268,435,455 字节（约 256MB）**，即单个 MQTT 报文载荷理论上限约 256MB——但**生产绝不能用这么大**，会打爆 broker 内存（见第 24 题）。
- 标志位要点：PUBLISH 的低 4 位含 **DUP（重发标志）、QoS（2 位）、RETAIN（保留标志）**，这是唯一"标志位有实际含义"的报文；其余报文标志位必须为固定值。

**3. MQTT 常用端口有哪些？**
要点：**1883**（TCP 明文）、**8883**（TLS/SSL）、**8083**（WebSocket）、**8084**（WebSocket over TLS）；MQTT over QUIC 走 UDP（EMQX 支持）。**生产必须走 8883**，1883 明文传输等于裸奔。

**4. MQTT 3.1 / 3.1.1 / 5.0 的区别？为什么现在推荐 5.0？** ⭐⭐
考点：高频送分题，也最容易答得含糊。
要点（3.1.1 → 5.0 的关键增量）：
| 能力 | 3.1.1 | 5.0 |
|---|---|---|
| 错误码 | CONNACK 只有几个码，**订阅失败只返回 0x80 一个笼统值** | **完整 Reason Code**（SUBACK/PUBACK/DISCONNECT 都有细分原因） |
| 会话过期 | `cleanSession` 布尔值，**要么全清要么永不过期** | **Session Expiry Interval**（秒级，可精确控制） |
| 元数据 | 无 | **User Properties**（自定义键值对，可带 traceId、租户、协议版本） |
| 请求响应 | 需应用层自己设计 | **Response Topic + Correlation Data** 协议原生支持 |
| 消息过期 | 无，离线消息无限期堆积 | **Message Expiry Interval**，过期消息不投递 |
| 流控 | 无 | **Receive Maximum**（在途未确认消息上限）、**Topic Alias**（用数字别名替代长 topic 省流量） |
| 服务端能力 | 无 | **Server Keep Alive**（服务端指定心跳）、**Assigned Client Identifier**、**Server Reference/重定向** |
| 认证 | 仅用户名密码 | **增强认证 AUTH 报文（SCRAM、Kerberos、DIGEST-MD5）** |
| 遗嘱 | 立即发布 | **Will Delay Interval**（延迟发布，抑制抖动误报） |
| 共享订阅 | 无标准（各家私有实现） | **标准化 `$share/{ShareName}/{topic}`** |
| 载荷格式 | 无 | **Payload Format Indicator + Content Type**（标明是 UTF-8 文本还是二进制、是 JSON 还是 Protobuf） |

- **为什么推荐 5.0**：可观测性（细粒度原因码 + 用户属性传 traceId）、可控性（消息过期、在途流控、会话过期）、原生支持请求响应与共享订阅。**IoT 平台做指令下发-响应关联、链路追踪，5.0 省掉大量应用层设计。**
- 现实约束：**大量存量设备固件只支持 3.1.1**，且 MQTT-SN 场景另有规范 → **平台通常要同时兼容 3.1/3.1.1/5.0**（EMQX 全支持）。

**5. MQTT 和 HTTP、CoAP、AMQP 有什么区别？各自适用场景？** ⭐
考点：选型视野题。
要点：

- **MQTT vs HTTP**：MQTT 长连接、服务端可主动推送、报文小；HTTP 短连接（或需 WebSocket/SSE）、客户端主动请求、头部开销大。设备上报频繁 + 需服务端下发 → MQTT；偶发请求、浏览器直连、需 CDN/缓存 → HTTP。
- **MQTT vs CoAP**：CoAP 基于 **UDP**、请求-响应模型、报文更小、支持组播，适合**极度受限设备 + 低功耗网络（如 NB-IoT、6LoWPAN）**；MQTT 基于 TCP，需要 broker 中转，覆盖更广但设备侧协议栈更重一点。EMQX 通过 CoAP 网关把 CoAP 消息桥接成 MQTT。
- **MQTT vs AMQP（RabbitMQ）**：AMQP 是**企业级消息队列协议**，功能全（多种交换器、路由、事务、优先级），报文重，适合**服务间可靠异步通信**；MQTT 是**设备到云的遥测传输协议**，轻量，适合**海量设备连接**。一句话：**MQTT 管"设备接进来"，AMQP/Kafka 管"业务系统之间流转"**。
- **MQTT-SN**：MQTT 的非 TCP 变体，跑在 ZigBee/蓝牙/UDP 上，不需要 broker 支持完整 TCP 栈，topic 用数字 ID 代替字符串，适合传感器网络。

---

## 二、连接、会话与保活（⭐重点组）

**6. CONNECT 报文里有哪些关键字段？CONNACK 返回什么？** ⭐⭐
要点：

- CONNECT 可变报头：**协议名（"MQTT"）、协议级别（4=3.1.1，5=5.0）、Connect Flags、Keep Alive**；Connect Flags 里含 **Clean Session、Will Flag、Will QoS、Will Retain、Password Flag、Username Flag**。
- CONNECT 载荷：**Client Identifier（必填）**、Will Topic、Will Payload、Username、Password。
- **CONNACK**：`Session Present 标志` + `Reason Code`。
  - **Session Present = 1**：服务端恢复了之前的会话（订阅关系和离线消息还在），**客户端不必重新订阅**；= 0 则是全新会话，需要重新 SUBSCRIBE。**很多客户端 bug 出在没判断这个标志。**
  - 3.1.1 常见拒绝码：`0x01 协议版本不支持`、`0x02 ClientID 被拒绝（非法字符/过长）`、`0x03 服务端不可用`、**`0x04 用户名或密码错误`**、**`0x05 未授权`**。
  - 5.0 扩展了大量码：`0x86 载荷格式非法`、`0x8C QoS 不支持`、`0x93 保留主题不支持`、`0x97 配额超限`、`0x9B 报文过大` 等。**细粒度错误码是 5.0 排障体验好得多的关键。**

**7. ClientId 有什么约束？两台设备用同一个 ClientId 会怎样？** ⭐⭐
考点：真实事故高发区。
要点：

- **必须全局唯一**；3.1.1 建议 1~23 字符（更长 broker 可能拒绝，EMQX 可放宽到 65535）；**不能包含 topic 通配符 `+`/`#` 与分隔符**，否则可能被用来绕过 ACL 检查（Mosquitto 文档明确做了这个校验）。
- **相同 ClientId 上线 → 会话接管（Session Takeover）**：broker **主动断开旧连接**，新连接接管该 ClientId 的会话。这是协议规定行为，不是 bug。
- **危害**：① 攻击者偷到凭证后**反复挤设备下线**，构成 DoS；② 产线调试时两台设备配了相同 ClientId，表现为**互相踢下线、日志里疯狂重连**——这是现场最常见的诡异问题之一。
- **实战建议**：ClientId 用设备唯一硬件标识（IMEI/MAC/SN）或其派生值；**监控"会话接管"事件频率，突增即密钥泄露或产线配错的信号**；EMQX 可通过 HTTP API / Webhook 感知 `client.disconnected` 及断开原因码，区分正常断开与被挤下线。
- 加分：MQTT 5.0 支持 **Assigned Client Identifier**——客户端可发空 ClientId，由服务端分配并通过 CONNACK 返回，适合无固定标识的临时客户端。

**8. Clean Session（cleanStart）到底控制什么？设为 true 和 false 有什么区别？** ⭐⭐
考点：离线消息能力的根基，必问。
要点：

- **控制的是"服务端是否为这个 ClientId 保存会话状态"**，会话状态包含：① **订阅关系**；② **未确认的 QoS1/QoS2 消息**；③ **该客户端离线期间到达的 QoS1/QoS2 消息**。
- **cleanSession = true**：**每次连接都是全新会话**，断开即清空。重连后**必须重新 SUBSCRIBE**，且**离线期间的消息全部丢失**。**QoS0 消息任何时候都不保存。**
- **cleanSession = false**：服务端持久化会话。设备重连后 **CONNACK 的 Session Present = 1**，订阅关系自动恢复，**离线期间的 QoS1/QoS2 消息会被补发**——这就是"设备离线数据不丢"的协议级实现。
- **MQTT 5.0 的改进**：拆成 **Clean Start（布尔，只影响本次连接是否复用已有会话）+ Session Expiry Interval（秒，控制会话在服务端存活多久）**。这样能做到"复用会话，但离线超过 24 小时就自动清理"，比 3.1.1 的非黑即白灵活得多。**Session Expiry Interval = 0 且 Clean Start = 1 等价于 3.1.1 的 cleanSession=true。**
- **选型结论**：**遥测数据可容忍丢失 → cleanSession=true，省 broker 内存**；**告警、指令响应、计费类数据不能丢 → cleanSession=false + QoS1**，但要评估离线消息堆积量（见第 24 题）。
- **坑点（务必讲）**：`cleanSession=false` **不是免费的**——broker 要为每个离线设备保留订阅表和消息队列，**百万设备长期离线会吃掉大量内存与磁盘**。所以 EMQX 等平台都有**离线消息数量上限**配置，超限后按策略丢弃最旧消息。

**9. Keep Alive 机制怎么工作？为什么是 1.5 倍？** ⭐⭐
考点：JD 里"设备上下线管理"的协议基础。
要点：

- CONNECT 中声明 `Keep Alive`（秒，**0 表示关闭心跳**）。客户端必须在 **1.5 × Keep Alive** 时间内向服务端发送**任意控制报文**（不只是 PINGREQ，PUBLISH/SUBSCRIBE 都算）。
- 空闲时客户端发 **PINGREQ**，服务端回 **PINGRESP**（**这两个报文载荷均为空**，只有 2 字节固定报头，非常省流量）。
- **服务端在 1.5 倍周期内未收到任何报文 → 判定客户端断开**，此时**发布遗嘱消息（LWT）**并清理会话。
- **为什么留 1.5 倍余量**：① 网络存在传输延迟与抖动；② 客户端可能因省电休眠略微延后发送；③ 容忍一次报文丢失。**如果严格等于 Keep Alive 就判死，弱网下会大量误杀**——这正是你在真实项目里"设备假离线"告警满天飞的根因。
- **实践数值**：设备侧常配 **60s**；服务端判定即 90s 无报文视为掉线。低功耗设备会拉到 300s 甚至更长（配合 MQTT 5.0 的 **Server Keep Alive**，由服务端统一下发心跳周期，避免每台设备各自配置不一致）。
- **与 TCP keepalive 的区别（必答）**：TCP keepalive 默认 **2 小时**才开始探测，无法及时发现**半开连接**；且**中间 NAT / 防火墙会静默丢弃空闲连接的映射表项**，导致连接"看起来还在，实际已死"。所以**MQTT 应用层心跳是必需的，TCP keepalive 只能做兜底**。
- **半开连接（half-open）**：设备断电/断网未发 FIN，broker 仍认为连接有效 → 设备状态显示"在线"但下发指令石沉大海。**这是 IoT 平台"假在线"问题的协议根因**，靠 Keep Alive 超时清理。

**10. 遗嘱消息（LWT）是什么？怎么用它做设备上下线？** ⭐⭐
考点：这题答好，"设备上下线管理"整题就通了。
要点：

- **CONNECT 时预先注册一条消息**（Will Topic + Will Payload + Will QoS + Will Retain），**当客户端异常断开（Keep Alive 超时、网络中断、broker 主动断开）时由 broker 代为发布**；**客户端
- 主动发 DISCONNECT 正常断开时不会发布遗嘱**（MQTT 5.0 中若 DISCONNECT 带非 0x00 原因码，仍会发布）。
- **标准上下线实现**（业界通用做法）：
  
  ```
  CONNECT 时注册:  Will Topic = device/{deviceId}/status
                   Will Payload = {"online":false}
                   Will QoS = 1, Will Retain = true
  连接成功后立即:  PUBLISH device/{deviceId}/status
                   Payload = {"online":true}, QoS1, Retain = true
  ```
  - **Retain = true 是关键**：任何新订阅者一连上就能立刻拿到该设备**最后一次状态**，不用等设备下次上报。业务侧（看板、告警）订阅 `device/+/status` 即可感知全量设备在线状态。
  - 异常掉线 → broker 自动发 offline 遗嘱；主动优雅下线 → 客户端先 PUBLISH offline 再 DISCONNECT。
- **MQTT 5.0 的 Will Delay Interval**：遗嘱**延迟 N 秒发布**。用途：**抑制网络抖动导致的假离线**——设备闪断后在 N 秒内重连，则遗嘱不会被投递，避免状态频繁跳变刷爆告警。这是 5.0 对 IoT 最实用的改进之一，**主动讲出来能明显拉开档次**。
- **落地注意**：遗嘱消息只解决"通知"，**平台侧还要做状态收敛**——延迟确认（连续 2~3 个心跳周期无响应才置 offline）、去重（同一状态不重复写库）、变更事件发 Kafka 给下游订阅，**避免在 broker 回调里直接写数据库**。

**11. 保留消息（Retained Message）是什么？有什么坑？** ⭐
考点：细节题，能讲出坑的人才算真用过。
要点：

- **PUBLISH 时置 RETAIN 标志**，broker 会**为该 topic 保存最后一条消息**；**任何客户端后续订阅该 topic 时，会立即收到这条保留消息**（且收到的 PUBLISH 报文 RETAIN 标志为 1，客户端可据此识别是"历史状态"而非"实时事件"）。
- **典型用途**：设备最新状态、传感器最新读数、配置版本号、上下线状态——**新订阅者无需等待下一次上报即可拿到当前值**。
- **坑 1（最常见）**：**保留消息是"状态"不是"事件流"**。把告警、日志这类事件设为 retained，会导致**每个新订阅者都收到一条陈旧告警**，业务侧误判为实时告警 → 重复告警。
- **坑 2**：**删除保留消息的方法是向该 topic 发布一条 RETAIN=true 且 Payload 为空的消息**（broker 收到空载荷保留消息会删除已存的保留消息）。很多人不知道这个操作。
- **坑 3**：**保留消息不参与 QoS 离线队列机制**，它是"最后值缓存（LVC）"，只存一条，不是队列。
- **坑 4（运维）**：百万设备 × 每设备若干 retained topic = **broker 内存/存储压力**。EMQX 等平台对 retained 消息有**总量上限**（例如 EMQX Cloud 有单连接/全局数量配额，超过连接数阈值后上限固定），**生产必须评估并配置上限与过期时间**（EMQX 支持 retained 消息过期）。

---

## 三、QoS 与消息可靠性（⭐⭐ 最高频考区）

**12. QoS 0 / 1 / 2 分别怎么工作？各自的报文交互过程？** ⭐⭐⭐
考点：MQTT 面试第一必考题，必须能画出来。
要点：

- **QoS 0 — At most once（至多一次）**：
  `PUBLISH →` 完。**无任何确认**，消息可能丢失。
  适用：**高频遥测、传感器周期上报**（丢一两个点无影响，下一周期会有新值）；追求极致吞吐与最低开销。
- **QoS 1 — At least once（至少一次）**：
  
  ```
  Sender:  PUBLISH(PacketId=10) ──►
  Receiver: ◄── PUBACK(PacketId=10)
  ```
  
  **发送方在收到 PUBACK 前必须保存消息并重传**，重传报文 **DUP 标志置 1**。
  → **可能重复投递**，**接收方必须做幂等处理**。
  适用：**绝大多数业务场景**——告警、指令下发、状态变更。**QoS1 + 业务幂等是工业界主流方案。**
- **QoS 2 — Exactly once（恰好一次）**：
  
  ```
  Sender:    PUBLISH(PacketId=10) ──►
  Receiver:  ◄── PUBREC(PacketId=10)      # 已收到，请别重传 PUBLISH
  Sender:    PUBREL(PacketId=10) ──►      # 请交付给上层应用
  Receiver:  ◄── PUBCOMP(PacketId=10)     # 交付完成
  ```
  
  **四次握手**，靠 `PUBREC/PUBREL/PUBCOMP` 三段状态确保**上层应用只收到一次**。
  适用：**计费、支付、不可重复执行的指令**（如"开锁一次"、"扣款一次"）。
  代价：**报文数 4 倍、broker 需为每条在途消息维护状态、内存与延迟开销显著**。**实际项目中 QoS2 使用率很低**，多数团队选择"QoS1 + 业务侧幂等键去重"，更可控也更好排查。
- **关键认知（拉开差距）**：**MQTT 的 QoS 只保证"客户端 ↔ broker"这一段的传输语义，不保证端到端**。
  设备以 QoS1 发到 broker、broker 以 QoS0 转发给订阅者 → 整体效果就是 QoS0。**发布 QoS 与订阅 QoS 取两者中较小值生效。** 所以设计时必须**两端都配到位**。

**13. QoS 1 的重复消息怎么去重？DUP 标志能用来去重吗？** ⭐⭐
考点：能答对"不能"的人很少。
要点：

- **DUP 标志只表示"这是同一 PacketId 的重传报文"，不能用于业务去重**。原因：① **PacketId 是连接级别的 16 位循环序号（1~65535），断开重连后会重新分配**，跨会话无法唯一标识一条业务消息；② DUP=1 也可能出现在 broker 向订阅者重投离线消息时，与业务语义无关。
- **正确去重方案**：**业务层携带唯一消息 ID**（如 `msgId = deviceId + seq` 或雪花 ID / UUID），放在 **payload 里**，或 **MQTT 5.0 用 User Properties 承载**（更干净，不污染业务载荷）。
- 消费端用 **Redis `SETNX msgId + TTL`**（TTL 覆盖最大重传窗口）或 **数据库唯一索引** 做幂等落库。
- 指令类消息：**指令要设计幂等语义**——同一 `cmdId` 重复到达时，设备侧应识别并只执行一次（如"设置温度为 26℃"天然幂等，"计数器 +1"不幂等，后者必须带序号）。

**14. 什么是 Inflight（在途）消息？消息顺序有保证吗？** ⭐
考点：进阶细节。
要点：

- **Inflight = 已发送但尚未收到最终确认的 QoS1/QoS2 消息**。客户端与 broker 都有 inflight 窗口上限（EMQX 有 `max_inflight` 配置，默认常见为 32），**窗口满时后续消息排队等待**。
- **MQTT 5.0 的 Receive Maximum** 属性让双方在 CONNECT/CONNACK 中**协商 inflight 上限**，实现协议级流量控制（3.1.1 只能靠 broker 单方面配置）。
- **顺序保证（重要且反直觉）**：
  - **QoS0 消息按发送顺序投递**（不重传，直接排队发出）。
  - **QoS1/QoS2 在 inflight 窗口 > 1 时，不保证严格有序**。原因：消息 A（PacketId=1）重传中、消息 B（PacketId=2）已确认完成 → B 可能先被交付。
  - **想要严格有序**：把 **inflight 窗口设为 1**（串行化，牺牲吞吐），或**业务层带序号由消费端排序**。
  - 加分句：**同一 deviceId 的消息应路由到 Kafka 同一分区**（按 deviceId hash），保证下游处理有序——这是接入层到消息层的完整顺序保障链。

**15. 消息不丢的完整链路怎么设计？（端到端可靠性）** ⭐⭐
考点：综合能力题，几乎必问。
要点（按链路分段讲，每段说清"哪里会丢"和"怎么兜住"）：

```
设备 ──①──► Broker(EMQX) ──②──► Kafka ──③──► 清洗消费者 ──④──► 时序库/告警
```

1. **设备 → Broker**：设备端用 **QoS1 + cleanSession=false**；网络中断时**本地缓存待发消息**（Flash/环形缓冲），重连后补发，配合 **msgId 幂等**；MQTT 5.0 用 **Message Expiry Interval** 防止补发过期数据（如温度读数过期就没意义，但告警要保留）。
2. **Broker → Kafka**：用 **EMQX 规则引擎 + 数据桥接（Kafka Sink）**，配置 **QoS1 订阅 + 桥接重试 + 死信**；EMQX 5.x 提供**缓冲与重试机制**，Kafka 不可用时消息在 broker 侧缓冲。**关键：broker 到 Kafka 这一跳是最容易丢的地方**（异步桥接失败静默丢弃），必须开启失败重试与告警监控。
3. **Kafka 侧**：Producer **acks=all + retries + 幂等/事务**；Topic **副本数 ≥ 3、min.insync.replicas ≥ 2**；Consumer **手动提交 offset，处理成功后再提交**（先提交后处理 = 丢消息）。
4. **消费 → 存储**：写库失败重试 + 死信队列 + 人工兜底；时序库批量写入要处理**部分失败**。
5. **兜底手段**：**离线消息补传**（cleanSession=false）、**设备端本地缓存**、**对账任务**（定时比对设备侧计数与平台侧入库数，发现缺口触发补拉）——**你在绿城做的"多方子系统数据一致性工具"就是这一层，可以直接引用**。
- 收口句：**"没有任何单点能保证不丢，可靠性是分段设计 + 端到端对账的组合结果。"**

**16. MQTT 的 QoS 和 Kafka 的 acks 是同一回事吗？**
考点：概念辨析，防止混淆。
要点：**不是同一层的东西，不能直接类比。**

- MQTT QoS 是**发布/订阅协议在"客户端↔broker"链路上的投递语义**，由协议报文交互（PUBACK/PUBREC…）实现。
- Kafka `acks` 是**生产者等待多少副本确认落盘**，管的是"消息在 broker 集群内的持久化可靠性"，与消费者是否收到无关（消费侧靠 offset 提交语义）。
- 组合关系：设备 QoS1 到 EMQX，EMQX 桥接到 Kafka 时配 `acks=all`，Kafka 消费者手动提交 offset → **三段各自保证，串起来才是端到端可靠**。

---

## 四、Topic 设计与订阅机制

**17. Topic 的结构规则？通配符 `+` 和 `#` 怎么用？有哪些限制？** ⭐⭐
要点：

- **Topic 是层级字符串，用 `/` 分隔**（`device/A001/data/up`），**不是队列名，没有"创建"操作**——发布即存在。
- **通配符只能用于 SUBSCRIBE，不能用于 PUBLISH**：
  - **`+` 单层通配**：`device/+/data` 匹配 `device/A001/data`、`device/B002/data`，**不匹配** `device/A001/x/data`。
  - **`#` 多层通配**：`device/#` 匹配 `device` 下所有层级；**`#` 必须是最后一个字符且前面必须是 `/`**（`device/#` 合法，`device#` 非法）。
  - **单独的 `#` 匹配所有 topic** —— 生产环境**必须用 ACL 禁止设备订阅 `#`**，否则一台设备能偷听全平台数据。
- **特殊 topic**：以 **`$` 开头的 topic 是服务端保留主题**（如 EMQX 的 `$SYS/` 监控主题），**通配符 `#` 与 `+` 不匹配 `$` 开头的 topic**，必须显式订阅 `$SYS/#`，且应限制为管理员权限。
- **限制**：topic **不能包含通配符字符**；有长度上限（UTF-8 编码，规范上限 65535 字节）；**区分大小写**；**避免以 `/` 开头或结尾**（会产生空层级，`/device` 与 `device` 是不同 topic，且空层级易引发匹配歧义）。
- **性能提示**：**层级过深、topic 数量爆炸**（如把时间戳、随机数放进 topic → `device/A001/20260906120000/data`）会**让 broker 的 topic 路由表急剧膨胀**。时间维度应该放在 **payload 里**，不是 topic 里。

**18. 设计一套 IoT 平台的 Topic 规范，你会怎么设计？** ⭐⭐
考点：开放设计题，能讲出层次就是资深。
要点（给一份可直接讲的规范）：

```
上行：  device/{productKey}/{deviceId}/telemetry      # 周期遥测
        device/{productKey}/{deviceId}/event/{type}   # 事件（告警、故障）
        device/{productKey}/{deviceId}/status         # 上下线（retained）
        device/{productKey}/{deviceId}/cmd/{cmdId}/reply   # 指令响应

下行：  device/{productKey}/{deviceId}/cmd/down       # 指令下发（设备订阅）
        device/{productKey}/{deviceId}/config/push    # 配置推送
        device/{productKey}/{deviceId}/ota/notify     # 升级通知

平台侧：sys/{region}/metrics/#                        # 内部业务 topic
```

设计原则逐条讲：

- **层级从粗到细**：产品/租户在前，设备在后，业务类型在最后 → **便于按层级做 ACL**（`device/{productKey}/{deviceId}/#` 精确到单设备）与**按前缀做规则引擎路由**。
- **上下行分离**：设备只订阅 `cmd/down` 类 topic，只发布 `telemetry` 类 topic，**ACL 可精确到动作（publish/subscribe）**。
- **productKey 单独一层**：便于按产品批量订阅（`device/{productKey}/+/telemetry`）、按产品做灰度和限流。
- **指令响应带 cmdId 一层**：`cmd/{cmdId}/reply` 让**请求-响应关联在 topic 层面就可见**，不用只靠 payload 里的 seq；也方便规则引擎按 cmdId 路由。
- **不把易变值放 topic**：时间戳、随机数、消息序号一律进 payload（呼应第 17 题性能提示）。
- **配套 ACL 模板**（EMQX 授权占位符）：
  
  ```
  allow publish    device/${clientid}/telemetry
  allow publish    device/${clientid}/event/#
  allow publish    device/${clientid}/cmd/+/reply
  allow subscribe  device/${clientid}/cmd/down
  deny  subscribe  device/#        # 禁止跨设备越权
  ```
  
  注意 `${clientid}` 要与 topic 中的 deviceId 位置对应，**若 topic 含 productKey，需保证 clientid 能推导出完整前缀，或用 username 占位符 + 授权数据源查询**。
- 加分：提到 **MQTT 5.0 Topic Alias** —— 高频上报时 topic 字符串重复传输浪费带宽，别名机制可显著省流量（**只在本次连接内有效，重连后需重新协商**）。

**19. 共享订阅（Shared Subscription）是什么？解决什么问题？** ⭐⭐
考点：集群消费的关键机制，必问。
要点：

- **问题背景**：普通订阅下，**同一 topic 被 N 个消费者订阅时，每个消费者都会收到全量消息**（广播语义）。业务服务多实例部署时会导致**重复处理**。
- **共享订阅**：topic 前加 `$share/{ShareName}/`，**同一 ShareName 的多个订阅者构成一个消费组，broker 把每条消息只投递给组内其中一个成员**（负载均衡语义），等价于 Kafka 的 consumer group。
  
  ```
  订阅：$share/consumer-group/device/+/telemetry
  ```
- **MQTT 5.0 才标准化**；3.1.1 时代是各家私有实现（EMQX 4.x 就有 `$share`，Moquette/VerneMQ 语法不同）→ **平台迁移时要注意兼容**。
- **分发策略**：EMQX 支持 `round_robin`、`random`、`sticky`、`hash_clientid`、`hash_topic` 等。**要保证同一设备消息被同一消费者处理（有序/有状态聚合）就用 `hash_clientid` 或 `hash_topic`**。
- **注意坑**：① 共享订阅**不支持离线消息**（组成员掉线，消息分发给其他在线成员，**不会为掉线成员保留**）；② **同一客户端不能用同一 ShareName 重复订阅同一 topic**；③ 组内成员数超过消息速率时会有空闲消费者。
- **实战结论**：如果下游是重度业务处理（清洗、聚合、写库、告警计算），**更推荐 EMQX 规则引擎桥接到 Kafka，由 Kafka consumer group 消费**——Kafka 有持久化、可重放、offset 管理、积压监控，比 MQTT 共享订阅可靠得多。**共享订阅适合轻量、实时、可容忍丢失的消费场景。**

**20. 订阅关系是持久化的吗？重连后要重新订阅吗？**
要点：取决于 **Clean Session**（见第 8 题）。

- `cleanSession=true`：**会话不持久化，重连后必须重新 SUBSCRIBE**（客户端 SDK 通常在 `connectComplete` 回调里统一重订阅，**不要在初始化时只订阅一次**）。
- `cleanSession=false`：**broker 保留订阅关系**，重连后 CONNACK 的 `Session Present=1`，**无需重订阅**。
- **实践建议**：**无论哪种，客户端都实现"连接成功后幂等重订阅"** —— 重复 SUBSCRIBE 是安全的（broker 会更新而非叠加），这样代码对 cleanSession 配置变化不敏感，更健壮。

---

## 五、Broker 与集群（EMQX 相关，JD 点名）

**21. EMQX 的核心能力有哪些？和 Mosquitto 的区别？** ⭐⭐
考点：JD 直接写了 EMQX/Mosquitto，必答。
要点：

- **EMQX**：Erlang/OTP 编写，**面向海量连接的分布式 MQTT 平台**。核心能力：
  - **集群**：Masterless 无主架构（5.x 用 **Mria** 集群方案），支持水平扩展，官方宣称企业版单集群最高 **1 亿并发连接**，单节点基准测试（64核128G）可达 **500 万连接**，企业版文档标注单节点稳定支持 **150 万**。
  - **规则引擎**：**类 SQL 语法**对消息做过滤/转换/路由（`SELECT payload.temp FROM "device/+/telemetry" WHERE payload.temp > 80`）。
  - **数据集成/桥接**：把消息转 **Kafka、MySQL、PostgreSQL、Redis、MongoDB、时序库（TDengine/InfluxDB）、HTTP Webhook** 等。
  - **多协议网关**：**MQTT-SN、CoAP、LwM2M、STOMP、OCPP 1.6（充电桩）、JT/T 808、GB/T 32960（车联网国标）、ExProto（自定义协议）**。
  - **认证授权链**：内置数据库 / MySQL / PostgreSQL / MongoDB / Redis / LDAP / **HTTP Server** / JWT / X.509 / TLS-PSK / MQTT 5.0 SCRAM；授权同理成链。
  - **Dashboard + 可观测性**：连接数、消息速率、订阅数、规则命中率、客户端在线列表与手动踢下线，支持 Prometheus 指标导出。
- **Mosquitto**：C 语言编写，**轻量单节点 broker**。优点是极省资源、部署简单、适合边缘/嵌入式网关；**缺点是单机架构、无原生集群、无规则引擎与数据集成、管理能力弱**。默认连接数约 1 万，调优后可到 2 万级。
- **选型结论**：**边缘侧/单机小规模 → Mosquitto 或 NanoMQ；云端海量设备接入 + 需要规则引擎和数据桥接 → EMQX**。二者可用 **MQTT 桥接（bridge）** 打通：边缘 Mosquitto 作为客户端桥接到云端 EMQX，实现边云协同。
- 其他同类：HiveMQ（商业，Java）、VerneMQ（Erlang）、NanoMQ（超轻量，边缘）、AWS IoT Core / 阿里云 MQTT（托管）。

**22. EMQX 集群怎么工作？设备连到不同节点，消息怎么互通？** ⭐
考点：集群原理。
要点：

- **Masterless 架构**：节点地位对等，无单点主节点；新节点加入通过**节点发现机制**（manual / dns / etcd / k8s）。
- **路由表机制（关键）**：EMQX 集群维护**两层信息** —— ① **订阅路由表**（topic → 哪些节点上有订阅者，**只记节点，不记具体客户端**，避免路由表随设备数爆炸）；② 节点本地的**订阅明细表**（本节点上哪些 client 订阅了什么）。
- **消息流转**：设备连到节点 A 发布消息 → 节点 A **查路由表得知哪些节点有订阅者** → 转发给这些节点 → 各节点查本地订阅明细 → 投递给本地客户端。**这就是"设备连不同节点也能互通"的原理。**
- **Mria（5.x）**：EMQX 5.0 引入的新集群方案，区分 **核心节点（core）与复制节点（replicant）**，内置数据库（如 retained 消息、路由表）在核心节点强一致复制，**复制节点可独立降级运行**，显著提升**大规模集群的水平扩展能力与网络分区容忍度**。
- **负载均衡接入**：设备侧不直连节点，而是通过 **LB（LVS/HAProxy/NGINX/DNS 轮询/云 SLB）** 分发；**TCP 长连接建议用四层 LB**，配合 **PROXY Protocol** 保留客户端真实 IP（EMQX 支持 PROXY Protocol V1/V2），否则所有设备 IP 看起来都是 LB 的 IP，**基于 IP 的限流与 ACL 全部失效**——这是个很实际的坑，讲出来很加分。

**23. EMQX 规则引擎和数据桥接怎么用？为什么不直接在 broker 里写业务逻辑？** ⭐⭐
考点：JD 第 3 条"数据接收、清洗、转发、存储"的直接对应。
要点：

- **规则引擎**：类 SQL 三段式 —— **事件源（FROM，topic 或 `$events/client_connected` 等系统事件）+ 条件（WHERE）+ 输出字段（SELECT）**；支持 SQL 函数做转换（取 payload 字段、时间戳格式化、编解码）。

- **典型用法**：
  
  ```sql
  -- 高温告警：提取超阈值数据，转 Kafka 告警 topic
  SELECT clientid, payload.deviceId as deviceId, payload.temp as temp, timestamp
  FROM "device/+/telemetry"
  WHERE payload.temp > 80
  
  -- 设备上下线事件写库
  SELECT clientid, connected_at, reason FROM "$events/client_connected"
  ```

- **动作（Action）**：转发到 **Kafka / RabbitMQ / MQTT 桥接 / MySQL / PostgreSQL / Redis / MongoDB / InfluxDB / TDengine / HTTP Webhook / 数据持久化（Republish 到另一 topic）**。

- **为什么不在 broker 里写重业务逻辑**：
  
  1. **规则引擎擅长"过滤、转换、路由"，不擅长"状态"** —— 涉及跨消息聚合、状态机、事务、复杂业务编排时，规则引擎表达力不足且难以调试。
  2. **EMQX 是 Erlang 写的，Java 团队改不动内核**，只能通过配置、规则 SQL、Webhook、ExHook 扩展。
  3. **broker 里跑重逻辑会拖垮接入层**，影响所有设备的连接稳定性 —— **接入层必须保持轻量和无状态**。
  - **正确分层**：`broker 规则引擎只做粗筛与转发 → Kafka 缓冲解耦 → Java 服务做清洗、聚合、告警计算、入库`。

- **加分**：主动说明**桥接失败的处理** —— Kafka 不可用时消息在 broker 缓冲，需配置**重试次数、缓冲上限、失败告警**，否则**桥接失败会静默丢消息**，这是生产事故高发点。

**24. 百万设备接入，broker 侧要注意什么？** ⭐
考点：容量规划视野。
要点：

- **内存是第一瓶颈**：每连接的**会话状态（订阅表 + inflight + 离线消息队列）** + Erlang 进程开销。`cleanSession=false` 的设备越多，内存占用越高。
- **离线消息与 retained 消息必须设上限**：EMQX 有 `max_mqueue_len`（离线消息队列长度）、retained 消息总量与过期配置。**不设上限，一次大规模断网重连就能把 broker 打爆。**
- **连接建立速率**：批量重连风暴（机房断网恢复、broker 重启）时，**建连速率**比总连接数更容易成为瓶颈（官方基准建连速率 5000/s 量级）。对策：**设备端指数退避 + 抖动**、**接入层限流**（EMQX 支持 listener 级 `max_conn_rate` 连接建立速率限制与并发连接上限）。
- **文件句柄与内核参数**：`ulimit -n`、`somaxconn`、`tcp_max_syn_backlog`、Erlang VM 参数（`+P` 进程数上限、`+Q` 文件描述符上限）。
- **消息吞吐**：官方性能参考显示单节点 1KB payload 约 **40K TPS**，平均时延 3ms 级、P99 12ms 级；**payload 越大 CPU 占用越高**（8KB 时 CPU 已到 90%+）→ **控制单条消息大小、拆分大报文**。
- **监控指标**：连接数、消息速率、订阅数、**规则引擎命中率与失败数**、**桥接积压/失败数**、Erlang VM 内存与进程数、GC/调度延迟。
- **务实结论**：**单节点 10 万~50 万连接是稳妥区间，超过就集群扩展**；官方 500 万是特定高配环境基准，**不要拿基准数据当生产容量承诺**——这句话讲出来面试官会认为你有生产判断力。

**25. EMQX 怎么做设备认证与授权？认证链是什么？** ⭐⭐
考点：呼应上一轮讨论的"一机一密"，高频。
要点：

- **认证（AuthN，连接阶段）**：校验 CONNECT 中的 clientId/username/password。数据源可选：**内置数据库（Mnesia）、MySQL、PostgreSQL、MongoDB、Redis、LDAP、HTTP Server、JWT、X.509 证书、TLS-PSK、MQTT 5.0 SCRAM**。
- **认证链**：可配多个认证器按序执行，**一旦某个认证成功即终止链并放行**；全部失败则拒绝连接。**注意 EMQX 默认不开启认证，允许所有客户端连接，生产必须显式配置**（这是最容易踩的安全坑）。
- **密码存储**：支持 plain/md5/sha/sha256/sha512/bcrypt/pbkdf2，**每台设备可配独立 salt**，数据库只存密文，防拖库后被彩虹表撞。
- **占位符**：SQL/Redis 命令/HTTP 请求体可用 `${clientid}`、`${username}`、`${peerhost}` 等客户端属性做动态参数，例如：
  
  ```sql
  SELECT password_hash, salt FROM device_auth WHERE device_id = ${clientid}
  ```
- **授权（AuthZ，发布/订阅阶段）**：控制 client 能 publish/subscribe 哪些 topic。数据源：**ACL 文件、内置数据库、MySQL、PostgreSQL、MongoDB、Redis、HTTP**，同样构成授权链；**未匹配时按 `no_match` 配置决定 allow/deny，生产务必设为 deny**（默认 allow 是巨大隐患）。
- **一机一密完整落地 = 认证 + 授权两条链都配好**。只做认证的话，设备 A 用合法凭证订阅 `device/B/#` 就能偷听设备 B 的全部数据。
- **HTTP 认证源最灵活**：可在一次回调里查密钥、租户、设备状态、黑名单、并发连接数，把复杂业务规则收在自己的 Java 服务里，broker 只做转发。
- **配套安全**：`client.disconnected` 等系统事件可订阅，用于感知会话接管/异常掉线；结合 **Topic ACL + 连接限流 + TLS** 构成完整防护。

---

## 六、场景设计与排障（压轴，占分最重）

**26. 设计一个"设备离线数据不丢"的方案。** ⭐⭐
考点：综合能力题，把 QoS + 会话 + 缓存串起来。
要点（分四段讲）：

1. **协议层**：设备用 **QoS1 + cleanSession=false**；broker 为离线设备保留订阅关系与 QoS1 消息，**重连后 Session Present=1 自动补发**。
2. **设备端**：网络中断时**本地缓存（Flash/环形队列）**未发送消息，带 **msgId + 采集时间戳**；重连后按序补发。用 **MQTT 5.0 Message Expiry Interval** 给不同类型数据设不同过期时间（温度读数 10 分钟过期，告警不过期）。
3. **Broker 侧容量保护**：设置 `max_mqueue_len` 离线消息上限，**超限时丢弃最旧消息并打点告警**——宁可丢部分遥测，也不能让 broker OOM 影响全部设备。
4. **平台侧对账兜底**：设备侧维护发送计数，平台侧维护入库计数，**定时对账发现缺口 → 触发主动补拉（下发"请上传 X 时段数据"指令）**。这是最后一道防线，也是你绿城"数据一致性工具"的思路。
- **必须说的取舍**：**"完全不丢"和"系统可用性"是有冲突的**。追求绝对不丢会导致 broker 内存无限增长。工程上的正确答案是**分级**：告警/计费类数据用 QoS2 或 QoS1+持久化+对账，普通遥测用 QoS1 且允许在极端情况下丢弃最旧数据。

**27. 设备频繁上下线（抖动）怎么处理？** ⭐
考点：真实运维痛点。
要点：

- **根因排查**：① 弱网/信号差；② **Keep Alive 设置过短**（设备休眠周期 > keepalive 就会被误判掉线）；③ **ClientId 冲突互相挤下线**；④ 服务端 GC 停顿或 LB 超时配置过短导致连接被断；⑤ NAT 超时时间短于 keepalive；⑥ 设备固件 bug 主动断连。
- **判定层面**：**延迟确认**——不是一收到断开事件就置 offline，而是**启动一个短窗口（如 30~60s），窗口内重连则视为未离线**；或用 MQTT 5.0 **Will Delay Interval** 让遗嘱延迟发布，协议层直接抑制抖动。
- **状态收敛**：状态变更**去重**（同一状态不重复写库、不重复发事件），**加最小变更间隔**（如 60s 内同一设备状态跳变超过 N 次则合并为一条"网络不稳定"告警，而不是发 N 条上下线告警）。
- **告警分级**：**单设备抖动 → 不告警或低优先级；同一区域/同一产品批量抖动 → 高优先级告警**（往往是网络故障或服务端问题的信号）。
- **数据侧**：设备在线率统计要用**时间加权**而不是简单计数，否则抖动设备会严重污染指标。

**28. 平台要给一台离线设备下发指令，怎么设计？** ⭐⭐
考点：设备影子 + 指令可靠性，必问。
要点：

- **核心矛盾**：设备离线时 TCP 连接不存在，指令无处可发。
- **方案一：设备影子（Device Shadow）/ 期望状态**
  - 影子包含 **desired（平台期望值）** 与 **reported（设备上报值）** 两段。
  - 平台下发时写入 `desired`（持久化存储，如 MySQL/Redis）；**设备上线后主动 GET 影子（或用 retained topic 立即收到）**，比对 desired 与 reported，执行差异动作后回写 reported；平台确认 desired 已达成后清空。
  - **注意**：**设备影子不是 MQTT 协议原生概念，是 AWS IoT / 阿里云 IoT / 各平台的应用层设计**，用 retained message + 持久化存储实现。这句区分能体现你概念清晰。
- **方案二：QoS1 + cleanSession=false 离线消息**
  - 指令发到设备订阅的 topic，**broker 因会话保持而缓存，设备重连后自动投递**。
  - 局限：**受 broker 离线消息上限约束**；**只对 QoS1/2 生效**；**设备长期离线会导致消息过期或堆积**；**多条指令按序堆积可能不是设备期望的（如"设置温度 26"后又"设置 28"，两条都投递会做无用功）**。
- **方案三（推荐组合）**：**影子承载"最终期望状态"（幂等、可覆盖）+ 在线时用 QoS1 实时下发 + 离线时只更新影子**。
  - 指令设计要**幂等且可覆盖**：用"设置目标值为 X"而非"执行动作 Y 次"。
  - 平台侧维护**指令生命周期状态机**：`待下发 → 已下发 → 设备已接收 → 执行成功/失败 → 超时`；每条指令带 **cmdId + 超时时间**，超时告警并可选重试（**重试必须用同一 cmdId 保证设备侧幂等**）。
- **响应关联**：MQTT 5.0 用 **Response Topic + Correlation Data** 原生支持；3.1.1 用 payload 里的 cmdId 关联，或用 `cmd/{cmdId}/reply` topic 结构。

**29. 怎么排查"设备显示在线但收不到数据"？** ⭐⭐
考点：线上排障，最能看出实战经验。
要点（按排查顺序讲，体现方法论）：

1. **确认连接状态真伪**：查 broker Dashboard 的客户端列表，看 **连接时间、Keep Alive、订阅列表、inflight 数、是否半开连接**。设备端说在线但 broker 无此 client → **ClientId 不一致或被挤下线**。
2. **确认订阅关系**：设备是否真的 SUBSCRIBE 成功了？**`cleanSession=true` 重连后没重订阅**是最常见原因。查 broker 侧该 client 的订阅列表。
3. **确认 ACL**：设备订阅的 topic 是否被授权链 **deny** 了？**SUBACK 返回 0x80（3.1.1）就是订阅被拒**——而 3.1.1 只给这一个笼统码，看不出原因，**升 5.0 才能拿到细分 Reason Code**（这个痛点讲出来很有说服力）。
4. **确认 topic 是否匹配**：层级拼错、大小写不符、多了/少了 `/`、通配符位置不对（`device#` 非法）、**`$` 开头 topic 不被 `#` 匹配**。
5. **确认 QoS 与 retain**：发布方 QoS0 + 订阅方离线 → 消息直接丢；**把"事件"当"状态"用，或依赖 retain 但发布时没置 RETAIN 标志**。
6. **确认桥接/规则引擎**：EMQX 规则引擎 WHERE 条件写错导致过滤掉全部消息（**看规则命中数指标**）；**Kafka 桥接失败静默丢弃**（看桥接失败计数与积压）。
7. **确认下游消费**：Kafka consumer lag 堆积、消费者组 rebalance、消费异常未提交 offset 导致重复处理或卡死。
8. **抓包定位**：**在 broker 侧抓包看 PUBLISH 是否到达、SUBACK 返回码是什么** —— 这是最快分清"设备没发"和"平台没处理"的手段（可用 Wireshark 解 MQTT）。
- **方法论收口**：**从设备 → broker → 桥接 → Kafka → 消费者 → 存储，逐段确认"消息到哪一步断了"**，配合每段的监控指标，不要一上来就猜。

**30. MQTT 消息堆积/积压了怎么办？** ⭐
要点：

- **先定位堆积在哪一层**：broker 的 **mqueue（离线消息队列）**？桥接缓冲区？Kafka topic lag？消费者处理慢？
- **Broker 侧堆积**：`max_mqueue_len` 已满 → 说明**下游消费能力不足或设备长期离线**；处置：调大上限（评估内存）、**丢弃低价值 topic 的离线消息**、加速下游消费。
- **桥接侧堆积**：Kafka 不可用或写入慢 → **检查桥接失败计数与重试配置**，Kafka 恢复后自动追平；必要时临时降级（丢弃非关键 topic）。
- **Kafka 侧积压（lag）**：① **扩分区 + 扩消费者**（消费者数不能超过分区数）；② 优化消费逻辑（批量入库代替单条、异步化、去掉慢查询）；③ **临时旁路**：新起一组消费者把数据直接写入，跳过重逻辑；④ 降级：非关键数据直接丢弃或降采样。
- **根因治理**：**接入侧限流**（EMQX listener 连接与消息速率限制）、**设备端上报频率优化**（变化上报而非周期上报、批量打包上报）、**topic 分级**（关键数据与普通遥测分 topic，积压时可差异化处理）。
- **监控前置**：**堆积必须有告警**，等用户报障才发现就晚了。指标：mqueue 长度、桥接失败数、Kafka lag、消费 RT。

---

## 七、与其他技术栈的对比选型

**31. MQTT 和 Kafka 能互相替代吗？在 IoT 架构里各自扮演什么角色？** ⭐⭐
考点：架构视野题，高频。
要点：**不能替代，是互补关系，分处链路两端。**
| 维度 | MQTT | Kafka |
|---|---|---|
| 定位 | **设备接入协议**（南向） | **数据流转与缓冲平台**（东西向/北向） |
| 连接模型 | 海量长连接（百万级客户端） | 少量生产者/消费者，高吞吐 |
| 消息持久化 | 弱（离线消息有上限、retained 只存一条） | **强（磁盘顺序写，可保留数天并重放）** |
| 消费模型 | 发布订阅 + 共享订阅 | **消费组 + offset，可重放、可回溯** |
| 单消息大小 | 小（几十字节到几 KB 最优） | 较大（默认 1MB，可调） |
| 顺序保证 | 同连接内 QoS0 有序，QoS1/2 不严格 | **分区内严格有序** |
| 运维复杂度 | broker 轻量 | 集群重（含 ZK/KRaft） |

- **标准 IoT 架构**：`设备 → MQTT/EMQX（接入层）→ 规则引擎桥接 → Kafka（缓冲解耦层）→ Java 服务（清洗/告警/入库）→ 时序库 + 业务库`
- **为什么中间要有 Kafka**：① **削峰**（设备整点集中上报的洪峰不打垮存储）；② **解耦**（多个下游系统各自消费，互不影响）；③ **可重放**（消费逻辑改了能重新处理历史数据）；④ **持久化可靠**（MQTT 不是为长期存储设计的）。
- **加分句**：**EMQX 5.x 也在往"消息队列能力"上走**（内置基于 RocksDB 与 Raft 的分布式存储，宣称可替代传统 MQTT Broker + Kafka/RabbitMQ 组合以降低基础设施成本），但**复杂业务流转、多消费组、长周期重放的场景，Kafka 仍是更成熟稳妥的选择**。

**32. 什么时候该用 MQTT，什么时候用 HTTP/WebSocket？** ⭐
要点：

- **用 MQTT**：设备数量大（万级以上）、需要**服务端主动下发**、上行频繁、网络不稳定、按流量计费、需要离线消息能力。
- **用 HTTP**：**偶发、低频的数据上报**（如一天一次的抄表）、**需要浏览器直接访问**、**要利用 CDN/网关/缓存生态**、**文件上传**（大文件走 MQTT 不合适）、**设备一次性激活/注册/OTA 下载固件包**。
- **用 WebSocket**：浏览器端实时看板需要推送时（MQTT over WebSocket 也可，EMQX 支持 8083/8084）。
- **实战组合**：**很多平台是混合的** —— 遥测走 MQTT，**OTA 固件包下载走 HTTP**（大文件 + 断点续传 + CDN），**设备激活与密钥下发走 HTTPS**（一次性、需强安全）。**能讲出这种混合架构说明你真做过。**

**33. Sparkplug B 了解吗？**
考点：工业物联网场景可能问到（NeuronEX 生态）。
要点：**Sparkplug B 是构建在 MQTT 之上的工业物联网应用层规范**（Eclipse Tahu 项目），解决"MQTT 只定义传输、不定义语义"的问题。核心：

- **统一 topic 命名空间**：`spBv1.0/{group_id}/{message_type}/{edge_node_id}/{device_id}`，消息类型含 **NBIRTH/NDATA/NDEATH/DBIRTH/DDATA/DDEATH/CMD** 等（B=出生含完整元数据，D=数据变化上报，DEATH=节点离线遗嘱，CMD=指令下发）。
- **定义统一 payload 格式**（Protobuf 编码的 metric 列表，带名称、数据类型、时间戳、别名），**实现设备语义自描述**，上层 SCADA/MES 无需为每家厂商写解析。
- **状态管理内建**：用 MQTT 遗嘱实现 **NDEATH/DDEATH**，天然支持节点在线状态感知。
- **EMQX Neuron/NeuronEX 支持 Sparkplug B 北向插件**，把 Modbus/OPC-UA 采集的数据按 Sparkplug B 规范上送。
- 适用场景：**工业物联网、需要与 SCADA/MES 打通、多厂商设备语义统一**。普通消费级 IoT（智能家居、共享设备）一般不用。

---

## 八、快问快答（查漏补缺）

**34. MQTT 的 Packet Identifier 有什么规则？**
**16 位整数，取值 1~65535**；**QoS0 不使用 PacketId**；**QoS1/QoS2 必须有**；同一连接内**未确认消息的 PacketId 不能重复**，用尽后回绕（所以要控制在途窗口）；**断开重连后重新分配，不能跨会话用于去重**（见第 13 题）。

**35. MQTT 5.0 的 Reason Code 有什么价值？**
CONNACK/PUBACK/SUBACK/DISCONNECT 等都带细分原因码。**3.1.1 的 SUBACK 失败只返回 0x80 一个值**，排障时根本不知道是 ACL 拒绝、topic 非法还是配额超限；5.0 能给出 `0x87 未授权`、`0x8F topic 名非法`、`0x97 配额超限`、`0x9B 报文过大` 等精确原因。**对平台可观测性是质的提升。**

**36. MQTT 支持消息广播吗？**
**协议层面没有广播概念**，用 **topic 通配符订阅**近似实现：所有设备订阅 `device/+/cmd/broadcast`，平台发布到该 topic 即全体收到。注意：**百万设备同时收到广播会引发处理风暴**，需配合**分批下发、随机延迟执行、共享订阅分组**等手段削峰。

**37. MQTT 能保证消息不重复吗？**
**QoS0 可能丢；QoS1 至少一次，可能重复；只有 QoS2 在"客户端↔broker"单跳内保证恰好一次。** 而且**QoS 只保单跳不保端到端**，发布 QoS 与订阅 QoS 取小值。**跨系统（broker→Kafka→业务库）的"恰好一次"必须靠业务幂等实现**，不要指望协议。

**38. retained 消息和 QoS 离线消息是一回事吗？**
**不是。** retained 是**每个 topic 只存最后一条**的"最后值缓存"，**新订阅者一连上就收到**，与会话无关；离线消息是**针对 cleanSession=false 的特定客户端**缓存的**队列**，只在该客户端重连后投递。二者机制、存储、投递时机都不同。

**39. MQTT over QUIC 是什么？解决什么问题？**
EMQX 支持的特性。**QUIC 基于 UDP，内建 TLS 1.3、多路复用无队头阻塞、连接迁移（Connection Migration）**。价值：**车联网、移动设备在 4G/WiFi 切换时 IP 变化，TCP 连接必须重建，而 QUIC 凭 Connection ID 可以无缝迁移连接**，大幅减少切换导致的断连与重连开销；弱网下握手更快（0-RTT/1-RTT）。

**40. MQTT 的 topic 需要预先创建吗？**
**不需要。** topic 是**发布即存在**的逻辑通道，没有"建 topic"操作（对比 Kafka 必须预创建 topic 与分区）。这是 MQTT 轻量的体现，但也意味着**topic 拼写错误不会报错，只会静默无消息**——排障时要特别注意（第 29 题）。

**41. 怎么监控 MQTT 平台的运行状态？**
要点：**EMQX Dashboard**（连接数、消息速率、订阅数、客户端列表、规则命中率、可手动踢客户端）；**Prometheus 指标导出**（`/api/v5/prometheus/stats`）接入 Grafana；**系统主题 `$SYS/#`**（仅管理员订阅，含 broker 运行指标）；**Webhook/规则引擎订阅 `$events/client_connected`、`$events/client_disconnected`** 把上下线事件推给业务系统。
**核心告警指标**：连接数突降、**建连失败率**、消息速率异常、**规则引擎/桥接失败数**、mqueue 堆积、**会话接管（挤下线）频率**、节点内存与 Erlang 进程数。

**42. 你实际用过 MQTT 吗？遇到什么问题？**
**这题不要背模板，用你自己的真实素材。** 建议叙事方向（结合你的经历改写）：

> "我在大华做充电停车一体化平台时对接过充电桩设备通信，在绿城做物联引擎时对接了近 50 家供应商、30 多种设备类型。多厂商接入最头疼的是**协议和鉴权方式五花八门**——有的走 MQTT，有的是私有 TCP 二进制协议，鉴权有一机一密签名、固定 token、IP 白名单、TLS 证书好几种。如果每接一家就在核心代码里加 if-else，很快就没法维护，而且**旧方案发版要停机，影响全省 400 多个社区的设备在线**。
> 所以我主导做了**插件化物联集成工具**：核心只定义统一的设备模型和身份上下文，协议解析、鉴权校验都做成插件，插件热加载、无损发布——这也是我那项专利要解决的问题。
> 另外还做过**多方子系统数据一致性工具**，处理第三方接口异常导致的数据错乱，思路是状态机 + 对账补偿 + 重试幂等，最终一致性而不是强一致。这两个经验放到 MQTT 场景是通用的：**接入层要能容忍异构与不可靠，可靠性靠分段保证 + 端到端对账兜底。**"

**这段叙事的三个好处**：① 诚实（不说自己精通 MQTT 源码）；② 把"没深度用过 MQTT"转成"解决过更难的多厂商异构接入"；③ 主动落到 JD 关键词（设备接入、协议解析、数据一致性、不停机发布）。

---

## 附：面试前 3 小时冲刺清单

1. **画出 QoS 0/1/2 的报文交互图**（PUBACK / PUBREC-PUBREL-PUBCOMP），边画边讲 1 分钟。
2. **背熟 5 个机制**：Clean Session 与 Session Present、Keep Alive 1.5 倍、遗嘱消息 LWT、保留消息 Retained、共享订阅 `$share`。
3. **写出你的 topic 规范**（第 18 题），能讲 3 分钟并配 ACL 模板。
4. **准备两个设计题的完整答案**：设备离线数据不丢（26 题）、离线设备指令下发（28 题）。
5. **准备排障方法论**：第 29 题的"逐段确认消息断在哪"的排查顺序。
6. **必背数字**：报文头最小 **2 字节**；载荷上限约 **256MB（268,435,455）**；PacketId **1~65535**；Keep Alive 判定 **1.5 倍**；MQTT 端口 **1883/8883**；EMQX 单节点基准 **500 万连接**（64核128G）、企业版文档 **150 万**、集群宣称 **1 亿**；单节点 1KB payload 约 **40K TPS**；Mosquitto 默认约 **1 万连接**。
7. **诚实兜底话术**：「MQTT 我在设备接入场景用过，协议机制和 broker 选型我清楚；EMQX 的规则引擎/集群运维层面我没有大规模生产经验，但接入层的协议解析、鉴权、会话管理我在多厂商物联平台上做过大量实战。」**切忌编造项目细节，被追问三层就露馅。**
