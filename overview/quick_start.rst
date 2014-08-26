Quick Start
==============
下面的过程将会在你的机器上，启动1个NSQ集群，通过publishing, consuming, 保存消息到磁盘上

#. 根据文档进行安装
#. 打开1个终端，启动 nsqlookupd::

   $ nsqlookupd

#. 在另外1个终端，启动 nsqd::

   $ nsqd --lookupd-tcp-address=127.0.0.1:4160

#. 在另外1个终端，启动 nsqadmin::

   $ nsqadmin --lookupd-http-address=127.0.0.1:4161

#. 发布1个消息，在集群中创建了主题::

   $ curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'

#. 最后，启动 nsq_to_file::

   $ nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161

#. 发布更多的消息::

   $ curl -d 'hello world 2' 'http://127.0.0.1:4151/put?topic=test'
   $ curl -d 'hello world 3' 'http://127.0.0.1:4151/put?topic=test'

#. 为了确定是否正常，在浏览器中打开 http://127.0.0.1:4171/ ，通过nsqadmin UI，检查统计信息。
   同样，在/tmp目录检查日志文件test.*.log
