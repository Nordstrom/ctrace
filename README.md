# ctrace
Canonical Trace.  ctrace is a Canonical [OpenTracing](http://opentracing.io/) output specification.  

## Why
[OpenTracing](http://opentracing.io) is a young specification and for most (if not all) SDK implementations, output format and wire protocol are specific to the backend platform implementation.  ctrace attempts to decouple the format and wire protocol from the backend tracer implementation.

## What
ctrace specifies a canonical format for trace logs.  By default the logs are output to stdout but you can configure them to go to any writable stream (file, etc).

## Required Reading
To fully understand this platform API, it's helpful to be familiar with the [OpenTracing project](http://opentracing.io) project, [terminology](http://opentracing.io/documentation/pages/spec.html), and [ctrace specification](https://github.com/Nordstrom/ctrace) more specifically.

## High-level View
The following shows how the ctrace format and libraries fit into a normal log aggregation strategy.

![High-level View](ctrace.png)

* Language libraries can be used to log trace events on Servers (EC2, K8 Instance, etc) to stdout (preferred) or a file.
* Language libraries can be used to log trace events in AWS Lambdas to stdout which is forwarded to Cloud Watch Logs.
* Logs are forwarded from Servers using traditional log forwarders (i.e. Logstash, Beats, Loggly, etc) to forward to your log collector / aggregator (i.e. Logstash / ElasticSearch, Loggly, etc)
* Visualization libraries provide connected trace visualizations on top of your aggregator / search (i.e. ElasticSearch, Zipkin, Loggly, etc)

## Canonical Libraries
The following libraries support the ctrace specification.

* **[ctrace-js](https://github.com/Nordstrom/ctrace-js/tree/new)** - Canonical OpenTracing for Javascript
* **[ctrace-go](https://github.com/Nordstrom/ctrace-go/tree/new)** - Canonical OpenTracing for GoLang
* **[ctrace-py](https://github.com/Nordstrom/ctrace-py)** - Canonical OpenTracing for Python (FUTURE)
* **[ctrace-java](https://github.com/Nordstrom/ctrace-java)** - Canonical OpenTracing for Java (FUTURE)
* **[ctrace-net](https://github.com/Nordstrom/ctrace-net)** - Canonical OpenTracing for .NET (FUTURE)
* **[ctrace-es](https://github.com/Nordstrom/ctrace-es)** - Canonical OpenTracing Visualizations for ElasticSearch (FUTURE)
* **[ctrace-zipkin](https://github.com/Nordstrom/ctrace-zipkin)** - Canonical OpenTracing translation to Zipkin format (FUTURE)


## Canonical Format
The format is JSON and is used to output data from a [Span](https://github.com/opentracing/specification/blob/master/specification.md) for 3 events:

* **Start-Span** - This event is triggered when Tracer.StartSpan is called to create a new Span.
* **Log** - This event is triggered when Span.Log is called to log an event within the Span.
* **Finish-Span** - This event is triggered when Span.Finish is called to finish or close the Span.

Here is the format:

```json
{
  "traceId":    " UInt64 in Hex (Base 16) representing the unique id of the entire multi-span Trace. ",
  "spanId":     " UInt64 in Hex (Base 16) representing the unique id of this Span. ",
  "parentId":   " UInt64 in Hex (Base 16) representing the unique id of the Parent to this Span.  
                  If this is an originating Span, this field is excluded. ",
  "operation":  " A human-readable string which concisely represents the work done by the Span
                  (for example, an RPC method name, a function name, or the name of a subtask or
                  stage within a larger computation). ",
  "start":      " UTC Epoch Unix Timestamp in Microseconds as JSON Number representing time at
                  which this Span was started. ",
  "tags": {
    "...":      " tags is a single-level JSON Object representing key/values of custom tags or data
                  for this Span.  If there are no tags, the tags object is excluded."
  },
  "log": {
    "timestamp": " UTC Epoch Unix Timestamp in Microseconds as JSON Number representing time at
                   which this Span was started. ",
    "event":     " A stable identifier for some notable moment in the lifetime of a Span.  
                   For Start-Span and Finish-Span events this is Start-Span and Finish-Span respectively.
                   For Log event this is specified as on of the log key/values.  
                   If not, the word Log is used. ",
    "...":       " For Log event there may be other key/values given.  They are included as
                   part of the log object. "
  },
  "baggage": {
    "...":       " baggage is a single-level JSON Object representing key/values of custom baggage or data
                   for the entire Trace that is carried from Span to Span.  If there are is no
                   baggage, the baggage object is excluded. "
  }
}
```

### Standard Tags
See [OpenTracing Data Conventions](https://github.com/opentracing/specification/blob/master/data_conventions.yaml)
for Standard Tags.  They are also listed here.

|Tag|Type|Description|Examples|
|---|----|-----------|--------|
|error|bool|true if and only if the associated Span is in an error state||
|component|string|The software package, framework, library, or module that generated the associated Span|"grpc", "django", "JDBI"|
|http.url|string|URL of the request being handled in this segment of the trace, in standard URI format|"https://domain.net/path/to?resource=here"|
|http.method|string|HTTP method of the request for the associated Span|"GET", "POST"|
|http.status_code|integer|HTTP response status code for the associated Span|200, 503, 404|
|span.kind|string|Either "client" or "server" for the appropriate roles in an RPC|"client", "server"|
|peer.hostname|string|Remote hostname'|"opentracing.io", "internal.dns.name"|
|peer.ipv4|string|Remote IPv4 address as a .-separated tuple|"127.0.0.1"|
|peer.ipv6|string|Remote IPv6 address as a string of colon-separated 4-char hex tuples|"2001:0db8:85a3:0000:0000:8a2e:0370:7334"|
|peer.port|integer|Remote port|80|
|peer.service|string|Remote service name (for some unspecified definition of "service")|'"elasticsearch", "a_custom_microservice", "memcache"|

### Recommended Tags
The following are tags we recommend.

|Tag|Type|Description|
|---|----|-----------|
|debug|bool|true if the Log is considered a debug log.  This can be used by ctrace implementations to exclude debug Logs from output.|
|exception|string|Error message and/or stack trace.  Should be included if error=true.|
|http.user_agent|string|HTTP UserAgent Header field.|
|http.remote_addr|string|HTTP X-Forwarded-For, X-Forwarded, X-Cluster-Client-Ip, or Client IP.  Shows the originating address of the call.|



## Start-Span
The Start-Span event is triggered when the Span is started.  All of the data present
in the Span at this time is output to the writable stream.  Here is an example.

```json
{
  "traceId": "0308745a0f03491b",
  "spanId": "940a9f22e7294a8c",
  "parentId": "19d0ea9d414f47f1",
  "operation": "CreateProduct",
  "start": 1458702548467393239,
  "tags": {
    "component": "ProductUpdater",
    "span.kind": "server",
    "http.url": "https://api.nordstrom.com/v1/products",
    "http.method": "POST",
    "http.user_agent": "Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/21.0",
    "http.remote_addr": "192.168.1.2",
    "styleId": "29392832",
    "sku": "293820133",
    "label": "New Pump"
  },
  "log": {
  	"timestamp": 1458702548467393239,
  	"event": "Start-Span"
  }
}
```

## Log
The Log event is triggered when Span.Log is called.  All of the data present in
the Span at this time is output to the writable stream with the log field being
populated by the key/values passed into the Log method.  Here is an example.

```json
{
  "traceId": "0308745a0f03491b",
  "spanId": "940a9f22e7294a8c",
  "parentId": "19d0ea9d414f47f1",
  "operation": "CreateProduct",
  "start": 1458702548467393239,
  "tags": {
    "component": "ProductUpdater",
    "span.kind": "server",
    "http.url": "https://api.nordstrom.com/v1/products",
    "http.method": "POST",
    "http.user_agent": "Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/21.0",
    "http.remote_addr": "192.168.1.2",
    "styleId": "29392832",
    "sku": "293820133",
    "label": "New Pump"
  },
  "log": {
    "timestamp": 1458702548467399939,
    "event": "UpdateProductRecord",
    "table": "Products",
    "transactionId": "xxxy39282"
  }
}
```

## Finish-Span
The Finish-Span event is triggered when the Span is finished.  All of the data present
in the Span at this time is output to the writable stream.  Here is an example.

```json
{
  "traceId": "0308745a0f03491b",
  "spanId": "940a9f22e7294a8c",
  "parentId": "19d0ea9d414f47f1",
  "operation": "CreateProduct",
  "start": 1458702548467393239,
  "duration": 738,
  "tags": {
    "component": "ProductUpdater",
    "span.kind": "server",
    "http.url": "https://api.nordstrom.com/v1/products",
    "http.method": "POST",
    "http.user_agent": "Mozilla/5.0 (X11; Linux x86_64; rv:12.0) Gecko/20100101 Firefox/21.0",
    "http.status_code": 200,
    "http.remote_addr": "192.168.1.2",
    "styleId": "29392832",
    "sku": "293820133",
    "label": "New Pump"
  },
  "log": {
    "timestamp": 1458702548467393239,
    "event": "Finish-Span"
  }
}
```
