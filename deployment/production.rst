生产环境的部署
================
虽然 **nsqd** 可以作为单独运行，但下面的内容假定你利用其分布式的特性。

有3个不同的二进制文件需要安装和运行。


nsqd
---------
nsqd 是个服务进程，负责接收，缓存，分发消息给客户端。

所有的配置都可以通过命令行选项进行控制。大多数情况下， 默认配置已经足够。
但是下面的配置项需要特别的注意：

**--mem-queue-size** 可以调整每个topic或者channel在内存中，排队的消息的个数。如果消息超过这个阀值，
会透明地写到磁盘上。文件的路径是 **--data-path**.

nsqd 需要通过配置 **--lookupd-tcp-address** 来寻找nsqlookupd。

拓扑结构上，我们建议nsqd和生产者运行在同台机器上。

nsqd可以配置推送数据到指定的 **statsd** . nsqd就会发生对应的 nsqd.*的统计数据到stats.


nsqlookupd
---------------
nsqlookupd是个服务进程，提供了运行时查找服务。消费者可以通过它找到特定的topic的nsqd.

nsqlookupd的状态不会持久化，也不需要和其他的nsqlookupd进行协调。

同时运行几个根据你的冗余要求。它们使用很少的资源，可以和其他服务进程混搭。我们的建议是运行3个。 



nsqadmin
----------------
nsqadmin是1个WEB服务进程，负责实时地观察和管理你的NSQ集群。
它通过nsqlookupd找到nsqd, 并且利用**graphite**出图。(nsqd需要开启statsd支持)

你只需要运行1个进程。为了从graphite显示图形，需要指定参数 **--graphite-url** .


监控
-------------
每个服务进程都提供HTTP接口，用来监控。 同样也可以用来实时调试：

::

    $ watch -n 0.5 "curl -s http://127.0.0.1:4151/stats"

通常情况下使用nsqadmin。
