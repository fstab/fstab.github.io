---
layout:   post
title:    "Stream States"
date:     2016-03-10
comments: true
---

In [HTTP/2], a _stream_ is a request/response interaction.
According to the HTTP/2 Standard, each stream is created in state `idle`,
and goes through a couple of states until it eventually reaches `closed`.

{% highlight text %}
                             +--------+
                     send PP |        | recv PP
                    ,--------|  idle  |--------.
                   /         |        |         \
                  v          +--------+          v
           +----------+          |           +----------+
           |          |          | send H /  |          |
    ,------| reserved |          | recv H    | reserved |------.
    |      | (local)  |          |           | (remote) |      |
    |      +----------+          v           +----------+      |
    |          |             +--------+             |          |
    |          |     recv ES |        | send ES     |          |
    |   send H |     ,-------|  open  |-------.     | recv H   |
    |          |    /        |        |        \    |          |
    |          v   v         +--------+         v   v          |
    |      +----------+          |           +----------+      |
    |      |   half   |          |           |   half   |      |
    |      |  closed  |          | send R /  |  closed  |      |
    |      | (remote) |          | recv R    | (local)  |      |
    |      +----------+          |           +----------+      |
    |           |                |                 |           |
    |           | send ES /      |       recv ES / |           |
    |           | send R /       v        send R / |           |
    |           | recv R     +--------+   recv R   |           |
    | send R /  `----------->|        |<-----------'  send R / |
    | recv R                 | closed |               recv R   |
    `----------------------->|        |<----------------------'
                             +--------+

       send:   endpoint sends this frame
       recv:   endpoint receives this frame

       H:  HEADERS frame (with implied CONTINUATIONs)
       PP: PUSH_PROMISE frame (with implied CONTINUATIONs)
       ES: END_STREAM flag
       R:  RST_STREAM frame
{% endhighlight %}

Since version [v0.0.11], [h2c] implements a `h2c stream-info` command that can
be used to view the stream states on the client.
This post shows some examples of the command.

## Run the Echo Server

We will re-use the [Echo Server] from the [Server Push Demo] to run the
_stream-info_ examples below. The [Echo Server] takes a POST request, and
then sends the received data back to the client using a [Server Push] message.

Download and compile the [Echo Server] as follows:

{% highlight bash %}
git clone https://github.com/fstab/http2-examples
cd http2-examples/jetty-http2-echo-server
mvn package
{% endhighlight %}

In order to run it, learn the [correct ALPN version for your JDK]
_(see table 14.1)_, and download the [ALPN boot JAR from Maven Central].
With the right `<path-to-alpn-boot.jar>`, run the [Echo Server] as follows:

{% highlight bash %}
java -Xbootclasspath/p:<path-to-alpn-boot.jar> \
    -jar target/jetty-http2-echo-server.jar
{% endhighlight %}

## Example Script

The `h2c stream-info` command is intended to be used when debugging a server,
where we can set breakpoints and go through each interaction step-by-step.
However, if we put a bunch of `h2c stream-info` commands into a shell script,
they are usually executed fast enough to see the state transitions.
Create a shell script `./test.sh` with the following commands
_(the example runs on Linux and OS X. For Windows use [cygwin])_:

{% highlight bash %}
#!/bin/bash

# Create a 66k file in /tmp/data
dd if=/dev/zero of=/tmp/data bs=1024 count=66

h2c connect localhost:8443
h2c post --file /tmp/data /data > /dev/null &
h2c stream-info
h2c stream-info
h2c stream-info
# ... repeat `h2c stream-info` a couple of times
{% endhighlight %}

Note that the `post` command is run in the background, because if we wait for
the `post` to be finished, we will only see streams in state `closed`.

Start [h2c] like this:

{% highlight bash %}
h2c start --dump
{% endhighlight %}

In another shell window, run `./test.sh`.

## Stream States for the POST request

The output shows stream states for two streams:

* The POST request with stream ID `1`.
* The Server Push message with stream  ID `2`.

Let's analyze the stream states for the POST request with stream ID `1` first:

{% highlight text %}
1: POST /data open
1: POST /data half closed (local)
1: POST /data closed
{% endhighlight %}

We cannot observe the initial `idle` state, because when the stream is created,
it switches to `open` immediately sending the request header.
When the request body is sent (`DATA` frame with `END_STREAM`), the state
becomes `half closed (local)`, indicating that the client is done with
its part of the request/response interaction.

Receiving the response moves the stream state to `closed`.

## Stream States for the Server Push message

As described in the [Server Push Demo], the [Echo Server] sends the posted
data back using a Server Push message.
The Server Push message consists of three parts sent from the server
to the client:

* `PUSH_PROMISE` to indicate that the server will send a "promised" response
  for the URL `/data`.
* The header of the "promised" response.
* The body of the "promised" response.

The corresponding state transitions for stream ID `2` are as follows:

{% highlight text %}
2: GET /data reserved (remote) (cached push promise)
2: GET /data half closed (local) (cached push promise)
2: GET /data closed (cached push promise)
{% endhighlight %}

Again, we cannot observe the initial `idle` state, because when the
stream is created upon receiving the `PUSH_PROMISE`, the stream
goes immediately from `idle` to `reserved (remote)`.
Receiving the header moves the stream state to `half closed (local)`,
receiving the body moves the state to `closed`.

Note that the "promised" stream goes from `idle` to `closed` only through
receiving frames, there is no sent frame involved.

## Summary

The `h2c stream-info` command provides a way to view [HTTP/2 stream states]
on the [h2c] client. This is particularly useful for debugging server
applications.

[HTTP/2]: http://http2.github.io
[v0.0.11]: https://github.com/fstab/h2c/releases
[h2c]: https://github.com/fstab/h2c
[Echo Server]: https://github.com/fstab/http2-examples/tree/master/jetty-http2-echo-server
[Server Push Demo]: http://unrestful.io/2015/09/23/push.html
[Server Push]: http://httpwg.org/specs/rfc7540.html#PushResources
[correct ALPN version for your JDK]: http://www.eclipse.org/jetty/documentation/current/alpn-chapter.html
[ALPN boot JAR from Maven Central]: http://repo1.maven.org/maven2/org/mortbay/jetty/alpn/alpn-boot/
[cygwin]: https://www.cygwin.com
[HTTP/2 stream states]: http://httpwg.org/specs/rfc7540.html#StreamStates
