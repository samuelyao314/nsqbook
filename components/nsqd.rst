nsqd
=========
**nsqd** 是接收，排队，传递消息给客户端的服务进程。

它可以独立地运行。但是通常运行在一个拥有nsqlookupd的集群中。在这种情况下，它会通知nsqlookupd，拥有的topic和channel。

它监听2个端口，1个tcp服务， 1个是http服务。可选地可以开启第3个https服务端口。


命令行选项
------------
::

  -auth-http-address=: <addr>:<port> to query auth server (may be given multiple times)
  -broadcast-address="": address that will be registered with lookupd (defaults to the OS hostname)
  -config="": path to config file
  -data-path="": path to store disk-backed messages
  -deflate=true: enable deflate feature negotiation (client compression)
  -e2e-processing-latency-percentile=: message processing time percentiles to keep track of (can be specified multiple times or comma separated, default none)
  -e2e-processing-latency-window-time=10m0s: calculate end to end latency quantiles for this duration of time (ie: 60s would only show quantile calculations from the past 60 seconds)
  -http-address="0.0.0.0:4151": <addr>:<port> to listen on for HTTP clients
  -https-address="": <addr>:<port> to listen on for HTTPS clients
  -lookupd-tcp-address=: lookupd TCP address (may be given multiple times)
  -max-body-size=5123840: maximum size of a single command body
  -max-bytes-per-file=104857600: number of bytes per diskqueue file before rolling
  -max-deflate-level=6: max deflate compression level a client can negotiate (> values == > nsqd CPU usage)
  -max-heartbeat-interval=1m0s: maximum client configurable duration of time between client heartbeats
  -max-message-size=1024768: (deprecated use --max-msg-size) maximum size of a single message in bytes
  -max-msg-size=1024768: maximum size of a single message in bytes
  -max-msg-timeout=15m0s: maximum duration before a message will timeout
  -max-output-buffer-size=65536: maximum client configurable size (in bytes) for a client output buffer
  -max-output-buffer-timeout=1s: maximum client configurable duration of time between flushing to a client
  -max-rdy-count=2500: maximum RDY count for a client
  -max-req-timeout=1h0m0s: maximum requeuing timeout for a message
  -mem-queue-size=10000: number of messages to keep in memory (per topic/channel)
  -msg-timeout="60s": duration to wait before auto-requeing a message
  -snappy=true: enable snappy feature negotiation (client compression)
  -statsd-address="": UDP <addr>:<port> of a statsd daemon for pushing stats
  -statsd-interval="60s": duration between pushing to statsd
  -statsd-mem-stats=true: toggle sending memory and GC stats to statsd
  -statsd-prefix="nsq.%s": prefix used for keys sent to statsd (%s for host replacement)
  -sync-every=2500: number of messages per diskqueue fsync
  -sync-timeout=2s: duration of time per diskqueue fsync
  -tcp-address="0.0.0.0:4150": <addr>:<port> to listen on for TCP clients
  -tls-cert="": path to certificate file
  -tls-client-auth-policy="": client certificate auth policy ('require' or 'require-verify')
  -tls-key="": path to private key file
  -tls-required=false: require TLS for client connections
  -tls-root-ca-file="": path to private certificate authority pem
  -verbose=false: enable verbose logging
  -version=false: print version string
  -worker-id=0: unique identifier (int) for this worker (will default to a hash of hostname)



HTTP API
-------------------

/pub
^^^^^^^
发布1个消息

::
    curl -d 'hello world 1' 'http://127.0.0.1:4151/put?topic=test'
    # 返回
    OK

处理函数， nsqd/http.go的doPUB


/mpub
^^^^^^^^
发布多个消息

处理函数， nsq/http.go的doMPUB


/stats
^^^^^^^^
获取运行状态信息


::

    curl  'http://127.0.0.1:4151/stats?format=json'

    # 返回
    {
    "status_code": 200,
    "status_txt": "OK",
    "data": {
        "version": "0.2.30",
        "topics": [
        {
            "topic_name": "bblog_change",
            "channels": [
            {
                "channel_name": "logcons",
                "depth": 0,
                "backend_depth": 0,
                "in_flight_count": 0,
                "deferred_count": 0,
                "message_count": 206465789,
                "requeue_count": 2542,
                "timeout_count": 2542,
                "clients": [
                {
                    "name": "pet_conf_dr_194_1_17",
                    "client_id": "pet_conf_dr_194_1_17",
                    "hostname": "pet_conf_dr_194_1_17",
                    "version": "V2",
                    "remote_address": "10.194.1.17:36704",
                    "state": 3,
                    "ready_count": 75,
                    "in_flight_count": 0,
                    "message_count": 638854,
                    "finish_count": 638854,
                    "requeue_count": 0,
                    "connect_ts": 1409556595,
                    "sample_rate": 0,
                    "deflate": false,
                    "snappy": false,
                    "user_agent": "go-nsq/1.0.0",
                    "tls": false,
                    "tls_cipher_suite": "",
                    "tls_version": "",
                    "tls_negotiated_protocol": "",
                    "tls_negotiated_protocol_is_mutual": false
                }
                ],
                "paused": false,
                "e2e_processing_latency": {
                "count": 0,
                "percentiles": null
                }
            }
            ],
            "depth": 0,
            "backend_depth": 0,
            "message_count": 206249153,
            "paused": false,
            "e2e_processing_latency": {
            "count": 0,
            "percentiles": null
            }
        }
        ]
    }
    }


处理函数, nsqd/http.go的doStats，调用nsqd/stats.go里GetStats,
返回的结果集TopicStats数组


ping
^^^^^^
测试连接性

::

    curl  'http://127.0.0.1:4151/ping'

    # 返回
    OK


/topic/create
^^^^^^^^^^^^^^
创建主题

::

    curl  'http://127.0.0.1:4151/topic/create?topic=test'

处理函数， nsqd/http.go的doCreateTopic


/topic/delete
^^^^^^^^^^^^^^^^^
删除主题和对应的channel, 会抛弃所有的消息，无论是在内存还是磁盘中

::

    curl  'http://127.0.0.1:4151/topic/delete?topic=test'
    # response
    {"status_code":200,"status_txt":"OK","data":null}

函数调用:

http.go:doDeleteTopic --> nsqd.go:DeleteExistingTopic


/topic/empty
^^^^^^^^^^^^^^^
删除主题，抛弃主题中的消息

::

    curl  'http://127.0.0.1:4151/topic/empty?topic=test'

函数调用:

http.go:doEmptyTopic --> topic.go:Empty


/topic/pause
^^^^^^^^^^^^^^
暂停主题消息流

::

    curl  'http://127.0.0.1:4151/topic/pause?topic=test'

函数调用:

http.go:doPauseTopic --> topic.go:Pause


/topic/unpause
^^^^^^^^^^^^^^
暂停主题消息流

::

    curl  'http://127.0.0.1:4151/topic/unpause?topic=test'

函数调用:

http.go:doPauseTopic --> topic.go:UnPause


/channel/create
^^^^^^^^^^^^^^^^^^
创建channel


/channel/delete
^^^^^^^^^^^^^^^^^^
删除channel


/channel/empty
^^^^^^^^^^^^^^^^^
清空channel


/channel/pause
^^^^^^^^^^^^^^^^
暂停channel里的消息流

/channel/unpause
^^^^^^^^^^^^^^^^^
恢复channel里的消息流



Debugging and Profiling
-----------------------------
**nsqd** 通过整合pprof提供性能优化接口。 参考nsqlookup部分.

TLS
-----------

AUTH
---------

End-to-End Processing Latency
-------------------------------

Statsd / Graphite Integration
---------------------------------



TCP API
--------------------
具体协议内请参考TCP协议约定。

nsqd启动后，监听TCP端口

::

    # nsqd.go#Main
    tcpListener, err := net.Listen("tcp", n.tcpAddr.String())
    if err != nil {
        log.Fatalf("FATAL: listen (%s) failed - %s", n.tcpAddr, err.Error())
    }
    n.tcpListener = tcpListener
    tcpServer := &tcpServer{context: context}
    n.waitGroup.Wrap(func() { util.TCPServer(n.tcpListener, tcpServer) })

    # util/tcp.go
    # nsqd/tcp.go#tcpServer实现了这个接口
    type TCPHandler interface {
        Handle(net.Conn)
    }

    func TCPServer(listener net.Listener, handler TCPHandler) {
        log.Printf("TCP: listening on %s", listener.Addr().String())

        for {
            clientConn, err := listener.Accept()
            if err != nil {
                if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
                    log.Printf("NOTICE: temporary Accept() failure - %s", err.Error())
                    runtime.Gosched()
                    continue
                }
                // theres no direct way to detect this error because it is not exposed
                if !strings.Contains(err.Error(), "use of closed network connection") {
                    log.Printf("ERROR: listener.Accept() - %s", err.Error())
                }
                break
            }
            # 每个客户端连接，都会有一个goroutine处理请求
            go handler.Handle(clientConn)
        }
        log.Printf("TCP: closing %s", listener.Addr().String())
    }

每个客户端请求，会启动单独的一个goroutine.  执行的入口函数是 nsqd/tcp.go#Handle.
客户端必须发送4字节的magic number, 如果不是V2，就直接判定为非法连接， 强行关闭。
成功后，就进入不停处理客户端请求的循环 nsq/protocol_v2.go#IOLoop,
会单独启动一个goroutine, 执行push消息的逻辑 protocol_v2.go#messagePump


SUB
^^^^^^
client向nsqd发送SUB指令，执行函数 protocol_v2.go#SUB

::
    # 如果没有，会自动创建
    topic := p.context.nsqd.GetTopic(topicName)
    # 如果没有，会自动创建
    channel := topic.GetChannel(channelName)

    # 关联client和channel
    channel.AddClient(client.ID, client)
    atomic.StoreInt32(&client.State, stateSubscribed)
    client.Channel = channel

    // update message pump
    client.SubEventChan <- channel

上面代码最后一行， 通知 protocol_v2.go#messagePump 所在的协程, 更新对应的状态

::

    select {
    ...
    case subChannel = <-subEventChan:
        // channel被赋值nil后，永久堵塞，所以不会再接受任何的SUB通知
        subEventChan = nil
    }


RDY
^^^^^^^
client向nsqd发送RDY, 通知，已准备好接受指定数量的消息;
执行函数 protocol_v2.go#RDY, 调用 client.SetReadyCount(count),

::

    # client_v2.go
    func (c *clientV2) SetReadyCount(count int64) {
        # ReadyCount表示当前已经准备好的消息，每次成功发送1个，计数减1
        atomic.StoreInt64(&c.ReadyCount, count)
        // LastReadyCount表示最近一次RDY请求的计数，只能由SetReadyCount重置
        atomic.StoreInt64(&c.LastReadyCount, count)
        c.tryUpdateReadyState()
    }

    func (c *clientV2) tryUpdateReadyState() {
        select {
        // 通知 protocol_v2.go#messagePump 所在的协程
        case c.ReadyStateChan <- 1:
        default:
        }
    }

tryUpdateReadyState触发select

::

    # protocol_v2.go#messagePump
    select {
    ...
    case <-client.ReadyStateChan:  // 什么事情也不做，只是进入下一次循环
    ...
    }

下一次循环, 进入第2个条件分支

::

    if subChannel == nil || !client.IsReadyForMessages() {
        ...
    } else if flushed {
        // last iteration we flushed...
        // do not select on the flusher ticker channel
        clientMsgChan = subChannel.clientMsgChan
        flusherChan = nil
    } else {
        ...
    }

正常情况下， 从channel收到消息，触发逻辑

::

    # protocol_v2.go#messagePump
    select {
    ...
    case msg, ok := <-clientMsgChan:
            if !ok {
                goto exit
            }

            if sampleRate > 0 && rand.Int31n(100) > sampleRate {
                continue
            }

            subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
            # ReadyCount计数减1， InFlightCount计数加1
            client.SendingMessage()
            err = p.SendMessage(client, msg, &buf)
            if err != nil {
                goto exit
            }
            flushed = false

    ...
    }

接着，就是等待client的 **FIN** 和 **REQ** 回复.


计数
^^^^^^^^^^^^^^^
clientV2有几个非常重要的计数

::

    MessageCount   uint64    // SendingMessage, +1
    FinishCount    uint64    // FinishedMessage, +1
    RequeueCount   uint64    // RequeuedMessage, +1

    // 表示正在传送中的消息个数
    // SendingMessage, +1,  从channel接受到1个消息，开始传送
    // FinishedMessage, -1, 成功传给client
    // RequeuedMessage, -1, 传输失败
    // TimedOutMessage, -1, 没有收到客户端回复，超时
    // Empty, 重置为0， channel清空
    InFlightCount int64

    // SetReadyCount, 最近一次RDY状态值, 表示client想要收到多少消息
    LastReadyCount int64

    // SetReadyCount, 重置为RDY状态
    // SendingMessage, 每一次发送消息，-1
    ReadyCount     int64

后面的三个计数会影响到client接收消息

::

    # client_v2.go
    func (c *clientV2) IsReadyForMessages() bool {
        ...

        readyCount := atomic.LoadInt64(&c.ReadyCount)
        lastReadyCount := atomic.LoadInt64(&c.LastReadyCount)
        inFlightCount := atomic.LoadInt64(&c.InFlightCount)

        if inFlightCount >= lastReadyCount || readyCount <= 0 {
           return false
        }
        return true
    }

    # protocol_v2.go#messagePump
    if subChannel == nil || !client.IsReadyForMessages() {
        // the client is not ready to receive messages...
    } else {
        ...
    }


FIN
^^^^^^^
client发给nsqd, 提示1个消息成功处理

收到这个指令，执行函数protocol_v2.go#FIN

::

    # protocol_v2.go#FIN
    # 通知channel, 消息成功发送，从inFlight队列里删除，不需要进行超时处理
    err = client.Channel.FinishMessage(client.ID, *id)
    if err != nil {
        return nil, util.NewClientErr(err, "E_FIN_FAILED",
            fmt.Sprintf("FIN %s failed %s", *id, err.Error()))
    }

    client.FinishedMessage()

    # client_v2.go
    func (c *clientV2) FinishedMessage() {
        atomic.AddUint64(&c.FinishCount, 1)
        atomic.AddInt64(&c.InFlightCount, -1)
        c.tryUpdateReadyState()
    }


REQ
^^^^^^^^
client通知nsqd, 提示1个消息处理失败

收到这个指令，执行函数protocol_v2.go#REQ


::

    # protocol_v2.go#REQ
    # 通知channel，消息发送失败，需要requeue
    err = client.Channel.RequeueMessage(client.ID, *id, timeoutDuration)
    if err != nil {
        return nil, util.NewClientErr(err, "E_REQ_FAILED",
            fmt.Sprintf("REQ %s failed %s", *id, err.Error()))
    }

    client.RequeuedMessage()

    # client_v2.go
    func (c *clientV2) RequeuedMessage() {
        atomic.AddUint64(&c.RequeueCount, 1)
        atomic.AddInt64(&c.InFlightCount, -1)
        c.tryUpdateReadyState()
    }


PUB  & MPUB
^^^^^^^^^^^^^
指定topic, 发布1个消息到nsqd.

PUB处理函数 protocol_v2.go#PUB

::

    # 放入该topic的incomingMsgChan
    topic.PutMessage(msg)


TOUCH
^^^^^^^^^^^
重置指定消息的超时时间

处理函数 protocol_v2.go#TOUCH

::

    # MsgTimeout是通过IDENTIFY指令, msg_timeout参数进行设置
    msgTimeout := client.MsgTimeout
    err = client.Channel.TouchMessage(client.ID, *id, msgTimeout)


CLS
^^^^^^
client通知这条协议，通知nsqd不再下发消息

处理函数 protocol_v2.go#CLS

::

    client.StartClose()

    # client_v2.go
    func (c *clientV2) StartClose() {
        // Force the client into ready 0
        c.SetReadyCount(0)
        // mark this client as closing
        atomic.StoreInt32(&c.State, stateClosing)
    }

    # lastReadyCount重置为0, 通知messagePump进入下一个循环,
    # 下面的判断必然为真，进入这个分支, clientMsgChan = nil,
    # 客户端不再接收消息
    if subChannel == nil || !client.IsReadyForMessages()
