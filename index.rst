.. NSQ实现和设计 documentation master file, created by
   sphinx-quickstart on Tue Aug  5 10:43:31 2014.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

NSQ实现和设计
==========================================
NSQ是一个实时的分布式消息平台。它的设计目标是为在多台计算机上运行的松散服务提供一个现代化的基础设施骨架。
这篇文章介绍了基于go语言的NSQ的内部架构，它能够为高吞吐量的网络服务器带来 性能的优化，稳定性和鲁棒性

Contents:

介绍
-----------
.. toctree::
   :maxdepth: 2

   overview/quick_start.rst
   overview/features.rst
   overview/design.rst
   overview/internal.rst
   overview/performance.rst
   overview/faq.rst


组件
------------
.. toctree::
   :maxdepth: 2

   components/nsqlookup
   components/nsqd

* nsqadmin
* utilities



客户端
------------
.. toctree::
   :maxdepth: 2

   client/tcp_protocol
   client/building_client_libraries


部署
------------
.. toctree::
   :maxdepth: 2

   deployment/production
   deployment/topology-patterns

* `installing`_
* `docker`_

.. _installing: http://nsq.io/deployment/installing.html
.. _docker: http://nsq.io/deployment/docker.html



源码分析
------------------
下面是SamuelYao阅读源码后的笔记。

.. toctree::
   :maxdepth: 2

   read/nsqlookupd




Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
