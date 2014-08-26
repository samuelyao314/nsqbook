nsqlookupd源码
=======================

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






参考资料
^^^^^^^^^^^^^^^

