# nsq初探


have fun with nsq 🚀

## 什么是nsq

推荐看看这篇文章，开发nsq的项目组对nsq的想法

[我们是如何使用NSQ处理7500亿消息的](http://www.jfh.com/jfperiodical/article/1949?)



### 消息队列

nsq是一种MQ（消息队列），常见的MQ还有比如RabbitMQ、Kafka等等

**那什么是消息队列呢？**

消息队列其实就是将数据/请求存放入队列中，生产者在队列一头提供数据/请求，消费者在队列另一头消费数据/请求

![](https://i.loli.net/2020/12/26/VxpmnR2QqTN6XuC.png "消息队列原型")

**为什么要用消息队列呢？**

我们将好处概括为**解耦**、**异步**和**均流**

##### `解耦`

多个系统之间数据往往有交互，这就导致了一个系统的崩溃会影响到其他系统，系统需求变更又会导致彼此接口的变更，可见系统的独立性大大下降；引入消息队列的一大好处是解耦了多系统，增加了可靠性

![](https://i.loli.net/2020/12/26/otRUYmywa8eZxlq.png "系统解耦的例子")



##### `异步`

异步实际是将后续操作延缓了，因为将数据放到了MQ中，不必等待其他系统的响应就直接返回；这样增加了系统的响应速度，但是导致了返回结果的不可靠性（因为后续可能会导致操作失败呀）；所以这里我们需要注意应用异步有一定的适用场景

![](https://i.loli.net/2020/12/26/9zRquQV7Lnk4rSM.png "异步快速响应")



##### `均流`

系统数据量在时间上分布一般是不均匀的，比如可能在白天下午时段有洪水期，而在其他时段数据流极小；通过应用MQ可以实现负载的均衡

![](https://i.loli.net/2020/12/26/nQU7lKWm6Lu2zd9.png "负载均衡")

### nsq组成模块

NSQ由3个进程组成：

- nsqd: 接收，维护队列和分发消息给客户端的daemon进程
- nsqlookupd: 管理拓扑信息并提供最终一致性的发现服务
- nsqadmin: 用于实时监控集群运行并提供管理命令的管理网站平台。



### nsq docker部署

```bash
docker pull nsqio/nsq

docker run -d --name lookupd -p 4160:4160 -p 4161:4161 nsqio/nsq /nsqlookupd

docker run -d --name nsqd -p 4150:4150 -p 4151:4151 nsqio/nsq /nsqd --broadcast-address=149.28.73.98 --lookupd-tcp-address=149.28.73.98:4160

docker run -d --name nsqadmin -p 4171:4171 nsqio/nsq /nsqadmin  --lookupd-http-address=149.28.73.98:4161
```

部署完成后可以通过浏览器访问nsqadmin

![](https://i.loli.net/2020/12/26/vGlgxypTqCSmIbR.png "nsq nodes管理页面")
