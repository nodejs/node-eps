| Title  | Implement HTTP/2            |
|--------|-----------------------------|
| Author | @jasnell                    |
| Status | DRAFT                       |
| Date   | 2016-08-11T10:00:00-07:00   |

## Description

While HTTP/2 may be the successor to HTTP/1, it is a very different type of
protocol with very different implementation requirements. Most significantly,
it moves to a binary framing model that allows multiple simultaneous request
and response flows to be inflight across a single connection (essentially
pipelining without the head-of-line blocking issue). The protocol also
introduces stateful header compression, prioritization groups, flow control,
push streams, and mandatory TLS for all practical purposes.

This proposal suggests implementing HTTP/2 within Core as a distinctly separate
implementation that the existing HTTP/1 implementation (in other words, adding
HTTP/2 without touching any part of the existing HTTP/1 implementation).

## Details

The implementation would be based on the excellent nghttp2 C/C++ implementation.
The nghttp2 library would be added as a new dependency in `/deps`. A new
`src/node_http2.cc` (and corresponding `src/node_http2.h`) would be added that
would serve as a bridge to the nghttp2 API. This class would export a
`process.binding('http2')` with internal objects that represent HTTP2
Session, HTTP2 Stream, and HTTP2 Header constructs.

The nghttp2 library does not do it's own I/O. All it handles is the processing
of the framing data, using callbacks to interface with the host application.
We would use existing mechanisms in Node.js to establish TCP and TLS
connections, then interface with the nghttp2 API for processing of the HTTP/2
specific data. This is not dissimilar to how the current HTTP/1 implementation
interfaces with the http_parser library.

Each TCP/TLS socket will have an associated `HTTP2Session`. The `HTTP2Session`
will be exported via the `process.binding('http2')` (that is, from
`src/node_http2.cc`). The `HTTP2Session` wraps the session state managed by
the nghttp2 library, which includes the state necessary for header compression,
settings, priority and flow-control. Fortunately, nghttp2 takes care of most
of these details on our behalf.

Each `HTTP2Session` will have `N` `HTTP2Stream` objects associated with it.
Each `HTTP2Stream` instance an associated identifier, prioritization, and
flow control window. Each `HTTP2Stream` also has a `Readable` side (for inbound
data) and a `Writable` side (for outbound data). Each of these sides has an
associated bucket of header key-value pairs as well as an optional bucket of
trailer key-value pairs.

HTTP-semantics are associated with each `HTTP2Stream` instance:

* On the server, the Request is associated with the `HTTP2Stream` `Readable`
  side; while the Response is associated with the `HTTP2Stream` `Writable` side.
* On the client, the Request is associated with the `HTTP2Stream` 'Writable'
  side; while the Response is associated with the `HTTP2Stream` `Readable` side.

## API

This proposal suggests three distinct API levels for the HTTP/2 implementation:

* Low-level Internal (via `process.binding('http2')`)
* Low-level Public
* High-level HTTP

The Low-level Internal API would consist of the internal API exposed by the
`process.binding('http2')`. This is essentially an API wrapper for the nghttp2
library that also includes a number of connection-management APIs to simplify
things and make the I/O handoff more efficient. This Low-level Internal API
would not be intended for public use and would not be supported as such.

The Low-level Public API would provide a public facing API that provides
direct access to the `HTTP2Session` and `HTTP2Stream`. These would, for
instance, allow a module developer to work directly with the binary framing
layer, with access to the flow control and prioritization primitives.

The High-level HTTP API would be an As-Close-As-Possible match to the existing
HTTP/1 API exposed by the `http` and `https` modules in core. This API would
be the most familiar to existing developers but would have the most constrained
access to HTTP/2 specific features. For instance, while it would be possible
to use push streams, access to prioritization and flow control primitives would
be hidden.

Rough example of Low-level Public API:
```js
const http2 = require('http2');

const session = http2.createSession(options);
session.on('stream', (session, stream) => {
  stream.on('header', (key, value) => {});
  stream.on('data', (chunk) => {});
  stream.on('end', () => {});
  //
  stream.setHeader(key, value);
  stream.write('some data\n');
  stream.end('all done.');
});
```

Rough example of High-level HTTP API:
```js
const http2 = require('http2');

const server = http2.createServer(options, (req, res) => {
  res.statusCode = 200;
  res.setHeader(key, value);
  res.write('some data\n');
  res.end('all done');
});
```

The difference between the Low-level Public and High-level HTTP APIs is similar
to that between using the existing HTTP API vs. writing directly to a socket,
with the exception that multiple `HTTP2Stream` instances share the same socket
connection under the covers.

The High-level HTTP API is essentially a convenient sugar on top of the
Low-level API that would provide as much parity as possible with the existing
HTTP module. Ecosystem developers such as expressjs, hapi.js, or others, could
choose to either extend the high-level API or build off the Low-level pieces to
provide an alternative API without being forced to monkey-patch into the
high-level API implementation.

## Issues

1. The HTTP/2 specification allows implementations to support both HTTP/1 and
   HTTP/2 in a manner that allows the protocol version to be negotiated per
   connection. To keep things simple, the Node.js implementation would require
   that a server is either HTTP/1 or HTTP/2, never both. The implementations
   would be distinctly separate from one another and could not each be listening
   to the same port (`require('http').createServer()` would *always* be HTTP/1,
   while `require('http2').createServer()` would *always* be HTTP/2).

2. While the HTTP/2 specification does not specifically require the use of TLS,
   most client implementations do. In the Node.js implementation, it will be
   possible to create a server instance that uses plain-text or TLS connections.

3. The HTTP/2 specification defines a fairly complex prioritization group model
   for streams. Initially, the Node.js implementation would expose only the
   bare minimum primitives for this mechanism via the Low-level public API.
   That is, it will be possible to set and retrieve the stream priority, but
   Node.js itself will not perform any specific prioritization of frame data.
   It will be up to users of the Low-level API to do the rest. Prioritization
   will not be exposed initially via the high-level HTTP API.

4. The HTTP/2 specification allows extensibility of binary frame types. The
   Node.js implementation will initially not provide for the use of extension
   frames via either the low-level or high level public APIs. This can be
   looked at eventually but it's just easier locking this down for now.

5. The HTTP/2 specification allows for push streams (server initiated streams
   with an *assumed* HTTP request). Both the low-level and high-level HTTP
   APIs will support the use of push streams on both the client and server
   side.
