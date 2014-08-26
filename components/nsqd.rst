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


