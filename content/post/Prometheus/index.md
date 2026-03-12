---
title: Promethus可观测性中间件介绍
date: 2026-03-12
description: 简单介绍 Prometheus 在 C 端和 B 端的应用
#image: Prometheus.png
categories:
  - 后端
  - 可观测性
  - 微服务

tags:
  - Prometheus
  - 监控系统
  - metrics
  - 高并发
  - anti-abuse
  - 审计
  - 风险管控
  - MQ
  - Kafka
---

# 微服务可观测性实战：从 C 端高并发防刷到 B 端高危审计的 Prometheus 埋点指南

在微服务项目中，Prometheus 往往是可观测性体系里最先落地的一环。但真正把它用好，并不是简单地暴露一个 `/metrics` 接口，再把 Grafana 大盘配出来就结束了。

对于不同类型的系统，监控的重点其实完全不同：

- 面向**海量用户**的 C 端，更关注吞吐、延迟、缓存命中率以及恶意流量拦截
- 而面向**内部管理**的 B 端，虽然并发不高，却往往涉及更高风险的操作，因此更强调审计、责任边界和高危行为监控。

本文就结合项目实践，聊聊我在微服务中落地 Prometheus 埋点时的一些思路，重点围绕两个典型场景展开：

- **C 端：高并发场景下的性能观测与防刷监控**
- **B 端：高危操作场景下的审计、风控与责任边界划分**

------

## 一、先从配置开始：为 Prometheus 暴露独立监控端口

以点赞系统的 `like-api` 为例，我们首先需要在 `.yaml` 中开启 Prometheus 相关配置：

```yaml
DevServer:
  Enabled: true 		#开启开发者服务器，如果设为 false，则监控全部关闭。
  Host: 0.0.0.0 		#监控地址
  Port: 9099    		#服务端口，每个服务部分的端口都应该不同,同一个系统的api和rpc的端口号也不同
  MetricsPath: /metrics #指标暴露路径，约定俗成，建议不变
  HealthPath: /healthz  #健康探针路径，约定俗成，建议不变
  EnableMetrics: true   #监控采集开关
  EnablePprof: true     #性能剖析开关
```

这里有一个很容易被忽略的点：点赞 API 本身已经有业务监听端口（例如 `8080`），为什么还要额外开放一个 `9099` 端口给 Prometheus？

原因在于，业务端口和监控端口的职责并不相同。

- **业务端口**：对前端应用开放，承载实际的业务请求流量。
- **监控端口**：用于暴露 `/metrics`、`/healthz`，以及 `pprof` 等调试和观测能力。

将监控能力放到独立端口之后，运维可以在 Nginx、服务网关或云防火墙层面直接限制访问范围，仅允许内网中的 Prometheus 服务器进行拉取。这样既不影响监控采集，也能避免将性能指标和运行时信息暴露到公网。

这其实体现的是微服务里很重要的一条原则：**业务流量入口与运维观测入口尽量隔离。**

## 二、C 端场景：高并发系统中的监控重点是什么？

在面向海量用户的 C 端系统中，例如点赞、收藏、关注、评论这类高频交互服务，监控的核心目标通常是：

- **观察吞吐量与耗时表现**
- **评估缓存和异步削峰链路是否健康**
- **尽早发现重复请求、恶意刷接口或下游依赖异常**

落地到微服务分层后，我通常会把监控视角拆成三部分：**API 层、RPC 层和 MQ 消费层**。

------

### 1. API / RPC / MQ：三层监控边界并不相同

在 Prometheus 落地过程中，一个很关键的问题是：**哪些指标应该交给框架自动采集，哪些指标必须自己手写埋点？**

通常来说：

- **API 层与 RPC 层**
   这两层主要处理同步请求。对于请求量、响应耗时、状态码分布等基础系统指标，框架通常已经提供了默认采集能力，因此几乎不需要我们在每个接口里重复手写。
- **MQ 消费层**
   这一层主要处理异步任务、削峰填谷和批量落库等场景。由于消费逻辑与具体业务强相关，框架无法理解“这一批消息到底代表什么”，所以这里往往是**自定义业务埋点最有价值的地方**。

换句话说，**系统级指标尽量复用框架能力，业务级指标则必须自己定义。**

------

### 2. MQ 消费埋点：观察真实吞吐量与失败损失

以点赞消息异步落库为例。Kafka 消费端从消息队列拉取数据后，会做聚合，再批量写入数据库。这里最值得关注的，不只是消费函数执行了多少次，而是**实际成功落库了多少条**数据，抑或是**失败损失了多少条**业务记录。

因此，我们定义一个专门的 Counter：

```go
var (
    ConsumeLikeMsgCount = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "business", 			//命名空间,是当前业务的总体大盘
        Subsystem: "like_mq",  			//子系统名,这里是点赞系统消息队列
        Name:      "consume_total",  	// 指标的具体名称:消息总数
        Help:      "Kafka 点赞消息消费与落库状态统计",
    }, []string{"action", "result"}) 	// 通过action和result两个维度进行打标
)
```

这里的标签设计是关键：

- `action`：表示动作阶段，例如 `flush_db`
- `result`：表示执行结果，例如 `success` 或 `error`

注：在不同的Counter中，我们需要根据逻辑的不同，自定义标签设计，下文中也会有其他的示范。

在具体逻辑里，埋点方式如下：

```go
err := l.svcCtx.LikeRecordModel.BatchUpsert(context.Background(), items)
if err != nil {
    // 失败逻辑触发 metrics
    // 注意这里没有用 .Inc() ,而是用 .Add(len) .因为我们是批量操作,必须记录真实损失的数据条数.
    metrics.ConsumeLikeMsgCount.WithLabelValues("flush_db", "error").Add(float64(len(items)))
    //...
}
// 记录成功落库的批次数据总量
metrics.ConsumeLikeMsgCount.WithLabelValues("flush_db", "success").Add(float64(len(items)))
```

当我们将这个指标配置到 Grafana 大盘上后，会得到以下两种极其直观的战报：
1. **看吞吐量（成功线）**：观察 `result=success` 的增长斜率，我们能实时知道当前点赞系统每秒钟能把多少条数据安全刷入 MySQL。如果**斜率平缓**，说明 Kafka **有积压**；如果**斜率陡峭**，说明**削峰效果极佳**。
2. **定损与预警（失败线）**：一旦 `result=error` 的数字大于 0 并触发报警，说明大批量的点赞数据在最后一步（落库）失败了。结合 `logger` 打印的日志，我们能瞬间判断是数据库宕机了，还是连接池被打满了。


------

### 3. RPC 层：基础性能指标之外，更重要的是业务健康度

对于点赞 RPC 这样的核心服务，框架一般已经帮我们统计了请求量、耗时、状态码等基础指标。但这些信息往往只够判断服务有没有慢，不够判断业务是不是已经在悄悄出问题。

因此，在 RPC 层，我们更应该关注动作全生命周期以及缓存健康度指标：

例如在点赞系统中：

```go
// 1.点赞动作全生命周期监控
var LikeActionCount = prometheus.NewCounterVec(prometheus.CounterOpts{
    Namespace: "business",
    Subsystem: "like_rpc",
    Name:      "action_total",
    Help:      "Total number of like/dislike actions",
}, []string{"target_type", "action", "result"})
// 2.缓存健康度监控
var LikeQueryCacheCount = prometheus.NewCounterVec(prometheus.CounterOpts{
    Namespace: "business",
    Subsystem: "like_rpc",
    Name:      "query_cache_total",
    Help:      "Cache hit/miss counts for like queries",
}, []string{"target_type", "result"})
```

在具体逻辑中，`result` 标签不应该只区分 `success` / `error`。更有价值的做法，是把故障原因继续细分，例如：

- `redis_error` 表示 Redis 缓存出错
- `json_error`   表示 JSON 格式转换出错
- `kafka_error` 表示向 Kafka 发送消息出错
- `idempotent`   表示不符合幂等性

例如在点赞逻辑中，如果 Lua 脚本返回 `0`，说明用户触发了重复点赞，被幂等逻辑成功拦截：

```go
if resInt == 0 {
    metrics.LikeActionCount.WithLabelValues(in.TargetType, actionStr, "idempotent").Inc()
}
```

这类指标看起来不像“错误”，但其实非常重要。因为它能帮助我们判断：

- 是否是前端防抖失效，导致重复请求增多；
- 是否有脚本或爬虫在高频刷接口；
- 是否存在用户层面的异常操作行为。

也就是说，**有些最有价值的指标，不是记录“系统报错”，而是记录“系统虽然没报错，但已经开始承压或遭遇异常流量”。**

------

###  4. API 层：尽量薄，只关注快速失败与入口拦截

在 C 端系统中，API 层通常更适合作为一个相对轻量的网关层。它的重点不是承载复杂业务，而是：

- 完成鉴权
- 完成基础参数校验
- 尽早拦截无效请求，避免流量继续穿透到下游 RPC

因此，在 API 层的埋点中，我一般不会记录过多业务过程，而是聚焦**入口拦截**本身。

例如定义一个拒绝计数器：

```go
var (
    ApiRejectCount = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "business",
        Subsystem: "like_api",
        Name:      "reject_total",
        Help:      "Total requests rejected at the API gateway layer",
    }, []string{"route", "reason"})
)
```

在 Logic 层中，如果从 `Context` 提取 JWT 信息失败，就立即打点并返回：

```go
func (l *LikeActionLogic) LikeAction(req *types.LikeReq) (*types.LikeResp, error) {
    userId, ok := l.ctx.Value("userId").(json.Number)
    if !ok {
        // 打点记录,无效token
        metrics.ApiRejectCount.WithLabelValues("/like/likeaction", "token_invalid").Inc()
        //...
    }

    uid, err := userId.Int64()
    if err != nil {
        // 打点记录,token转换失败
        metrics.ApiRejectCount.WithLabelValues("/like/likeaction", "token_parse_error").Inc()
        //...
    }

    // 校验通过后，继续调用下游 RPC
}
```

这类埋点最大的价值在于：它记录的是**被挡在系统入口之外的异常请求**。如果某个时间段 `token_invalid` 或 `token_parse_error` 指标突然上升，就值得进一步排查是否有伪造 Token、重放请求或上游认证链路异常。

------

### 5. 小结：哪些指标交给框架，哪些指标必须手写？

在 Prometheus 落地中，很容易走两个极端：

- 要么什么都自己埋，业务代码里到处都是 `Inc()`
- 要么完全依赖框架，最后只能看到接口报错，却看不到具体业务哪里出了问题

真正的正确用法，是弄清在服务中，究竟什么是**核心业务（Business）**：

我们可以将微服务监控严格划分为两大阵营：

#### 第一阵营：白嫖框架的“系统级指标”（完全无需手写）

**底层逻辑**：现代框架和云原生环境已经通过**中间件（Middleware）、拦截器（Interceptor）以及探针（Agent）**帮我们把脏活累活干完了。

这部分指标通常遵循业界的 **RED 原则**（Rate 速率、Errors 错误、Duration 耗时）和 **USE 原则**（Utilization 使用率、Saturation 饱和度、Errors 错误），你不需要在业务代码里写哪怕一行 `Inc()`。

那么系统默认会帮你采集哪些指标呢？通常可以分为两类：

- **请求侧指标（网络与流量）**：
  - **QPS/TPS（吞吐量）**：任意 HTTP/RPC 接口每秒的请求数。
  - **Latency（延迟）**：接口响应耗时，尤其是 P90、P99 等长尾分位值。
  - **Error Rate（协议级错误率）**：HTTP 4xx/5xx 状态码比例，或 RPC 的框架级异常。
  - **Traffic（带宽大小）**：请求与响应的 Byte 大小。
- **运行时与基础设施（Runtime & OS）**：
  - CPU 使用率、内存占用、Goroutine/线程池存活数量、GC 停顿时间、数据库连接池饱和度。

#### 第二阵营：必须亲自动手的“业务级指标”（核心发力点）

**底层逻辑**：底层框架再牛，它也听不懂你的业务。它只知道 `/api/order` 被调用了一次，但它绝对不知道用户是在下单还是退款，更不知道这次扣款失败是因为余额不足还是微信接口超时。 **凡是与资金、核心链路流转、安全风控强相关的数据，必须由开发者亲手埋点。**

在 `Service/Logic` 层，强烈建议手写以下四类指标：

1. **业务里程碑计数（Business Milestones）**
   - **场景**：注册成功数、订单支付成功数、高价值道具掉落总数。
   - **价值**：这是给老板和产品经理看的大盘，也是评估系统“有没有真正在赚钱”的终极指标。
2. **精细化异常归因（Granular Error Tracing）**
   - **场景**：当系统对外返回“操作失败”时，通过 `Label`（标签）精准记录是 `db_timeout`（数据库超时）、`third_party_error`（第三方接口挂了）还是 `invalid_param`（参数非法）。
   - **价值**：告警触发时，研发能一秒钟定位到是哪个外部依赖出了问题，实现**故障的精准定位**。
3. **静默逻辑与安全防刷（Silent/Idempotent Intercepts）**
   - **场景**：由于前端连击触发的“防抖拦截”、触发 Lua 脚本的“防超卖拦截”、MQ 消费端的“幂等性拦截”。
   - **价值**：代码没报错，不代表业务没受攻击。这些“**异常但被成功防御**”的动作，是洞察黑客刷单、爬虫攻击或前端 Bug 的最强雷达。
4. **异步/批处理吞吐量（Async & Batch Throughput）**
   - **场景**：Kafka 等消息队列的消费者，或者定时任务的批量落库（如 `BatchInsert` 1000 条数据）。
   - **价值**：框架中间件通常只能记录“消费函数执行了 1 次”，但你需要手写 `Add(1000)` 来记录真实的业务数据吞吐量，以防发生隐性积压。

------

## 三、B 端场景：低并发不代表低风险

当我们将目光从**面向用户的 C 端**（例如点赞系统）转向**面向内部管理的 B 端**（例如管理员系统）时，系统的防守阵型必须发生彻底的改变。

很多开发者习惯了 C 端的写法，直接把代码复制到 B 端，这是架构设计上的大忌。两者在监控指标的诉求上有着本质的区别：

1. **流量特征与风险倒挂**：
   - **C端系统**：**高并发**、**大流量**。单个用户的报错通常只会影响个体，风险较低（通过框架统一监控 QPS 和错误率即可）。
   - **B端系统**：极低并发。但管理员的一个按钮，可能重置一万个用户的密码或发放百万积分。**单次调用的破坏力极大**。
2. **网关层的角色定位**：
   - **C端网关**：定位是“**安检员**”。只要检查 Token 没问题，闭着眼睛把请求扔给底层 RPC，绝不在网关层浪费 CPU 去做额外的状态码翻译，以追求极致的吞吐量。
   - **B端网关**：定位是“**翻译官与审计员（BFF）**”。它必须把底层冰冷的 RPC 错误转化为前端可读的业务错误，并且必须**留下极其严密的定损边界证据**，以应对极其严苛的内网风控审计。

------

###  1. B 端最值得手写埋点的四类场景

对于 B 端，我们手写 Metrics 埋点必须死死守住以下四条底线：

**BFF 层的定损边界划定**

- **普适策略**：B 端的 API 网关通常扮演 BFF（服务于前端的后端）角色，需要将底层冰冷的 RPC 错误翻译成前端可读的业务提示（如把 `NotFound` 翻译为“用户不存在”）。在这个翻译的过程中，我们必须手写打点（如 `rpc_error`）。
- **架构价值**：这是为了在内部跨部门协作中建立**绝对的免责边界**。一旦内部系统报错，网关层的这个打点能立刻证明 API 层的 Token 鉴权和参数校验已完美通过，是下游核心 RPC 业务逻辑抛出了异常，从而终结无休止的扯皮。

**零信任准入与防爆破雷达**

- **普适策略**：对于管理员的登录（Login）、注册、修改密码等身份校验接口，必须全量手写打点，并细分 `result` 标签（如 `success`、`pwd_error`、`account_locked`）。
- **架构价值**：普通业务报错是 Bug，但在准入接口，极短时间内的报错激增意味着**系统正在遭受暴力破解或撞库攻击**。手写这些指标，能让安全团队在内鬼或黑客破门而入之前，直接在网关层触发自动封禁。

**特权操作的“事中熔断”**

- **普适策略**：对于所有具备“写操作”的高危特权接口（如封禁用户、修改资金、发放全站优惠券），哪怕代码完美执行，没有报任何错，我们也**必须为其手写 `success` 打点**。
- **架构价值**：很多开发者认为高危操作只要写进 MySQL 审计日志就行了。但问题在于，审计日志更适合事后追查，而不是实时发现风险。譬如，管理员账号被盗，半夜用脚本疯狂执行重置密码，底层的框架监控只会显示接口 100% 成功。只有我们手写的特权 `success` 指标在非工作时间出现百倍暴增，系统才能在 1 分钟内拉响红色警报，实现**事中实时熔断**，避免灾难性的资损。

**审计视角的链路追踪注入（把业务主键刻进骨髓）**

- **普适策略**：在 B 端 RPC 执行核心逻辑前，不仅要打 Metrics，还要手动调取当前的 Trace Span，强行将业务关键信息（如 `operator_id` 操作人ID、`target_uid` 被操作人ID）作为 Attributes 注入到追踪链路中，这一部分将在 Jaeger 文章中详细讲解。
- **架构价值**：这是微服务监控的高维打击。Jaeger 不再仅仅是排查“接口为什么慢”的工具，而是变成了**可视化的 3D 审计利器**。老板问“谁动了我的配置”时，只需在 Jaeger 搜索业务 ID，操作人的整个调用时间线和路径将无处遁形。

------

### 2. RPC 层：高危操作必须有成功指标

例如在 `Admin` 这样的接口里，可以这样定义如下两个指标：

```go
var (
    // 一是管理员登录，登录作为身份验证的接口，是安全性的基石
    AdminLoginCount = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "business",
        Subsystem: "admin_rpc",
        Name:      "login_total",
        Help:      "Admin login attempts and results",
    }, []string{"result"})
	// 二是管理员操作监控，用于内部审计
    AdminActionCount = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "business",
        Subsystem: "admin_rpc",
        Name:      "high_risk_action_total",
        Help:      "Total number of high-risk admin actions",
    }, []string{"action", "result"})
)
```

在具体的高危操作，例如 `BanUser` 逻辑中：

```go
func (l *BanUserLogic) BanUser(in *pb.BanUserReq) (*pb.BanUserResp, error) {
    err := l.svcCtx.AdminModel.UpdateUserStatusByUid(l.ctx, in.Uid, 1)
    if err != nil {
        if err == model.ErrorNotFound {
            // 操作对象不存在
            // 这说明系统没坏，可能是输错了 UID，或者前端传参异常。
            metrics.AdminActionCount.WithLabelValues("ban", "user_not_found").Inc()
            //...
        }
		// 数据库宕机或死锁
        // 明确是DB产生的问题，不用怀疑代码逻辑。
        metrics.AdminActionCount.WithLabelValues("ban", "db_error").Inc()
        //...
    }
	// 成功也必须记录
    // 高危操作的 success 是事中熔断的触发器。例如半夜两点该指标激增，说明管理员账号已泄露，系统需立刻断网止损。
    metrics.AdminActionCount.WithLabelValues("ban", "success").Inc()
    //...
}
```

这里的重点不是“代码是否复杂”，而是**指标是否真正服务于风险识别**。
 如果某短时间 `ban + success` 的指标突然陡增，那就值得立刻报警，表示系统安全可能遭到了冲击，而不是等第二天再去翻数据库审计表。

------

### 3. API 层：记录边界，而不是重做业务

在 Admin API 层，我更倾向于把它当作一个 BFF：
 它不负责做核心业务决策，而是负责明确调用者身份和把下游错误翻译成前端可处理的结果两件事。

例如定义一个 API 拦截指标：

```go
var (
    AdminApiInterceptCount = prometheus.NewCounterVec(prometheus.CounterOpts{
        Namespace: "business",
        Subsystem: "admin_api",
        Name:      "intercept_total",
        Help:      "Total number of Admin API interceptions",
    }, []string{"path", "result"})
)
```

在具体逻辑里，我们需要区分到底是 rpc 部分出现了问题，还是：

```go
func (l *BanuserLogic) Banuser(req *types.BanUserReq) (resp *types.BanUserResp, code int) {
    var adminUid int64
    if uidNumber, ok := l.ctx.Value("uid").(json.Number); ok {
        adminUid, _ = uidNumber.Int64()
    }

    rpcReq := &pb.BanUserReq{
        Uid:        req.Uid,
        OperatorId: adminUid,
    }

    rpcResp, err := l.svcCtx.AdminRpc.BanUser(l.ctx, rpcReq)
    if err != nil {
        // API 网关拦截记录
        // 明确API鉴权已通过，是下游RPC出现了问题
        metrics.AdminApiInterceptCount.WithLabelValues("/admin/banuser", "rpc_error").Inc()
		
        //...
        }
    }
	// 成功记录
    metrics.AdminApiInterceptCount.WithLabelValues("/admin/banuser", "success").Inc()
	//..
}
```

这一层的意义主要在于：
 **它不是再做一遍业务，而是在接口边界上补齐身份、翻译错误并记录责任归因。**

------

## 四、最后总结：Prometheus 埋点不是越多越好，而是越准越好

总体来说，Prometheus 落地最容易犯的错误，不是不打点，而是什么都想打。

真正有效的监控，并不依赖埋点数量，而依赖你是否抓住了系统中最值得观察的那几个关键位置：

- 在 **C 端**，重点看吞吐量、缓存命中、幂等拦截和异步落库链路；
- 在 **B 端**，重点看登录准入、高危操作、错误归因和审计身份；
- 对于 **系统级指标**，尽量复用框架和运行时能力；
- 对于 **业务级指标**，必须结合具体链路手写埋点。

也正因为如此，我用下面这句话来解释我对Prometheus的理解：

**这个系统最脆弱的地方在哪里，最值得被持续盯住的地方又在哪里。**