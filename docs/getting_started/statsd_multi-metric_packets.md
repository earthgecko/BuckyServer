## statsd multi-metric packets with BuckyServer

statds has the ability to receive multi-packet metrics (https://github.com/etsy/statsd/blob/master/docs/metric_types.md#multi-metric-packets)

```
gorets:1|c\nglork:320|ms\ngaugor:333|g\nuniques:765|s

```

In some other clients such as nc when you send this directly to statsd, you send that string as is with `\n`, not with BuckyServer and this can be confusing.

With BuckyServer the multi-metric packets udp rules do not apply, there are no such limits via TCP.

BuckyServer gives you the ability to ship valuable statsd metrics more reliably longhual over TCP, rather than longhual over UDP.  However this does have a price, you lose non-blocking fire and forget, but it does have a number of advantages too.

It is very important from the client side to be able to ship as many metrics per call as possible, with statds the packet size is limited by "total length of the payload within your network's MTU".
There you may have to split your metrics into string < max_paylod_bytes and loop through with nc hits to statsd many times (fire and forget).
With the HTTP client to BuckyServer can would send all of those metrics in one `POST` with no `\n`

This is very advantageous if you are sending lots of critical long namespace metrics to statsd, especially from distributed, longhaul geographic regions.

BuckyServer simply submits each metric to statsd, it does not submit them to statsd as multi-metrics packets (so more UDP connections local, but reliable UDP).

With the HTTP client the post data must not have the `\n` but rather a metric per line.

*Good POST data*
```
POST_DATA="test.bucky.alive:1|g
test.bucky.does_multi_metric_packets:1|c"
```

This example below would be submited to statsd, but it would *NOT* make it into statsd, meaning to would not be forwarded on to graphite, et al.

*BAD POST data \n*
```
BAD_POST_DATA="test.bucky.alive:1|g\ntest.bucky.does_multi_metric_packets:1|c"

```

#### HTTP client to BuckyServer and statsd

With an HTTP client like wget (or curl) to make it "non blocking" do not forget timeouts (no curl timeout was added in example)

```
BUCKYSERVER="your.buckyserver.ipaddress"
BUCKYSERVERPORT="your.buckyserver.port"
POST_DATA="test.bucky.alive:1|g
test.bucky.does_multi_metric_packets:1|c"
wget  --tries=1 --timeout=1 --dns-timeout=1 --post-data="$POST_DATA" --header="Content-Type: text/plain" http://$BUCKYSERVER:$BUCKYSERVERPORT/bucky/v1/send
# or with curl
# curl -X POST -H "Content-Type: text/plain" -d "$POST_DATA" http://$BUCKYSERVER:$BUCKYSERVERPORT/bucky/v1/send
```

BuckyServer will send statsd each line, no udp multi-metric packets required.

```
[me@here ~] cat /tmp/tcpdump.8125.log | grep "bucky"
E.....@.@.<h.............o..test.bucky.alive:1|g
test.bucky.does_multi_metric_packets:1|c
[me@here ~]
```

The correct metrics and values are relayed to graphite and our valuable metrics have TCP transport.
