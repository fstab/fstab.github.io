---
layout:   post
title:    "HTTP/2 Java Client Examples"
date:     2015-10-10 12:55:12
comments: true
---

This post presents some examples of [HTTP/2][1] Java clients.
The source code for the examples can be found on [github.com/fstab/http2-examples][2].

The examples show how to run multiple requests over a single connection
with HTTP/2's [multiplexing][3].
Multiplexing is HTTP/2's ability to handle multiple requests within one TCP
connection independent of each other, so a blocked or stalled request or
response does not prevent progress on other streams.

The [example server][4] takes GET requests and responds after 6 seconds delay.
Each client sends three GET requests over a single connection.
With HTTP/1 this would imply that the last response is received after
[3*6=18][5] seconds.
As the clients use HTTP/2, all requests are processed in parallel,
so all three responses are received after 6 seconds.

The implementations exemplify the three most common HTTP/2 client libraries
for Java developers.

Jetty Client
------------

The [Jetty HTTP/2 client][8] provides a low level API and a high level API.
As of now, the high level API seems still [not very stable][6],
and I found it easier to use the low level API to implement the example.
The code is mostly copied from an example in the Javadoc for Jetty's
[`HTTP2Client`][7] class.

To send the request, you have to create a HEADERS frame from a `Request`
template and send it in a new stream:

{% highlight java %}
MetaData.Request request = new MetaData.Request("GET", new HttpURI("https://localhost:8443"), HttpVersion.HTTP_2, requestFields);
HeadersFrame headersFrame = new HeadersFrame(request, null, true);
session.newStream(headersFrame, new FuturePromise<>(), responseListener);
{% endhighlight %}

The response is received in the `responseListener` callback. The `ResponseListener`
implements a `onData()` method that is invoked when response data is received:

{% highlight java %}
Stream.Listener responseListener = new Stream.Listener.Adapter() {
    @Override
    public void onData(Stream stream, DataFrame frame, Callback callback) {
        // ... do something with frame.getData()
        callback.succeeded();
    }
}
{% endhighlight %}

Netty Client
------------

Like Jetty, the [Netty project][9] also provides a low level API for HTTP/2 clients.
A working example is provided in [`io.netty.example.http2.client`][10].

To create a request, you have to create an HTTP/1 request object.
As the connection is an HTTP/2 connection, the request will be implicitly
converted to HTTP/2 frames internally.

{% highlight java %}
FullHttpRequest request = new DefaultFullHttpRequest(HTTP_1_1, GET, "");
request.headers().add(...); // add custom request headers
channel.writeAndFlush(request);
{% endhighlight %}

In order to receive the response, you have to implement a response handler
and register it with the socket channel's pipeline in the client's bootstrap
code:

{% highlight java %}
public class Http2ClientInitializer extends ChannelInitializer<SocketChannel> {
    @Override
    public void initChannel(SocketChannel ch) throws Exception {
        // ...
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast("HttpResponseHandler", responseHandler);
    }
}
{% endhighlight %}

The `HttpResponseHandler` in [Netty's example code][10] uses the stream ID
for mapping response messages to the corresponding `ChannelPromies` objects.
In order to do that, the implementation assumes that the stream ID of the first
request is 3, and the stream ID for each subsequent request is incremented by 2.
That's an easy way to allow the `HttpResponseHandler` to learn which
promise is associated with a response, but it means that the client needs
to keep track of the number of requests sent.

The `HttpResponseHandler` implements a `messageReceived()` callback that
is invoked when responses are received.

{% highlight java %}
public class HttpResponseHandler extends SimpleChannelInboundHandler<FullHttpResponse> {
    @Override
    protected void messageReceived(ChannelHandlerContext ctx, FullHttpResponse msg) throws Exception {
        // do something with msg.content()
        ChannelPromise promise = streamidPromiseMap.get(streamId);
        promise.setSuccess();
    }
}
{% endhighlight %}

OkHttp Client
-------------

The [OkHttp][12] example uses a much more high level API than the two other examples:

{% highlight java %}
Request request = new Request.Builder()
    .url("https://localhost:8443")
    .build();
client.newCall(request).enqueue(/* callback */);
{% endhighlight %}

Requests are created with a `Request.Builder`, and sent by registering a
callback for the response.

Responses are handled in the callback's `onResponse()` method:

{% highlight java %}
new Callback() {
    public void onResponse(Response response) throws IOException {
        // do something with response.body()
    }
}
{% endhighlight %}

Summary
-------

All three Java client libraries are able to re-use HTTP/2 connections for
multiple requests, and to implement efficient multiplexing within the connection.
As of now, [OkHttp][12] seems to have the most easy to use API.
However, it should be easy to wrap the other two examples into a higher level
API if needed.

[1]: https://httpwg.github.io/specs/rfc7540.html#Overview
[2]: https://github.com/fstab/http2-examples/tree/master/multiplexing-examples
[3]: https://httpwg.github.io/specs/rfc7540.html#StreamsLayer
[4]: https://github.com/fstab/http2-examples/blob/master/multiplexing-examples/server/src/main/java/de/consol/labs/h2c/examples/server/Servlet.java
[5]: https://en.wikipedia.org/wiki/Head-of-line_blocking
[6]: https://bugs.eclipse.org/bugs/show_bug.cgi?id=479277
[7]: http://download.eclipse.org/jetty/stable-9/apidocs/org/eclipse/jetty/http2/client/HTTP2Client.html
[8]: http://www.eclipse.org/jetty/documentation/current/http2.html
[9]: http://netty.io/
[10]: http://netty.io/5.0/xref/io/netty/example/http2/client/package-summary.html
[11]: http://netty.io/5.0/xref/io/netty/example/http2/client/Http2Client.html
[12]: http://square.github.io/okhttp/
