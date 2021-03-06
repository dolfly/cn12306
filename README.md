cn12306
=======

现在还有不少人在讨论12306的设计，在这里写一个简单的设计思路

1. 网站不是为了解决高峰期票少人多的问题，争论里总讨论这个话题没意义
2. 排队机制不能到处套用，拿网游的常规做法来处理web不是很合适，应该最大限度提升系统的响应速度
3. 最好的方式是开票后10分钟内热门车次票就被订光了，抢到票的高高兴兴去付钱，没抢到的骂骂咧咧想其他途径，早死早超生
4. 响应速度上去后，人们自然不用各种外挂、脚本来刷票了，对减轻网站压力也有好处
5. 交易量没有那么夸张，昨天上海热线微博发的消息：长假期间9月28日至10月1日4天网络预售票达到40万张，约占全部预售车票量的40%，建议旅客尽快换取纸质车票。据此可以估计总的交易规模
6. 12306一些纯配置数据的查询也很慢，比如车次、时刻表、票价等。说明设计确实做得比较渣
7. 静态内容处理好cdn、缓存是比较简单的，配置类数据可以缓存在各个服务节点，或者加一个配置服务，此类数据的查询web server自己就可以处理，放到nginx都行，再配合缓存，基本没大的压力
8. 发邮件，发短信之类的多弄几个队列发就是了
9. 查询时有余票，下订单时却没票属于合理情况，没必要回避

现在看看交易的主要部分：余票查询、订票和订单支付。只关注服务本身，不涉及http cache server、http proxy server等。

假设全国共5000车次（实际4000多，有一部分在12306还查不到），每个车次定员2000人（从网上资料看，很多车型定员不满1000），停靠站不超过64站（6245是62站，实际上超过30站的很少），预售12天车票。

如果这些票全通过网络预订，那么估计每天的交易量<1kw，实际上现在电话，窗口还是主要途径

可以用int来表示车次和日期，实际上int16就够了，用一个int64来表示每个座位的预订状态。每个车次/日期用组合用一个单独的队列来存储，预计占用的空间为定员数*size(int64)再加一些其他数据，可以算出这些数据全放到内存中也占不了多大的地。这样车票的查询和预订就可以全部在内存中计算。

车票预定后有45分钟的支付时间，可以用redis来存储这部分数据，因为存储的时间短，再优化一下存储结构，占不了多少空间。

支付后数据进入传统的RDMS，在查看订单，退票、取票时会用到。这些数据相对而言查询量应该不大，transaction的粒度小、好控制，分区策略比较容易制定。各大RDMS应付这个访问量都不成问题。

用golang写了一个数据结构的原型，没怎么用golang的特有功能比如chan，这样可以方便的移植到其他语言。

原先查询和预订放在一起的，后来为了设计一个简单的master/slave结构，把查询和预订分来了。数据持久化用redis，接口简单，吞吐量也有保证。

这样整个服务被划分成若干组service set，每一组service set由master\slave\redis组成，负责若干车次/日期的查询和预订

查询的流程如下:

1. 根据配置分析起点/终点，确定车次，根据车次/日期确定调用service的地址  (web)

2. 并发调用这些地址  (web)

3. 在对应的队列查询符合条件的票数，最多返回10  (slave)

4. 组合查询结果，输出到浏览器  (web)



查询逻辑基本上是没有锁的，而且全部内存中计算，响应速度很快。 查询结果可以缓存10-30秒, 对业务没什么影响。利用pipleline也可以提升浏览器的响应速度 

预订的流程如下:

1. 根据配置分析起点/终点，确定车次，根据车次/日期确定调用service的地址（web）

2. 找到对应的队列，加锁  (master)

3. 找到可用的座位   (master)

4. 调用redis的HMSET修改状态   (master)

5. 修改内存中的数据，释放锁   (master)

6. 订单写入redis, 并返回web端 (master)

7. 同步座位状态 (slave)


还是有很大的优化空间的，比如悲观锁可以改成乐观锁，用cas保证数据同步; 优化座位扫描的起始点以减少扫描次数等。
只要网络部分处理的好，并发链接数和响应速度不是问题, 或者简单一点，直接用go.http提供rest接口性能也不错

如果master挂了, slave可以在很短的时间内切换为master，同时再另起一个服务作为新的salve

订单支付的流程如下

1. 验证支付结果，写日志，写队列
2. 移除redis中的数据
3. 通过队列写RDMS、发邮件、发短信

