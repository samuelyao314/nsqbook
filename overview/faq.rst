FAQ
=======

部署
-----------
* What is the recommended topology for nsqd?  ::

    我们强烈建议nsqd与任何产生消息的服务一起运行。
    nsqd是一个占用有限内存的相对轻量级进程，这使得它非常适合于“playing nice with others”。
    这种模式构建消息流有利于消费问题，而不是一个生产问题。
    另一个好处是，它本质上形成一个独立，分片，筒仓的数据对于一个给定主机上的topic。
    注意：这不是一个绝对的要求，虽然，它只是简单的（见下面的问题）。

* Why can't nsqlookupd be used by producers to find where to publish to?  ::

    NSQ提供了消费者发现模型，降低了消息者找到所需 topic 之前的前期配置负担。
    然而，它不提供任何手段来解决一个服务应发布到哪的问题。
    这是一个鸡和蛋的问题，topic 不会在第一个发布之前存在。

    通过协同定位nsqd（见上面的问题），你完全回避这个问题（您的服务只是发布到本地nsqd），并允许NSQ运行时发现系统自然工作。

* I just want to use nsqd as a work queue on a single node, is that a suitable use case?  ::

    是的，独立运行nsqd就好了。
    nsqlookupd 有利于在更大的分布式环境。

* How many nsqlookupd should I run?  ::

    通常只要几个，取决于你集群大小，nsqd节点和消费者的数目，以及您需要的容错性。
    3或5个就能够为部署上百的主机与数千的消费者工作很好。
    nsqlookupd节点并没有需要协调来回答查询。集群中的元数据是最终一致。


发布消息
-----------
* Do I need a client library to publish messages?::

    事实上，绝大多数NSQ部署使用HTTP进行发布。

* Why force a client to handle responses to the TCP protocol’s PUB and MPUB commands?  ::

    我们认为NSQ的默认模式应优先考虑安全性，我们希望协议是简单和一致。

* When can a PUB or MPUB fail? ::

    1. topic名称格式不正确（以字符/长度的限制）。请参阅topic和channel名称规范。
    2. 消息太大（这个限制是作为参数传递给nsqd）
    3. topic正在被删除中
    4. nsqd正在退出中
    5. 发布时, 任何客户端连接相关的故障

    (1) and (2) 应当考虑的编程错误. (3) and (4) 是罕见的， (5) is a natural part of any TCP based protocol.

* How can I mitigate scenario (3) above::

    删除topic是一个相对较少的操作。 If you need to delete a topic, orchestrate the timing such that publishes
    eliciting topic creations will never be performed until a sufficient amount of time has elapsed since deletion.


设计与理念
---------------
* How do you recommend naming topics and channels?::

    一个 topic name 应该描述流里的数据。
    一个channel name 应该描述消费者执行的工作。

    For example, good topic names are encodes, decodes, api_requests, page_views and good channel names are archive,
    analytics_increment, spam_analysis.

* Are there any limitations to the number of topics and channels a single nsqd can support?  ::

    没有内置强加的限制。它仅受限于运行nsqd 的主机内存和CPU。

* How are new topics announced to the cluster?::

    在topic上的第一个PUB或SUB命令将在nsqd上创建 topic。然后topic的元数据将传播到nsqlookupd。
    其它readers将通过定期查询nsqlookupd发现这个topic

* Can NSQ do RPC?  ::

    是的，这是可以的，但NSQ最初并没有设计这个用例。


pynsq
-----------------
#. Why are you forcing me to use Tornado?::

    `pynsq` was originally intended as a consumer-focussed library and consuming
    the NSQ protocol is far simpler to implement in Python with an asynchronous
    framework (notably due to NSQ's push oriented protocol).

    Tornado's API is simple and performs reasonably well.

#. Is the Tornado IOLoop required to publish::

    No, `nsqd` exposes HTTP endpoints (`/pub` and `/mpub`) for dead-simple
    programming language agnostic publishing.

    If you're worried about the overhead of HTTP, don't be.  Additionally,
    `/mpub` reduces the overhead of HTTP by publishing in bulk (atomically!).

#. When would I want to use Writer then?::

    Use 'Writer' and don't specify a callback to the publish method.

    this *only* has the effect of simpler client side code, behind the scenes
    'pynsq' still has to handle the response from `nsqd` (i.e. there isn't
    a *performance* benefit of doing this).
