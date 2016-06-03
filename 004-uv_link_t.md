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
`StreamBase` C++ class and auxiliary `StreamReq` and `StreamResources` classes.
This class is an analog of JavaScript streams in C++.
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
    implement protocols separately and chain them together in an easy and
    high-performant way using very narrow interfaces.

Given that [`uv_link_t`][0] depends only on [libuv][1], it is very easy to
envision how it will be used in Node.js addons. Even core modules could
be reimplemented on a user level without a need to patch the core itself.

Additionally, with the `uv_link_observer_t` (which is a part of [`uv_link_t`][0]
distribution), all of the reads that right now happen internally within the
`StreamBase` will be **observable**. This means that JavaScript user code will
still be able to listen for `data` events on the `net.Socket` that was consumed
by some `uv_link_t` stream.

## 3.1. uv_link_t technical description

_(NOTE: chain is built from links)_

_(NOTE: many of these API methods have return values, check them!)_

First, a `uv_stream_t*` instance needs to be picked. It will act as a source
link in a chain:
```c
uv_stream_t* stream = ...;

uv_link_source_t source;

uv_link_source_init(uv_default_loop(), &source, stream);
```

A chain can be formed with `&source` and any other `uv_link_t` instance:
```c
uv_link_t link;

static uv_link_methods_t methods = {
  .read_start = read_start_impl,
  .read_stop = read_stop_impl,
  .write = write_impl,
  .try_write = try_write_impl,
  .shutdown = shutdown_impl,
  .close = close_impl,

  /* These will be used only when chaining two links together */

  .alloc_cb_override = alloc_cb_impl,
  .read_cb_override = read_cb_impl
};

uv_link_init(&link, &methods);

/* Just like in libuv */
link.alloc_cb = my_alloc_cb;
link.read_cb = my_read_cb;

/* Creating a chain here */
uv_link_chain(&source, &link);

uv_link_read_start(&link);
```

Now comes a funny part, any of these method implementations may hook up into
the parent link in a chain to perform their actions:

```c
static int shutdown_impl(uv_link_t* link,
                         uv_link_t* source,
                         uv_link_shutdown_cb cb,
                         void* arg) {
  fprintf(stderr, "this will be printed\n");
  return uv_link_propagate_shutdown(link->parent, source, cb, arg);
}
```

## 3.2. How things are going to be hooked up into the core

TCP sockets, IPC pipes, and possibly TTYs will be a base elements of every
chain, backed up by the `uv_link_source_t`.

HTTP, TLS, WebSockets(?) will be custom `uv_link_methods_t`, so that the links
could be created for them and chained together with the `uv_link_source_t`.

The chaining process will happen in JavaScript through the method that will
take `v8:External` from the both `uv_link_t` instances, and run `uv_link_chain`
on them.

User `uv_link_methods_t` implementations will work in pretty much the same way,
bringing consistency to the C++ streams implementation.

## 3.3. Observability

`uv_link_observer_t` may be inserted between any two `uv_link_t`s to observe
the incoming data that flows between them. This is a huge difference for TLS,
where right now it is not possible to read any raw incoming bytes.

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
