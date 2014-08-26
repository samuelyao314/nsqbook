TCP协议约定
=============
.. _tcp-protocol:

NSQ协议足够的简单，任何语言都实现其客户端。我们提供官方的python和golang client libraries。

nsqd进程监听一个可配置的TCP端口，接受客户端的连接。

连接后，客户端必须发送一个4字节的"magic"标识表示他们将沟通什么版本的协议（升级容易）。

V2 (4字节ASCII [space][space][V][2])，是对消费者基于推模式的流协议。
认证后，客户端可以选择发送IDENTIFY命令提供自定义的元数据（比如，举例来说，更具描述性的标识），
并协商功能。为了开始使用消息，客户端必须SUB到一个通道。

在订阅客户端被置于0的 **RDY** 状态, 这意味着没有信息将被发送到客户端。
当客户准备好接收消息,  会发生命令，更新其 **RDY** 状态到它准备处理的消息数，例如100。
不需要额外的命令，100条短信将被传递到客户端（递减服务器端对应的这个客户端的RDY计数）。

V2协议支持客户端心跳协议。时间间隔默认是30秒，可以配置。每过30秒， nsqd会发送 **_heartbeat_** 。
如果客户端处于空闲状态，就要发送 **NOP** 回复。 如果连续2次，nsqd没有收到回复，nsqd就会超时，
强行关闭客户端连接。 **IDENTIFY** 命令可以用来修改/禁用此行为。

.. note:: * 除非特殊说明，所有的整数都是网络序
          * 合法的topic和channel的名称, 只能是字符 **[.a-zA-z0-9_-]** , 并且长度满足>1, <= 64 


命令字
-------------------

IDENTIFY
^^^^^^^^^^^
在服务器上, 客户端更新元数据和协商功能 ::

    IDENTIFY\n
    [ 4-byte size in bytes ][ N-byte JSON data ]

json字段如下

* **short_id** 已经废弃，使用 **client_id**
* **long_id**  已经废弃，使用 **hostname**
* **client_id** , 唯一标示此客户端
* **hostname**, 主机名称
* **feature_negotiation** , bool值，提示此客户端是否支持特性协商。如果服务端有此能力，就返回支持的功能和元数据的JSON有效载荷。
* **heartbeat_interval** , 心跳包时间间隔，milliseconds作为单位 ::

    Valid range: `1000 <= heartbeat_interval <= configured_max` (`-1` disables heartbeats)
    `--max-heartbeat-interval` (nsqd flag) controls the max
    Defaults to `--client-timeout / 2`

* **output_buffer_size** , nsqd对此客户端的写缓冲区的大小。 ::

    Valid range: `64 <= output_buffer_size <= configured_max` (`-1` disables output buffering)
    `--max-output-buffer-size` (nsqd flag) controls the max
    Defaults to `16kb`

* **output_buffer_timeout** , 超时后，nsqd会自动此客户端的刷新写缓冲区数据。::

    Valid range: `1ms <= output_buffer_timeout <= configured_max` (`-1` disables timeouts)
    `--max-output-buffer-timeout` (nsqd flag) controls the max
    Defaults to `250ms`

    **Warning**: configuring clients with an extremely low (`< 25ms`) `output_buffer_timeout`
    has a significant effect on `nsqd` CPU usage (particularly with `> 50` clients connected).

    This is due to the current implementation relying on Go timers which are maintained by the Go
    runtime in a priority queue.  See the [commit message][043b79ac] in
    [pull request #236][pull_req_236] for more details.

* **tls_v1**, 开启TLS ::

    `--tls-cert` and `--tls-key` (nsqd flags) enable TLS and configure the server certificate
    If the server supports TLS it will reply `"tls_v1": true`
    The client should begin the TLS handshake immediately after reading the `IDENTIFY` response
    The server will respond `OK` after completing the TLS handshake

* **snappy**, 开启snappy压缩 ::

    `--snappy` (nsqd flag) enables support for this server side

    The client should expect an *additional*, snappy compressed `OK` response immediately
    after the `IDENTIFY` response.

    A client cannot enable both `snappy` and `deflate`.

* **deflate**, 开启deflate压缩 ::

  `--deflate` (nsqd flag) enables support for this server side

  The client should expect an *additional*, deflate compressed `OK` response immediately
  after the `IDENTIFY` response.

  A client cannot enable both `snappy` and `deflate`.

* **deflate_level**, 配置deflate压缩级别 ::

    --max-deflate-level (nsqd flag) configures the maximum allowed value
    Valid range: 1 <= deflate_level <= configured_max
    Higher values mean better compression but more CPU usage for nsqd.

* **sample_rate**, deliver a percentage of all messages received to this connection. ::

    Valid range: `0 <= sample_rate <= 99` (`0` disables sampling)
    Defaults to `0`

* **user_agent**, 字符串标示，参考HTTP代理 ::

    Default: `<client_library_name>/<version>`

* **msg_timeout**, 服务端交付给客户端的消息，超时时间

成功:::

    OK

失败： ::

    E_INVALID
    E_BAD_BODY


SUB
^^^^^^^^^


