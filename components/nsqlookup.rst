nsqlookup
==============
**nsqlookupd** 是管理网络拓扑的服务进程。 通过nsqlookupd，客户端可以发现特定topic的nsqd。

有2个接口: TCP接口被nsqd用来广播; HTTP接口，消费者通过它来查找生产者和系统管理


命令行选项
-------------
::

    -broadcast-address="": address of this lookupd node, (default to the OS hostname)
    -config="": path to config file
    -http-address="0.0.0.0:4161": <addr>:<port> to listen on for HTTP clients
    -inactive-producer-timeout=5m0s: duration of time a producer will remain in the active list since its last ping
    -tcp-address="0.0.0.0:4160": <addr>:<port> to listen on for TCP clients
    -tombstone-lifetime=45s: duration of time a producer will remain tombstoned if registration remains
    -verbose=false: enable verbose logging
    -version=false: print version string


Deletion和Tombstones
----------------------------
当1个topic完全不再产生消息，从集群中删除相应的信息是相对简单的操作。
假设生成这个topic的消息的所有的生产者，都离线了, 通过对nsqlookupd使用 **/topic/delete**, 消费者就不会再找到这个主题。

对于1个channel的删除操作也是如此。区别是使用``/channel/delete``。

但是，实际过程中，当一个nsqd不再产生一个特定的toipc, 需要去掉这个toipc, 它就变的复杂。

因为其中存在1个竞争条件：

* 试图尝试删除topic信息
* 新的消费者已经发现这个主题的节点，重连, 会更新nsqlookup

解决方案是使用 **tombstones** 。
A tombstone in nsqlookupd context is producer specific and lasts for a configurable --tombstone-lifetime time.
在窗口时间内，新的消费者将不能够通过 **/lookup** 查询到该topic的生产者，旧的生产者，删除内部状态，同时广播
消息给nsqlookupd, 删除之前tombstoned的信息，这样就能防止上面提到的竞争。



HTTP接口
------------
/ping
^^^^^^^
心跳协议

::

    OK

/lookup
^^^^^^^^^
返回指定主题的一系列生产者

::

    curl http://127.0.0.1:4161/lookup?topic=t_soc_game_goods

    {
    "status_code": 200,
    "status_txt": "OK",
    "data": {
        "channels": [
        "logcons"
        ],
        "producers": [
        {
            "remote_address": "172.17.134.171:57410",
            "hostname": "devtest_zone_17_134_171",
            "broadcast_address": "172.17.134.171",  # broadcast_address:tcp_port，是生产者的连接地址
            "tcp_port": 4150,
            "http_port": 4151,
            "version": "0.2.30"
        }
        ]
    }


/topics
^^^^^^^^^
返回一系列的主题

::

    curl http://127.0.0.1:4161/topics

    {
    "status_code": 200,
    "status_txt": "OK",
    "data": {
        "topics": [
        "t_study",
        "bblog_change",
        "t_send_msg"
        ]
     }
    }


/channers
^^^^^^^^^^^
返回指定topic的channers

::

    curl http://127.0.0.1:4161/channels?topic=bblog_change

    {"status_code":200,"status_txt":"OK","data":{"channels":["logcons"]}}



/nodes
^^^^^^^^^^
返回一系列的nsqd节点

::

    {
    "status_code": 200,
    "status_txt": "OK",
    "data": {
        "producers": [ {
            "remote_address": "172.17.134.171:57410",
            "hostname": "devtest_zone_17_134_171",
            "broadcast_address": "172.17.134.171",
            "tcp_port": 4150,
            "http_port": 4151,
            "version": "0.2.30",
            # 对应下面的topic, 是否临时的表示无效
            "tombstones": [
                false,
                false,
                false,
            ],
            "topics": [
            "t_study",
            "t_send_msg",
            "tb_petquan_change",
            ]
        }
        ]
    }
    }


/topic/create
^^^^^^^^^^^^^^^^
创建主题

::

    curl http://127.0.0.1:4161/topic/create?topic=clicks

    {"status_code":200,"status_txt":"OK","data":null}


/topic/tombstone
^^^^^^^^^^^^^^^^^^^^
屏蔽主题. 屏蔽时间默认是45秒

::

    curl "http://127.0.0.1:4161/topic/tombstone?topic=clicks&node=172.17.134.171:4151"
    # 不建议使用
    curl "http://127.0.0.1:4161/tombstone_topic_producer?topic=clicks&node=172.17.134.171:4151"


/topic/delete
^^^^^^^^^^^^^^^
删除主题，会影响到 **/lookup**, **/topics**, **/nodes**, **/channers**

::

    curl http://127.0.0.1:4161/topic/delete?topic=clicks


/channel/create
^^^^^^^^^^^^^^^^
创建channel

::

    curl "http://127.0.0.1:4161/channel/create?topic=clicks&channel=metrics"

    {"status_code":200,"status_txt":"OK","data":null}


/channel/delete
^^^^^^^^^^^^^^^^^
删除channel, 会影响到 **/lookup**, **/channers**


::

    curl "http://127.0.0.1:4161/channel/delete?topic=clicks&channel=metrics"

    {"status_code":200,"status_txt":"OK","data":null}



TCP接口
--------------
nsqd使用简单的协议, 跟nsqlookupd交互。
在连接后，nsqd先发生4个字节的magic标示使用的是那个版本的协议::

    [space][space][V][1]

请求，每一行表示1条协议::

    <cmd> [arg1] [arg2] ...\n

回复协议::

    [4 bytes, BigEndian, length of data][data]


PING
^^^^^^^^
心跳协议::

    PING\n
    # data
    OK


IDENTIFY
^^^^^^^^^^
更新客户端的元信息

::

    IDENTIFY\n
    [ 4-byte size in bytes ][ N-byte JSON data ]

    # json请求，最终是组成peer对象，合法的参数见下面的tag
    type PeerInfo struct {
        lastUpdate       int64     // PING或IDENTIFY更新
        id               string    // remote adress
        RemoteAddress    string `json:"remote_address"`
        Hostname         string `json:"hostname"`
        BroadcastAddress string `json:"broadcast_address"`
        TcpPort          int    `json:"tcp_port"`
        HttpPort         int    `json:"http_port"`
        Version          string `json:"version"`
    }

    # 返回json字符串, 例如
    {"tcp_port": "lookupd的tcp端口", "http_port": "lookupd的http端口",
        "version": "协议版本", "broadcast_address": "", "hostname": ""}


REGISTER
^^^^^^^^^^
注册topic, channel

::

   REGISTER  <topicName> [channelName]\n

   # 返回
   OK


UNREGISTER
^^^^^^^^^^^^
取消注册topic, channel

::

    UNREGISTER <topicName> [channelName]\n

    #返回
    OK


统一的服务器模式
------------------
利用 :ref:`goroutine_chapter` 提到的管理方式。nsqlookupd, nsqd等服务代码都采用了下面的模式

::

    # apps/nsqlookupd/nsqlookupd.go

    var (
        flagSet = flag.NewFlagSet("nsqlookupd", flag.ExitOnError)

        config      = flagSet.String("config", "", "path to config file"))
        ...
    )

    func main() {
        flagSet.Parse(os.Args[1:])

        if *showVersion {
            fmt.Println(util.Version("nsqlookupd"))
            return
        }

        # 启动单独的goroutine, 监听信号量
        signalChan := make(chan os.Signal, 1)
        exitChan := make(chan int)
        go func() {
            <-signalChan
            exitChan <- 1
        }()
        signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)

        # 配置优先从命令行选项中获取，然后才从配置文件获取
        # 实践：现网使用配置文件，测试阶段用命令选项
        var cfg map[string]interface{}
        if *config != "" {
            _, err := toml.DecodeFile(*config, &cfg)
            if err != nil {
                log.Fatalf("ERROR: failed to load config file %s - %s", *config, err.Error())
            }
        }

        opts := nsqlookupd.NewNSQLookupdOptions()
        options.Resolve(opts, flagSet, cfg)
        daemon := nsqlookupd.NewNSQLookupd(opts)

        log.Println(util.Version("nsqlookupd"))

        daemon.Main()
        # 主routine阻塞，等待信号量
        <-exitChan
        daemon.Exit()
    }


    # nsqlookupd/nsqlookupd.go

    func (l *NSQLookupd) Main() {
        ...

        l.waitGroup.Wrap(func() { util.TCPServer(tcpListener, tcpServer) })

        # 启动单独的协程，进入loop中
        l.waitGroup.Wrap(func() {
            util.HTTPServer(httpListener, httpServer, l.opts.Logger, "HTTP")
        })
    }

    func (l *NSQLookupd) Exit() {
        # 做一些服务器的清理工作
        ...

        # 这里会堵塞，只到服务相关的所有的协程都结束，才会继续执行
        l.waitGroup.Wait()
    }



性能监控
------------------
NSQ的进程都使用了pprof，可以随时知道实时的性能指标。

例如

::

    # 文件 nsqlookupd/http.go
    import httpprof "net/http/pprof"

    func (s *httpServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
        ...

        err = s.debugRouter(w, req)
        if err != nil {
            log.Printf("ERROR: %s", err)
            util.ApiResponse(w, 404, "NOT_FOUND", nil)
        }
    }

    func (s *httpServer) debugRouter(w http.ResponseWriter, req *http.Request) error {
        switch req.URL.Path {
        case "/debug":
            util.NegotiateAPIResponseWrapper(w, req,
                func() (interface{}, error) { return s.doDebug(req) })
        case "/debug/pprof":
            httpprof.Index(w, req)
        case "/debug/pprof/cmdline":
            httpprof.Cmdline(w, req)
        case "/debug/pprof/symbol":
            httpprof.Symbol(w, req)
        case "/debug/pprof/heap":
            httpprof.Handler("heap").ServeHTTP(w, req)
        case "/debug/pprof/goroutine":
            httpprof.Handler("goroutine").ServeHTTP(w, req)
        case "/debug/pprof/profile":
            httpprof.Profile(w, req)
        case "/debug/pprof/block":
            httpprof.Handler("block").ServeHTTP(w, req)
        case "/debug/pprof/threadcreate":
            httpprof.Handler("threadcreate").ServeHTTP(w, req)
        default:
            return errors.New(fmt.Sprintf("404 %s", req.URL.Path))
        }
        return nil
    }


cpu
^^^^^^^^^^^^^^^
下面的例子, nsqd部署在172.17.134.171，监听4151端口

::

    # 第一个参数是程序的二进制文件， 第二个参数是程序运行http pprof的地址
    pprof --pdf  newlog/logcons/bin/logcons http://10.194.1.17:5001/debug/pprof/profile > /tmp/logcons.pdf

    # pdf支持需要安装下面的包
    yum install ghostscript
    yum install graphviz

    # 输出细节见 http://google-perftools.googlecode.com/svn/trunk/doc/cpuprofile.html


heap
^^^^^^^^^^^^^^
堆栈信息

::

    pprof --pdf  http://172.17.134.171:4151/debug/pprof/heap > heap.pdf

    # 输出细节见 http://google-perftools.googlecode.com/svn/trunk/doc/heapprofile.html


goroutine
^^^^^^^^^^^^
线程信息

::

   curl  http://172.17.134.171:4151/debug/pprof/goroutine?debug=2
