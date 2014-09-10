TCP协议约定
=============
.. _tcp-protocol:

NSQ协议足够的简单，任何语言都可以实现其客户端。我们提供官方的python和golang客户端库。

nsqd进程监听一个可配置的TCP端口，接受客户端的连接。

连接后，客户端必须发送一个4字节的"magic"标识表示他们将沟通什么版本的协议（方便之后的升级）。

V2协议(4字节ASCII, [space][space][V][2])，是基于推模式的流协议。

认证后，客户端可以选择发送IDENTIFY命令提供自定义的元数据（比如，更多描述的标识），
并协商功能。为了得到消息，客户端必须 **SUB** 一个通道。

在订阅客户端被置于0的 **RDY** 状态, 这意味着没有信息将被发送到客户端。
当客户准备好接收消息,  会发生命令，更新其 **RDY** 状态到它准备处理的消息数，例如100。
不需要额外的命令，100条短信将被传递到客户端（递减服务器端对应的这个客户端的RDY计数）。

V2协议支持客户端心跳协议。默认每隔30秒， nsqd会发送 **_heartbeat_** 。
如果客户端处于空闲状态，就要回复 **NOP** 。 如果连续2次， **nsqd** 没有收到回复，判定超时，强行关闭客户端连接。
**IDENTIFY** 命令可以用来修改/禁用此行为。

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

* **msg_timeout**,  配置每个消息，nsqd交付给客户端的超时时间, 单位是millisecond

成功:::

    OK

失败： ::

    E_INVALID
    E_BAD_BODY


SUB
^^^^^^^^^
订阅特定的 topic/channel

::

    SUB <topic_name> <channel_name>\n

    <topic_name> - a valid string
    <channel_name> - a valid string (optionally having #ephemeral suffix)

成功返回: ::

    OK

失败返回: ::

    E_INVALID


PUB
^^^^^^^^^^
发布指定topic的消息

::

    PUB <topic_name>\n
    [ 4-byte size in bytes ][ N-byte binary data ]

    <topic_name> - a valid string

成功返回：::

    OK

失败返回: ::

    E_INVALID
    E_BAD_TOPIC
    E_BAD_MESSAGE
    E_PUB_FAILED


RDY
^^^^^^^
更新 **RDY** 状态, 说明你已经准备好接收消息。

注意： **--max-rdy-count** 配置最大值

::

    RDY <count>\n
    <count> - a string representation of integer N where 0 < N <= configured_max

成功没有返回包。

失败返回: ::

    E_INVALID


FIN
^^^^^^^^^
成功处理消息

::

    FIN <message_id>\n
    <message_id> - message id as 16-byte hex string

成功没有返回。

失败返回：::

   E_INVALID
   E_FIN_FAILED


REQ
^^^^^^^^^^
消息处理失败，需要重发

::

    REQ <message_id> <timeout>\n

    <message_id> - message id as 16-byte hex string
    <timeout> - a string representation of integer N where N <= configured max timeout
        0 is a special case that will not defer re-queueing

成功没有返回。

失败返回： ::

    E_INVALID
    E_REQ_FAILED


TOUCH
^^^^^^^^^
重置指定消息的超时时间

::

    TOUCH <message_id>\n

    <message_id> - the hex id of the message

成功没有返回。

失败返回: ::

    E_INVALID
    E_TOUCH_FAILED


CLS
^^^^^^^^^^
不再接受消息

::

    CLS\n

成功返回： ::

    CLOSE_WAIT

失败返回: ::

    E_INVALID

NOP
^^^^^^
No-op, 客户端回复nsqd的心跳包协议

::

    NOP\n


AUTH
^^^^^^
如果 **IDENTIFY** 回复提示 **auth_required=true**, 客户端就可以发送 **AUTH**  指令。
如果 **auth_required** 不存在或者为false, 客户端就不需要授权.


Data Format
^^^^^^^^^^^^^^^
nsqd 发给client的数据， 支持多种回复

::

    [x][x][x][x][x][x][x][x][x][x][x][x]...
    |  (int32) ||  (int32) || (binary)
    |  4-byte  ||  4-byte  || N-byte
    ------------------------------------...
        size     frame type     data

客户端可能受到以下几种回复

::

    FrameTypeResponse int32 = 0
    FrameTypeError    int32 = 1
    FrameTypeMessage  int32 = 2

消息格式如下

::

    [x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x][x]...
    |       (int64)        ||    ||      (hex string encoded in ASCII)           || (binary)
    |       8-byte         ||    ||                 16-byte                      || N-byte
    ------------------------------------------------------------------------------------------...
      nanosecond timestamp    ^^                   message ID                       message body
                             (uint16)
                              2-byte
                              attempts
