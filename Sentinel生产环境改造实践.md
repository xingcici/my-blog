# Sentinel生产环境改造实践

### 引言
随着业务的发展，公司的服务需要对某些接口进行限流或者根据某些属性对请求进行限制。之前的方式是将黑白名单存储在数据库或者配置中心，无法较快和方便得对请求进行限制。公司安排平台组进行相关的调研，调研的结果显示阿里开源的 Sentinel 最符合我们目前的需求。
根据文档，目前开源的版本v1.6.3无法在生产环境直接使用。根据公司目前的情况，我们只需要使用网关部分的功能，要进行以下几点改造。
1.监控数据的持久化。
2.网关流控规则和API组规则接入 Apollo 配置中心。
3.登录鉴权体系。
本文主要介绍在改造过程中的实践。

---


### 实践
#### Sentinel 网关限流架构
在实践开始之前，先了解一下 Sentinel 网关限流的架构。如下图所示。

![](https://pic.superbed.cn/item/5d65fb26451253d178f26bd3.png)

惠借准备将原先的 Zuul 1.x 网关升级成 Spring Cloud Gateway，本文就以Spring Cloud Gateway 为例，以下简称 SCG。SCG中需要加入sentinel适配SCG的jar包，并且在启动时加入一些配置参数。
1. SCG启动时会向 Dashboard 服务请求API组规则和流控规则，加入到本机的配置中。
2. 在请求经过SCG时会对请求进行统计和输出日志。
3. Dashboard 更新规则时会更新本地内存中的规则并推送至SCG。
4. Dashboard 会定时拉取SCG输出的日志的统计结果。

#### 监控数据持久化
在上述架构图中，集成 Sentinel SCG jar包后，Sentinel 会在 SCG的过滤器中加入一个 SentinelGatewayFilter。对 SCG 请求的拦截就由这个过滤器来实现。具体监控数据的生成和输出由一系列的 Slot 负责，具体原理这里不展开叙述，因为本文主要描述生产环境使用的改造，有兴趣的同学可以搜一下 Sentinel 时间窗口。
原先的监控数据由 Dashboard 收集后存储在 Map 中，并且只存储五分钟内的数据，在 Dashboard 重启后监控数据就消失，这在生产环境中肯定是不能接受的。由于我们公司本身也有ES服务，这些数据又是个时序数据，自然而然就想到可以放入ES中进行持久化。
首先新建 ElasticsearchMetricsRepository 来实现 MetricsRepository 接口类。分别实现其中的 save、saveAll、queryByAppAndResourceBetween 和 listResourcesOfApp。在 MetricController、 MetricFetcher 中把自动注入的 MetricsRepository 加上 `@Qualifier(value = "elasticsearchMetricsRepository")`。
还有另外一个点在于 Dashboard 查询监控数据进行展示时，开源的版本并不会将调用量为零的那段数据补齐，导致监控数据展示得非常奇怪。所以我们需要在查询监控数据接口对ES返回的数据进行补齐，让监控数据是连续的。
下图是改造完的监控。

![](https://pic.superbed.cn/item/5d65fb8a451253d178f2785a.png)


#### 网关流控规则和 API 规则接入Apollo配置中心
开源版本的 Sentinel 流控规则和API规则也都是内存态的，在 SCG 和 Dashboard 重启后也会消失。
我们原先的配置中心采用的就是 Apollo，Sentinel 刚好可以跟 Apollo 进行集成。实现 Dashboard 推送至 Apollo，Apollo 再推送至 SCG，SCG启动时也从 Apollo 进行加载。
##### SCG方面
跟普通应用接入 Apollo 一样，在 Apollo 建立应用。同时需要新增两个特殊的 namespace，分别为 flow_rule 、 api_group。在SCG中加入 Sentinel的 Apollo 依赖，同时配置一般的 Apollo 参数。
然后需要对 Sentinel 的流控规则和API组规则加载方式进行修改。在 SentinelGatewayConfiguration 配置中加入如下两段代码

![](https://pic.superbed.cn/item/5d65fb50451253d178f2710d.png)
后续加载规则分别从某 namespace 下的某 key 加载。

##### Dashboard方面
Dashboard 这边本身 Sentinel 已经把普通应用的流控规则从 Apollo 加载的代码已经写好放在 test 目录下，但是没有实现网关的代码。其实也很简单，Sentinel 的数据加载和推送的方式本身就提供了较好的扩展性。我们只需要分别实现 DynamicGatewayRuleProvider 和 DynamicGatewayRulePublisher。
这里需要注意的一个点是 SCG 向 Dashboard 注册的 appName 最好和它本身在 Apollo 的 appId 保持一致。这样 DynamicGatewayRuleProvider在加载数据的时候直接用 appName 作为 appId 向 Apollo 请求配置。加载流控规则代码如下图所示。

![](https://pic.superbed.cn/item/5d65fb50451253d178f2710d.png)

至于 DynamicGatewayRulePublisher 其实有两个过程，第一步是向Apollo更新数据，第二步是要求 Apollo 对该 namespace 进行发布。代码如下图所示。

![](https://pic.superbed.cn/item/5d65fbf6451253d178f284e4.png)

这里面涉及到的 Apollo 开放平台的授权之类的过程（这次也发现这是Apollo的一个非常强大的功能，实际上可以有很多的应用，特别是在自动化方面）请参考 Apollo 的文档。
在这样的一个过程后很多同学应该能很自然而然地想到是不是可以在 Sentinel 或者其他监控工具监控到某些条件后自动地对 Apollo 的中的限流配置进行修改。是的，这也应该是我们以后的目标，做到限流的自动化和智能化。
随后还需要新增 GatewayApiControllerV2，GatewayFlowRuleControllerV2，基本上就是把原有的GatewayApiController、GatewayFlowRuleController复制过来， 然后分别注入 ApolloProvider 和  ApolloPublisher。
Controller 要改动的地方主要有两个，第一个将获取规则的方式从向SCG请求改为向 Apollo 请求，第二个是新增和修改规则先更新到 Apollo 再让Apollo 推送到 SCG。再将前端页面请求的路径指向新的 Controller。
下图是获取API规则的代码改造前和改造后区别。ApiDefinitionProvider 就是 DynamicGatewayApiDefinitionProvider 的从 Apollo 获取 API 规则的实现。
![](https://puui.qpic.cn/fans_admin/0/3_1409075683_1567151625668/0)



此处还有个暗坑，一开始没注意到。本身Dashboard在新增规则时会生成一个ID，这个ID是个 AtomicLong, 随着Dashboard的重启自动就清空了，导致在重启后新增或修改规则会将ID=1的规则顶替。目前解决的办法是在新增和修改规则时加载目前的全部规则，然后在获取ID时获取当前全部规则的最大ID，从最大值开始递增。

#### 登录和鉴权
原先的登录和鉴权只提供了基础的登录功能且安全性很低。我们将其加入了公司的统一登录中心，实现了帐号密码及企业微信扫码登录和访问权限的控制。

---

### 尾声
Sentinel 的改造陆陆续续进行了半个多月，其中也遇到了不少了问题，基本上都得到了较好的解决。目前仅仅是生产环境能用的状态，还有很多功能尚未完善。在改造的过程中也对Sentinel的基本原理和架构有了较清晰的认识，为后续使用中解决问题提供了基础。