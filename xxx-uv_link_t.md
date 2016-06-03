| Title  | Port core to uv_link_t      |
|--------|-----------------------------|
| Author | @indutny                    |
| Status | DRAFT                       |
| Date   | 2016-06-03                  |

## 1. Proposal

I propose to replace `StreamBase` with [`uv_link_t`][0], and port existing
classes that inherit from `StreamBase`.

## 2. Current state of C++ Streams in Node

The fast HTTP and TLS protocol implementation in Node core depends on so called
`StreamBase` C++ class. This class is an analog of JavaScript streams in C++.
The main ideas behind `StreamBase` are to:

1. Avoid unnecessary memory allocations by reading data directly into buffer
   from which it will be synchronously parsed
2. Avoid calling JavaScript unless absolutely necessary

However, this API has lots of dependencies on the core internals, and thus
can't be used outside of it.

## 3. uv_link_t

From [`uv_link_t`][0] readme:

    Chainable libuv streams.

    It is quite easy to write a TCP server/client in [libuv][1]. Writing HTTP
    server/client is a bit harder. Writing HTTP server/client on top of TLS
    server could be unwieldy.

    `uv_link_t` aims to solve complexity problem that quickly escalates once
    using multiple layers of protocols in [libuv][1] by providing a way to
    implement protocols separately and chain them together in easy and
    high-performant way using very narrow interfaces.

Given that [`uv_link_t`][0] depends only on [libuv][1], it is very easy to
envision how it will be used in Node.js addons. Even core modules will no longer
be required to be distributed with core, since they could be built without core
dependencies.

Additionally, with the `uv_link_observer_t` (which is a part of [`uv_link_t`][0]
distribution), all of the reads that right now happen internally within the
`StreamBase` will be **observable**. This means that JavaScript user code will
still be able to listen for `data` events on the `net.Socket` that was consumed
by some `uv_link_t` stream.

## 4. Proof-of-Concept

Several modules were created as a Proof-of-Concept [`uv_link_t`][0]
implementations (which possibly could be modified to fit into core):

* [`uv_ssl_t`][2]
* [`uv_http_t`][3]

They can be combined together quite trivially as demonstrated in
[file-shooter][4].

## 5. Further Thoughts

Another abstraction level could be added for multiplexing [`uv_link_t`][0]s to
provide a common C interface for http2 and http1.1. This will help us bring
a http2 implementation into the core while reusing as much as possible of the
existing code.

[0]: https://github.com/indutny/uv_link_t
[1]: https://github.com/libuv/libuv
[2]: https://github.com/indutny/uv_ssl_t
[3]: https://github.com/indutny/uv_http_t
[4]: https://github.com/indutny/file-shooter
