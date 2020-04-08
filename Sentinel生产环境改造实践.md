# Sentinel生产环境改造实践

### 引言

随着业务的发展，公司的服务需要对某些接口进行限流或者根据某些属性对请求进行限制。之前的方式是将黑白名单存储在数据库或者配置中心，无法较快和方便得对请求进行限制。公司安排平台组进行相关的调研，调研的结果显示阿里开源的sentinel最符合我们目前的需求。

根据文档，目前开源的版本v1.6.3无法在生产环境直接使用。根据公司目前的情况，我们只需要使用网关部分的功能，要进行以下几点改造。

1.监控数据的持久化。

2.网关流控规则和api规则接入apollo配置中心。

3.登录鉴权体系。

### 实践

#### sentinel网关限流架构

在实践开始之前，先了解一下sentinel网关限流的架构。如下图所示。

![](https://pic.downk.cc/item/5e8d606a504f4bcb04f09ca8.png)

惠借准备将原先的 Zuul 1.x 网关升级成 Spring Cloud Gateway，本文就以Spring Cloud Gateway 为例，以下简称 SCG。SCG中需要加入sentinel适配SCG的jar包，并且在启动时加入一些配置参数。

1.SCG启动时会向dashboard服务请求API组规则和流控规则，加入到本机的配置中。

2.在请求经过SCG时会对请求进行统计和输出日志。

3.Dashboard更新规则时会更新本地内存中的规则并推送至SCG。

4.Dashboard会定时拉取SCG输出的日志的统计结果。

#### 监控数据持久化

在上述架构图中，集成sentinel SCG jar包后，sentinel 会在 SCG的过滤器中加入一个 SentinelGatewayFilter。对SCG请求的拦截就由这个过滤器来实现。具体监控数据的生成和输出由一系列的Slot负责，具体原理这里不展开叙述，因为本文主要描述生产环境使用的改造，有兴趣的同学可以搜一下 sentinel时间窗口。

原先的监控数据由Dashboard收集后存储在Map中，并且只存储五分钟内的数据，在Dashboard重启后监控数据就消失，这在生产环境中肯定是不能接受的。由于我们公司本身也有ES服务，这些数据又是个时序数据，自然而然就想到可以放入ES中进行持久化。

首先新建 ElasticsearchMetricsRepository 来实现 MetricsRepository 接口类。分别实现其中的 save、saveAll、queryByAppAndResourceBetween和listResourcesOfApp。在 MetricController、 MetricFetcher中把自动注入的 MetricsRepository 加上 `@Qualifier(value = "elasticsearchMetricsRepository")`。

还有另外一个点在于Dashboard查询监控数据进行展示时，开源的版本并不会将调用量为零的那段数据补齐，导致监控数据展示得非常奇怪。所以我们需要在查询监控数据接口对ES返回的数据进行补齐，让监控数据是连续的。

#### 网关流控规则和API规则接入apollo配置中心

开源版本的sentinel流控规则和API规则也都是内存态的，在dashboard重启后也会消失。

在惠借我们原先的配置中心采用的就是apollo，sentinel刚好可以跟apollo进行集成。dashboard推送至apollo，apollo再推送至SCG，SCG启动时也从apollo进行加载，实现了规则的持久化。

##### SCG方面

跟普通应用接入Apollo一样

##### Dashboard方面