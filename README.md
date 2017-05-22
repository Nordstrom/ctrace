[![OpenTracing Badge](https://img.shields.io/badge/OpenTracing-enabled-blue.svg)](http://opentracing.io)

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

* **[ctrace-js](https://github.com/Nordstrom/ctrace-js)** - Canonical OpenTracing for Javascript
* **[ctrace-go](https://github.com/Nordstrom/ctrace-go)** - Canonical OpenTracing for GoLang
* **[ctrace-py](https://github.com/Nordstrom/ctrace-py)** - Canonical OpenTracing for Python (FUTURE)
* **[ctrace-java](https://github.com/Nordstrom/ctrace-java)** - Canonical OpenTracing for Java (FUTURE)
* **[ctrace-net](https://github.com/Nordstrom/ctrace-net)** - Canonical OpenTracing for .NET (FUTURE)
* **[ctrace-es](https://github.com/Nordstrom/ctrace-es)** - Canonical OpenTracing Visualizations for ElasticSearch (FUTURE)
* **[ctrace-zipkin](https://github.com/Nordstrom/ctrace-zipkin)** - Canonical OpenTracing translation to Zipkin format (FUTURE)


## Canonical Events
ctrace supports 2 modes of output.

* **Single-Event**: In this mode only one output event is sent per Span.  This mode is the default mode.
* **Multi-Event**: In this mode multiple output events are sent per Span.

### Single-Event Mode
In this mode the following output event is sent per Span.  This mode is the default mode.

* **Finish-Span** - This event is triggered when Span.Finish is called to finish or close the Span.  In this mode, the tags, logs, and baggage are all collected during the Span lifetime and output when Span.Finish
is called.

### Multi-Event Mode
In this mode output a sent for Start-Span, Log, and Finish-Span events.  Logs only are only output for the log corresponding
to the event begin output.  Here are the output events for multi-event mode.

* **Start-Span** - This event is triggered when Tracer.StartSpan is called to create a new Span.
* **Log** - This event is triggered when Span.Log is called to log an event within the Span.
* **Finish-Span** - This event is triggered when Span.Finish is called to finish or close the Span.

## Canonical Format
The format is JSON and is used to output data from a [Span](https://github.com/opentracing/specification/blob/master/specification.md).

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
  "logs": [{
    "timestamp": " UTC Epoch Unix Timestamp in Microseconds as JSON Number representing time at
                   which this Span was started. ",
    "event":     " A stable identifier for some notable moment in the lifetime of a Span.  
                   For Start-Span and Finish-Span events this is Start-Span and Finish-Span respectively.
                   For Log event this is specified as on of the log key/values.  
                   If not, the word Log is used. ",
    "...":       " For Log events there may be other key/values given.  They are included as
                   part of the log object. "
  }],
  "baggage": {
    "...":       " baggage is a single-level JSON Object representing key/values of custom baggage or data
                   for the entire Trace that is carried from Span to Span.  If there are is no
                   baggage, the baggage object is excluded. "
  }
}
```

## Standard Span Tags and Log Fields
See [Semantic Conventions](https://github.com/opentracing/specification/blob/master/semantic_conventions.md)
for Standard Span Tags and Log Fields.

## Recommended Span Tags, Log Fields, and Trace Baggage
The following are Span Tags, Log Fields, and Trace Baggage recommended by ctrace.

### Recommended Span Tags
Span tags apply to **the entire Span**; as such, they apply to the entire timerange of the Span, not a particular moment with a particular timestamp: those sorts of events are best modelled as Span log fields (per the table in the next subsection).

|Span Tag|Type|Description|
|---|----|-----------|
|debug|bool|true if the span is considered a debug span. This can be used by ctrace implementations to exclude these spans from output when debugging is not enabled.|
|http.user_agent|string|HTTP UserAgent Header field.|
|http.remote_addr|string|HTTP X-Forwarded-For, X-Forwarded, X-Cluster-Client-Ip, or Client IP.  Shows the originating address of the call.|

### Recommended Log Fields
Every Span log has a specific timestamp (which must fall between the start and finish timestamps of the Span, inclusive) and one or more **fields**. What follows are the recommended fields.

|Log Field|Type|Description|
|-------------------|----|-----------|
|debug|bool|true if the log is considered a debug log.  This can be used by ctrace implementations to exclude these logs from output when debugging is not enabled.  NOTE: If the log belongs to a span that is also debug=true, setting the log to debug=true is not necessary as the debug span will include any logs in this same debug state.|
|http.request_body|string|In cases where there is an error detected in HTTP request handling, this field can be used to output the request body.  This can also be used in success requests as well, but it is discouraged as a general practice.|
|http.response_body|string|In cases where there is an error detected in HTTP request handling, this field can be used to output the response body.  This can also be used in success responses as well, but it is discouraged as a general practice.|

### Recommended Trace Baggage
Trace Baggage apply to **the entire Trace**; as such, they are transferred (in process or over the wire)
from Parent Spans to Child Spans.  We recommend being very careful with the usage of Baggage as it
can bloat data over the wire and cause performance repercussions.  We recommend one or more of the following Baggage as needed.

|Baggage Tag|Type|Description|
|-----------|----|-----------|
|origin|string|Identifies the origin (IP along with Region dimensions) of the top-level request or RPC call. It is useful for tracking, debugging, and defending against attack traffic.|
|agent|string|Identifies the processed user agent of the top-level request or RPC call.  This is different than the http.user_agent tag because it is for the top-level request and it has been processed, filtered, or normalized to be more useful for matching.  This tag is useful for tracking, debugging, and defending against attack traffic.|
|user|string|This represents opaque or obfuscated identification (opaque id, auth token, or similar) of the user of the top-level request or RPC call.  It is useful for tracking, debugging, and defending against attack traffic.|

## Single-Event Mode
In Single-Event Mode, the Finish-Span event is triggered when the Span is finished.  All of the data present
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
  "logs": [{
    "timestamp": 1458702548467393239,
    "event": "Finish-Span"
  },{
    "timestamp": 1458702548467399939,
    "event": "UpdateProductRecord",
    "table": "Products",
    "transactionId": "xxxy39282"
  },{
    "timestamp": 1458702548467393239,
    "event": "Start-Span"
  }],
  "baggage": {
    "origin": "216.58.194.110/US/CA/Mountain View",
    "agent": "iPhone6/iOS 10.1.0",
    "user": "83b60620f7364a8c8f07d91ccb009999"
  }
}
```

## Multi-Event Mode
In Multi-Event Mode, the Start-Span, Log, and Finish-Span events are triggered when the Span is started,
a Span log is given, or Span is finished respectively.  The logs output for each event has only 1 log that
corresponds to that event as opposed to Single-Event Mode which outputs all of the logs for the entire
Span at the time it is finished.

### Start-Span
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
  "logs": [{
  	"timestamp": 1458702548467393239,
  	"event": "Start-Span"
  }],
  "baggage": {
    "origin": "216.58.194.110/US/CA/Mountain View",
    "agent": "iPhone6/iOS 10.1.0",
    "user": "83b60620f7364a8c8f07d91ccb009999"
  }
}
```

### Log
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
  "logs": [{
    "timestamp": 1458702548467399939,
    "event": "UpdateProductRecord",
    "table": "Products",
    "transactionId": "xxxy39282"
  }],
  "baggage": {
    "origin": "216.58.194.110/US/CA/Mountain View",
    "agent": "iPhone6/iOS 10.1.0",
    "user": "83b60620f7364a8c8f07d91ccb009999"
  }
}
```

### Finish-Span
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
  "logs": [{
    "timestamp": 1458702548467393239,
    "event": "Finish-Span"
  }],
  "baggage": {
    "origin": "216.58.194.110/US/CA/Mountain View",
    "agent": "iPhone6/iOS 10.1.0",
    "user": "83b60620f7364a8c8f07d91ccb009999"
  }
}
```

## Carrier Formats
OpenTracing does not specify implementation details for transmitting SpanContext over the wire
other than providing standard Carrier Format types (see [Note: required formats for injection and extraction](https://github.com/opentracing/specification/blob/master/specification.md#note-required-formats-for-injection-and-extraction)).  ctrace provides canonical formats for these Carrier Types.

### Text Map Carrier Format
The Text Map format is passed as a key/value map with the following definition.  All keys and values are output as strings.

* `ct-trace-id` - Trace ID.
* `ct-span-id` - Span ID.
* `ct-bag-*` - Trace Baggage.  Each baggage key is prefixed with `ct-bag-`

For example

  ```
"ct-trace-id":"0308745a0f03491b"
"ct-span-id":"940a9f22e7294a8c"
"ct-bag-origin":"216.58.194.110/US/CA/Mountain View"
"ct-bag-agent":"iPhone6/iOS 10.1.0"
```

### HTTP Headers Carrier Format
The HTTP Headers format is passed as header key/values with the following definition.  All keys and values are output as strings that adhere to [Header Fields 1.1 Spec](https://tools.ietf.org/html/rfc7230#section-3.2)

* `Ct-Trace-Id` - Trace ID.
* `Ct-Span-Id` - Span ID.
* `Ct-Bag-*` - Trace Baggage.  Each baggage key is prefixed with `Ct-Bag-` and its first character is capitalized and normalization applied to adhere to [Header Fields 1.1 Spec](https://tools.ietf.org/html/rfc7230#section-3.2).

For example
```
Ct-Trace-Id: 0308745a0f03491b
Ct-Span-Id: 940a9f22e7294a8c
Ct-Bag-Origin: 216.58.194.110/US/CA/Mountain View
Ct-Bag-Agent: iPhone6/iOS 10.1.0
```

> NOTE: For consistency across languages and platforms, be sure that baggage is all lowercase.  Some languages and
> frameworks force standard header key format.  Making baggage all lowercase makes translation in such cases an easier task.

### Binary Carrier Format
TBD
