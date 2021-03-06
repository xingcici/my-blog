# 针对流量的 AB TEST 的一点思考

之前公司提了个需求，不同比例的用户导向不同的落地页，统计转化效率。我调研了一些业界的方案后有了一些自己的思考。

#### 在阅读下述方案之前 我们需要明确几个定义

* AB TEST 方案分版本，不同的 AB TEST 方案拥有不同的标识，例如 ABTEST_2020_01_01。

* 某个确定的 AB TEST 版本下，会有不同的分组，每个分组也有确定的标识，例如 GROUP_A。下文中的 AB TEST 桶 ，指的就是不同的分组。

* 流量进入后会被打上一个或者多个 AB TEST 版本标识及 AB TEST 桶标识。

#### 单纯针对该需求的方案 

目标: 相同用户返回相同落地页 不同用户根据比例返回不同落地页 跟踪不同落地页转化效率 

方案: 根据ABTEST版本 建立唯一标识, 请求跳转链接时根据用的IP或者CookieId进行HASH，根据HASH命中区间( AB TEST 桶)返回不同的落地页，返回结果时带上TEST版本标识，转化效率的跟踪在H5进行。同时留有后门方便指定用户分入不同的区间中。

#### 更通用的方案(也只是简略描述 具体还要根据详细的需求来细化) 

目标: 根据不同用户特征返回不同结果，跟踪不同返回结果的转化效率。 

* 思路A 流量完全分层 所有流量可以循环利用 一个请求可以用于多个 AB TEST

方案:

1. 设置要针对的特征，圈定范围用户。比如 用户的IP 用户的 OPENID

2. 设置 AB TEST 方案，每个AB TEST方案都有唯一标识。再在 AB TEST 方案中，根据不同特征或者比例设置不同的 AB TEST 桶，每个 AB TEST 桶都有一个唯一标识用于跟踪，每个 AB TEST 桶可以设置一定比例的流量。 

3. 在网关设置过滤器，对进入的符合要求的流量特征进行HASH(同时也可以针对特定的用户标识强行进行分组,方便测试)，打上进行中的 AB TEST 方案的标识， 再根据该 AB TEST 方案打上 AB TEST 桶的标识。再对该流量进行下一次循环，查询是否还有 AB TEST 方案可以利用它，有的话则继续进行标记。符合最后的标记可能是类似于TESTA_GROUP1|TESTB_GROUP2。

4. 流量到服务的时候，根据标识返回不同的结果，在返回的同时带上 AB TEST 方案的标识。

存在的问题：当不同的 AB TEST 方案对相同页面做实验时就会出现问题。

* 思路B 流量预先分桶 一个请求只能用于一个 AB TEST 

方案:

1. 设置要针对的特征，圈定范围用户。比如 用户的IP 用户的 OPENID。

2. 设置实验方案，每个 AB TEST 方案都有每一标识。再在 AB TEST 方案中，根据不同特征或者比例设置不同的 AB TEST 桶，每个 AB TEST 桶都有一个唯一标识用于跟踪。 

3. 在网关设置过滤器，对圈定范围流量先随机进行HASH，按实验需要的流量大小随机分到不同的流量桶中，再在流量桶中实施具体的 AB TEST 方案， 再根据该 AB TEST 方案打上 AB TEST 桶的标识 例如 TESTA_GROUP1，随后直接进入服务端。 

4. 流量到服务的时候，根据标识返回不同的结果，在返回的同时带上 AB TEST 方案的标识。 

存在的问题：当 AB TEST 方案逐渐增加时，后来增加的 AB TEST 方案可能无法获取到足够的流量 

附图 



![图片](https://uploader.shimo.im/f/kVeqhjiJKlUAhBs2.png!thumbnail)

