nsqadmin
===========
nsqadmin is a Web UI to view aggregated cluster stats in realtime and perform various administrative tasks.


Command Line Options
-----------------------
::

    -config="": path to config file
    -graphite-url="": graphite HTTP address
    -http-address="0.0.0.0:4171": <addr>:<port> to listen on for HTTP clients
    -lookupd-http-address=: lookupd HTTP address (may be given multiple times)
    -notification-http-endpoint="": HTTP endpoint (fully qualified) to which POST notifications of admin actions will be sent
    -nsqd-http-address=: nsqd HTTP address (may be given multiple times)
    -proxy-graphite=false: proxy HTTP requests to graphite
    -statsd-interval=1m0s: time interval nsqd is configured to push to statsd (must match nsqd)
    -statsd-prefix="nsq.%s": prefix used for keys sent to statsd (%s for host replacement, must match nsqd)
    template-dir="": path to templates directory
    -use-statsd-prefixes=true: expect statsd prefixed keys in graphite (ie: 'stats_counts.')
    -version=false: print version string



statsd / Graphite Integration
------------------------------
