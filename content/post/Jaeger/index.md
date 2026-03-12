---
title: Jaeger链路追踪中间件介绍
date: 2026-03-12
description: 简单介绍 Jaeger 链路追踪
#image: Prometheus.png
categories:
  - 后端
  - 可观测性
  - 微服务

tags:
  - Jaeger
  - OpenTelemetry
  - 链路追踪
  - go-zero
  - 分布式追踪
  - TraceID
  - 审计
  - MQ
  - Kafka
---

# Jaeger 链路追踪，让系统“内鬼”无处遁形

在微服务架构中，当我们完成了**结构化日志（Logger）**的封装，又铺设了 **Prometheus 监控大盘（Metrics）**之后，很多开发者会觉得系统的防线已经固若金汤了。

但当线上海啸真正来临时，您很快就会遇到一个极其让人崩溃的场景：

前端同学跑来抱怨：“点赞接口太卡了，点一下要转 3 秒钟圈圈！” 当您打开 Prometheus 大盘，发现 `like-api` 的 P99 耗时确实飙到了 3 秒。 您去查 Logger 日志，发现根本没有报错，代码是完美执行成功的。

此时，面对 `like-api -> like-rpc -> Redis -> MySQL -> Kafka` 这样一条长长的微服务调用链，您陷入了沉思：**这 3 秒钟，到底是被谁吃掉的？是查数据库遇到慢 SQL 了？是 Redis 抽风了？还是发 Kafka 消息时网络阻塞了？**

这就是微服务拆分带来的最大痛点——**排障黑盒化**。

### 一、 为什么要引入 Jaeger？

为了打破这个黑盒，我们需要微服务可观测性的最后一块拼图：**全链路追踪（Distributed Tracing）**，而 Jaeger 便是其中的集大成者。

如果说 Prometheus 是看宏观大盘的“高空预警雷达”，Logger 是记录案发现场细节的“行车记录仪”，那么 Jaeger 就是**追踪单一请求轨迹的“微观单兵 GPS”**。

引入 Jaeger 后，系统会获得以下三大颠覆性的收益：

1. **打破服务孤岛（TraceID 贯穿全链路）**：当一个 HTTP 请求打到网关时，Jaeger 会自动为它生成一个全局唯一的 `TraceID`。这个 ID 会像一根隐形的线，穿透所有的 RPC 服务、数据库、缓存和消息队列。
2. **精准定位耗时“内鬼”（可视化时间轴）**：在 Jaeger UI 面板上，这 3 秒钟的耗时会被画成一张极其直观的“瀑布流”图。您一眼就能看穿：API 耗时 10ms，RPC 耗时 20ms，**往 Kafka 发消息耗时了整整 2970ms！** 破案十分轻松。
3. **厘清跨部门责任边界（跨服务拓扑图）**：通过分析海量的链路数据，Jaeger 能自动画出微服务之间的依赖拓扑图。在故障发生时，谁是上游谁是下游，故障是从哪个节点开始蔓延的，一切铁证如山，彻底终结跨团队扯皮。

### 二、 极简接入：在 go-zero 中开启 Telemetry

讲清楚了为什么需要它，我们再来看如何引入。

得益于 `go-zero` 框架在底层对 `OpenTelemetry`（云原生时代链路追踪的统一标准）的深度集成，我们开启链路追踪的成本低得令人发指——**在绝大多数纯网络请求场景下，我们甚至不需要改动一行代码，只需要在 `.yaml` 配置文件中开启即可。**

以我们的点赞系统网关 (`like-api`) 为例，只需在配置文件中加入 `Telemetry` 模块：

```yaml
Telemetry:
  Name: like-api                # 【极其重要】服务名称，将在 Jaeger 面板上作为区分标识
  Endpoint: localhost:34317     # 上报地址：OpenTelemetry Collector 或 Jaeger Agent 的 gRPC 接收端口
  Sampler: 1.0                  # 采样率：1.0 表示 100% 全量采集（通常用于开发/测试环境）。在 QPS 极高的生产环境，为了节省存储，通常会调小（如 0.01 表示只采集 1% 的样本）
  Batcher: otlpgrpc             # 上报协议：使用行业标准的 OTLP gRPC 协议批量发送链路数据，性能极佳
```

仅仅加上这四行配置，重启服务后，`go-zero` 底层的 HTTP 中间件就会自动开始拦截所有的网络请求，生成 TraceID，并默默地将这些耗时数据打包发送给后端的 Jaeger 收集器了！

*(注：同理，如果是 `like-rpc` 服务，只需将 yaml 中的 Name 改为 `like-rpc`，其他配置保持一致即可。)*

### 三、 链路追踪的“三层铁律”：到底哪里该手写代码？

配置好 YAML 文件后，很多开发者会陷入两个极端的误区：要么一行代码都不写，结果排障时对着一堆干瘪的 URL 抓瞎；要么在每一个函数里疯狂手写 `tracer.Start()`，导致 Jaeger 面板变成了一个毫无重点的俄罗斯套娃。

在真正的千万级高并发大厂实践中，我们在代码层面对待 Jaeger Span 的态度有着极其严苛的界限。决定要不要手写 `tracer.Start()`，本质上是问自己三个灵魂问题：**“链路断了吗？”、“业务重吗？”、“并发高吗？”**

总结起来，就是以下三大实战铁律：

#### 第一层：纯享框架红利（绝对零侵入）

**适用场景：** API 网关层（BFF）、基础的微服务间 RPC 调用入口。

在 `go-zero` 等现代云原生框架中，API 层的作用仅仅是路由分发和参数校验。当用户的 HTTP 请求打进来时，底层的 `tracingHandler` 会自动为您生成一个全局唯一的 TraceID，并创建一个 Root Span。

**实战做法：** 一行追踪代码都不要写！您唯一要做的，就是像接力赛一样，老老实实地把 `context.Context`（通常是 `l.ctx`）作为第一个参数，传递给下游的 RPC 客户端。千万不要在 API 层自作聪明地新建 Span，那只会破坏框架自动生成的完美时间轴，并造成严重的业务代码耦合。

------

#### 第二层：注入业务灵魂（只加属性，不建 Span）

**适用场景：** **高并发** C 端 RPC 逻辑层（如点赞、获取点赞数等）。这些接口 QPS 极高（几万到几十万），逻辑相对简单。

这是绝大多数核心业务代码所在的地方。前端传过来的 `ctx` 已经带了 Span，底层如果使用了 GORM 或 go-zero 的 Redis 客户端，它们也自带了 OTel 插件来记录耗时。**因此，我们依然不需要（也不能）新建 Span。** 因为 C 端流量太大了，每新建一个 Span，都会极大浪费微服务的 CPU 算力，并撑爆后端的 Elasticsearch 存储。

但是，如果不加干预，Jaeger 上显示的永远只是冷冰冰的 `/pb.LikeService/LikeAction`。故障发生时，我们根本不知道是哪个用户触发了卡顿。我们需要在这一层进行**高维业务注入**和**精准染红**。

**实战案例（以 C 端获取点赞状态为例）：**

```go
func (l *GetLikeStateLogic) GetLikeState(in *pb.GetLikeStateReq) (*pb.GetLikeStateResp, error) {
    // 1. 提取框架自动生成的当前 Span (不新建)
    span := trace.SpanFromContext(l.ctx)
    
    // 2. 注入上帝视角：记录业务主键和查询规模
    span.SetAttributes(
        attribute.Int64("query.user_id", in.UserId),
        attribute.String("query.target_type", in.TargetType),
        attribute.Int("query.batch_size", len(in.TargetIds)),
    )

    // ... 业务逻辑 ...

    if misses > 0 {
        // 3. 注入黄金性能指标：缓存穿透情况
        span.SetAttributes(attribute.Int("cache.misses", misses))
        
        // 调用底层数据库时，必须传入 l.ctx！
        // 否则 GORM 无法获取 TraceID，这条 DB 慢 SQL 将在 Jaeger 上彻底隐身！
        dbRespMap, dbErr := l.svcCtx.LikeModel.GetUserBatchLikeState(l.ctx, in.UserId, in.TargetType, missingIds)
        if dbErr != nil {
            // 4. 染红关键错误：让底层系统级报错在 Jaeger 面板上极其醒目
            span.RecordError(dbErr)
            logger.LogBusinessErr(l.ctx, errmsg.ErrorDbSelect, dbErr)
            return nil, dbErr
        }
    }
    return resp, nil
}
```

------

#### 第三层：自定义 Span 的普适纲领（必须手写的两类场景）

只要满足“**高价值重审计**”或“**脱离前端触发线**”，就必须果断使用 `tracer.Start()` 创建新 Span。这分为两种截然不同的流派：

**流派 A：低并发 B 端核心链路（精细审计流 -> 新建子 Span）**

- **特征：** 后台管理员封号 (`BanUser`)、支付创单、库存扣减。QPS 较低，但业务价值极高，审计要求极其严苛。
- **做法：** 必须新建“子 Span”。使用 `tracer.Start(l.ctx, "Business-Action-Name")`。传入的是当前的 `l.ctx`。
- **原因：** 框架自动生成的 Span 包含了网络传输的耗时。而在 B 端核心逻辑中，我们极度关心“纯业务代码”到底执行了多少毫秒。用一个明确命名的业务 Child Span 把底层动作包裹起来，能在 Jaeger 上形成一个极度清晰的树状审计结构。

**实战案例（B 端封禁用户）：**

```go
func (l *BanUserLogic) BanUser(in *pb.BanUserReq) (*pb.BanUserResp, error) {
    // 建立精细化的业务子 Span，精准剥离网络耗时
    tracer := otel.Tracer("admin-rpc")
    ctx, span := tracer.Start(l.ctx, "Action-Admin-BanUser")
    defer span.End()
    
    // 注入审计级别的强业务标签
    span.SetAttributes(
        attribute.Int64("audit.operator_id", in.OperatorId),
        attribute.Int64("audit.target_uid", in.Uid),
    )
    
    err := l.svcCtx.AdminModel.UpdateUserStatusByUid(ctx, in.Uid, 1)
    if err != nil {
        span.RecordError(err)
        // ... 错误处理 ...
    }
    // ...
}
```

**流派 B：跨越异步鸿沟（断点接骨流 -> 新建根 Span）**

- **特征：** MQ 消费者、定时任务（Cron）、独立后台 Goroutine。
- **做法：** 必须新建“根 Span”。使用 `tracer.Start(context.Background(), "Task-Name")`。传入的是 `Background`。
- **原因：** 这里的 `l.ctx` 要么压根不存在（如 Cron），要么无法跨越协程安全传递（如 `BulkExecutor` 异步批量落库）。我们必须强行开天辟地，自造一个起源，否则底层的 DB 落库操作就会变成永远查不到的黑盒。

**实战案例（Kafka 批处理的断点接骨）：**

```go
func (l *LikeUpdateService) flush(items []any) {
    if len(items) == 0 {
        return
    }
    
    // 1. 重新开天辟地：以 Background 为起点，手动创建一个全新的 Root Span
    tracer := otel.Tracer("like-mq")
    ctx, span := tracer.Start(context.Background(), "Flush-Like-Batch")
    defer span.End() // 确保函数结束时结算总耗时
    
    // 2. 注入批处理特征
    span.SetAttributes(attribute.Int("batch.size", len(items)))

    // ... 解析 items 组装 records ...

    // 3.将这个的全新 ctx 传给底层的 GORM
    err := l.svcCtx.LikeRecordModel.BatchUpsert(ctx, records)
    if err != nil {
        span.RecordError(err) // 染红落库失败
        logger.LogBusinessErr(ctx, errmsg.ErrorDbSelect, err)
        return
    }
    // ...
}
```

### 四、总结与升华：让微服务从“黑盒”走向“全息投影”

至此，我们的《微服务可观测性三部曲》已经全部连载完毕。我们以一个千万级高并发的点赞系统为切入点，一步步为其披上了坚不可摧的铠甲：

1. **首部曲（Logger 日志体系）**：我们将满天飞的普通文本日志，升级成了带有 `TraceID` 和多维业务标签的 JSON 结构化日志，为排障留下了最精准的“行车记录仪”。
2. **二部曲（Metrics 监控大盘）**：我们利用 Prometheus 与 Grafana，不仅监控了接口的 P99 耗时和 Redis 缓存命中率，甚至从业务维度打造了防刷风控的“高空预警雷达”。
3. **终结篇（Tracing 链路追踪）**：我们克制地使用 Jaeger，在 API 层享受框架红利，在 RPC 层注入业务灵魂，在 MQ 异步批处理层手动“断点接骨”，打通了微服务的五脏六腑。

**架构师的终极底牌：LMT 联动破局**

在这套体系大成之后，当海啸般的流量打垮了线上的某个节点，您再也不需要在深夜里对着 SSH 终端满头大汗地 `grep` 日志了。您拥有的是一套极其优雅的**“大厂标准排障 SOP（标准作业程序）”**：

- **第一步（发现问题）**：钉钉/飞书机器人在深夜疯狂报警，您打开 **Grafana (Metrics)**，发现大盘上 `GetLikeList` 接口的红色线条突然飙升，错误率暴涨。—— **（知道“出事了”）**
- **第二步（定位边界）**：您顺手打开 **Jaeger (Tracing)** 面板，找出一条飘红的慢请求链路。瀑布图清晰地显示：API 耗时 5ms，RPC 耗时 10ms，但是底层的 MySQL `GetTargetLikerList` Span 耗时整整 2000ms！—— **（知道“死在哪了”）**
- **第三步（锁定元凶）**：您复制这个 Span 上的 `TraceID`，扔进 **Kibana/ES (Logging)** 搜索框。瞬间，系统过滤出了这条请求所有的结构化日志。最底下赫然躺着一条 `logger.LogBusinessErr`：“发现缺少时间戳的脏数据，记录ID: 8848”。—— **（知道“为什么死”）**

这就是微服务可观测性三剑客（Logger, Metrics, Tracing）合璧后的无上威力！

**写在最后：敬畏生产，掌控复杂**

很多人以为高并发架构就是用 Redis Lua 挡一挡、用 Kafka 削峰填谷、用 Go 语言开几个协程。但真正经历过生产环境毒打的架构师都知道：**代码跑得快只是基本功，在雪崩来临时能“看清战局”并从容按下“降级开关”，才是真正的内功。**

可观测性不是毫无节制地打日志和建 Span，而是一门**关于“控制系统复杂度”的妥协艺术**。希望这三部曲能帮您在微服务的深水区里，少踩几个坑，多几分掌控全局的底气。