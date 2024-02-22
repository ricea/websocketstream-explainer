# WebSocketStream Explained


## Introduction

The [WebSocket API](https://html.spec.whatwg.org/multipage/web-sockets.html)
provides a JavaScript interface to the
[RFC6455](https://tools.ietf.org/html/rfc6455) WebSocket protocol. While it has
served well, it is awkward from an ergonomics perspective and is missing the
important feature of
[backpressure](https://streams.spec.whatwg.org/#backpressure). In particular,

* The
  [onmessage](https://html.spec.whatwg.org/multipage/web-sockets.html#handler-websocket-onmessage)
  event will keep firing until the page becomes completely unresponsive. The
  user agent will buffer incoming messages until it runs out of memory and
  crashes.
* The only way to determine when the network or remote server can’t keep up
  with your sent messages is to test the
  [bufferedAmount](https://html.spec.whatwg.org/multipage/web-sockets.html#dom-websocket-bufferedamount)
  attribute. To find out when it is safe to start sending messages again, it is
  necessary to poll bufferedAmount.

WebSocketStream aims to solve these deficiencies with a new API.

Here’s a basic example of usage of the new API:

```javascript
const wss = new WebSocketStream(url);
const { readable } = await wss.opened;
const reader = readable.getReader();
while (true) {
  const { value, done } = await reader.read();
  if (done)
    break;
  await process(value);
}
done();
```

This is the roughly equivalent code with the old API:

```javascript
const ws = new WebSocket(url);
ws.onmessage = evt => process(evt.data);
ws.onclose = evt => evt.wasClean ? done() : signalErrorSomehow();
```

The major difference is that the second example won’t wait for asynchronous
activity in `process()` to complete before calling it again; it will keep
hammering it as long as messages keep arriving.

Also note that because the old API was designed before Promises were added to
the language, error-handling is awkward.

Writing also uses the backpressure facilities of the Streams API:

```javascript
const wss = new WebSocketStream(url);
const { writable } = await wss.opened;
const writer = writable.getWriter();
for await (const message of messages) {
  await writer.write(message);
}
```

The second argument to WebSocketStream is an option bag to allow for future
extension. One option is “protocols”, which behaves the same as the second
argument to the WebSocket constructor:

```javascript
const wss = new WebSocketStream(url, {protocols: ['chat', 'chatv2']});
const { protocol } = await wss.opened;
```

The selected protocol is part of the dictionary available via the
`wss.opened` promise, along with “extensions”. All the information about
the live connection is provided by this promise, since it is not relevant if the
connection failed.

```javascript
const { readable, writable, protocol, extensions } = await wss.opened;
```

The information that was available from the onclose and onerror events in the
old API is now available via the “closed” Promise. This rejects in the event
of an unclean close, otherwise it resolves to the code and reason sent by the
server.

```javascript
const { closeCode, reason } = await wss.closed;
```

An AbortSignal passed to the constructor makes it simple to abort the handshake.

```javascript
const controller = new AbortController();
const wss = new WebSocketStream(url, { signal: controller.signal });
setTimeout(() => controller.abort(), 1000);
```

The close method can also be used to abort the handshake, but its main purpose
is to permit specifying the code and reason which is sent to the server.

```javascript
wss.close({closeCode: 4000, reason: 'Game over'});
```


## Mapping to the protocol

There is a 1:1 mapping between WebSocket messages and stream chunks.

Each call to `read()` returns one WebSocket message. If a message is split into
multiple frames on the wire, it won't be returned by `read()` until the final
frame (the one with the FIN flag set) arrives.

When `read()` is not called, the browser and operating system will still buffer
data to some extent, so backpressure will not be detected immediately by the
server.

Text messages appear in JavaScript as strings. Binary messages appear as
Uint8Array objects.

A clean close will result in `read()` returning an object with `done` set to
true. An unclean close will result in a rejected promise.

Each call to `write()` (or chunk that is piped into the `writable`) will be
converted to one message. The browser may split the message into multiple
frames. BufferSource (ArrayBuffer or TypedArray) objects will be sent as binary
WebSocket messages. Any other type will be converted to a string and sent as a
text message.

The promise returned by `write()` will resolve when the message has been
buffered (either by the browser or operating system). The size of the buffer is
finite but unspecified. It is not a signal that the message has been delivered
to the WebSocket server (the browser does not have this information).

The promise returned by `write()` will reject if the connection is closed or
errored.


## Goals

* Provide a WebSocket API that supports backpressure
* Provide a modern, ergonomic, easy-to-use WebSocket API
* Allow for future extension


## Non-goals

* Support Blob chunks. The old WebSocket API defaults to receiving messages as
  Blobs; however, creating and reading Blobs is more costly than creating and
  reading ArrayBuffers. In practice, even though it requires explicitly setting
  binaryType, 97% of messages are received as ArrayBuffers. On the send side,
  sending Blobs adds considerable complexity to the implementation because the
  contents are not available synchronously. Since less than 4% of sent messages
  are Blobs it is better to avoid this complexity where we can.
* Changing, replacing or extending the underlying network protocol.
  [WebTransport](https://developer.mozilla.org/en-US/docs/Web/API/WebTransport)
  has many advanced features that are not supported by the WebSocket protocol,
  such as datagram support over UDP/IP. It should be preferred when advanced
  networking features are required.
* Allowing user JavaScript to select [WebSocket
  extensions](https://tools.ietf.org/html/rfc6455#page-48). Since the server
  already negotiates the extensions to use, adding additional controls to client
  JavaScript seems redundant. The existing JavaScript API has never supported
  this, although some non-browser implementations have added options to the
  constructor for it.


## Non-goals in the first version

* Bring-your-own-buffer reading
* Reading or writing individual messages as streams (for example, to handle
  messages larger than memory)
* Exposing WebSocket [pings and
  pongs](https://tools.ietf.org/html/rfc6455#page-37) to JavaScript.


## Use cases

* High-bandwidth WebSocket applications that need to retain interactivity, in
  particular video and screen-sharing.
* Similarly, video capture and other applications that generate a lot of data in
  the browser that needs to be uploaded to the server. With backpressure, the
  client can stop producing data rather than accumulating data in memory.


## End-user benefits

Applications written with the new API will automatically be more responsive due
to respecting backpressure. High throughput applications will adapt to the
capabilities of the client, providing everyone with a smooth experience.


## Alternatives

It’s possible to implement backpressure at the application level, but it’s
complex and difficult to achieve peak throughput. For example, client JavaScript
could send an application-level confirmation message to the server every time it
finishes processing a message. The server could keep track of how many messages
it has sent that have not yet been confirmed, and stop sending if the number
gets above a certain threshhold.

Aside from backpressure, the rest of the API can be emulated by wrapping the
existing WebSocket API, but this will not permit future extensions.

Adding new attributes to the existing WebSocket API was considered but not
adopted because having two APIs on one object would be confusing and create odd
semantics.

[WebTransport](https://github.com/WICG/web-transport/blob/master/explainer.md)
also provides backpressure, and may replace WebSocket for many purposes. In the
near future the WebSocket protocol has the advantage that it works on networks
that block QUIC, and has much existing deployed infrastructure.

An older version of this explainer had the readable stream producing ArrayBuffer
chunks. This was changed to Uint8Array chunks to align better with WebTransport
and modern practice.

Previously the `closeCode` attribute was called `code`, but this conflicted with
the `code` attribute of `DOMException`.


## Future work

* Adding bring-your-own-buffer reading is a natural extension with the potential
  to improve performance.
* Customisable buffer sizes to allow developers to make the trade-off between
  throughput and response to backpressure explicitly.


## See also

* [WebSocketStream design
    notes](https://drive.google.com/a/chromium.org/open?id=1La1ehXw76HP6n1uUeks-WJGFgAnpX2tCjKts7QFJ57Y)
    for definition of the new API and technical details.
