| Title  | Make C++ Stream APIs public |
|--------|-----------------------------|
| Author | @indutny                    |
| Status | REJECTED                    |
| Date   | 2015-12-17 19:39:46         |

## Description

Currently we use so called `StreamBase` APIs through the core to implement
various performance-critical networking protocols with maximum efficiency.
It is evident that this performance improvement could be beneficial for
user-land C++ addons too.

I propose to make this APIs public, introducing C++ interface similar to the
following one (simplified):

## Interface

```C++
class Stream {
 public:
  //
  // Low-level part
  //

  // Perform protocol-specific shutdown (EOF)
  virtual int DoShutdown(ShutdownWrap* req_wrap) = 0;

  // Write data asynchronously
  virtual int DoWrite(WriteWrap* w,
                      uv_buf_t* bufs,
                      size_t count,
                      uv_stream_t* send_handle) = 0;

  // **(optional)**
  // Try to write provided data immediately without blocking, if not
  // possible to do - should return `0` if no data was written, should decrement
  // `count` and change pointers/length in `bufs` if some data was written.
  virtual int DoTryWrite(uv_buf_t** bufs, size_t* count);

  //
  // High-level part
  //

  // Cast to the deepest class. Just a stub, probably does not need to be
  // here.
  virtual void* Cast() = 0;

  // Return `true` or `false` depending on liveness of the underlying
  // resource. If Stream is not alive - all operations on it will result in
  // `EINVAL`
  virtual bool IsAlive() = 0;

  // Return `true` if stream is currently closing. Closed streams should return
  // `false` when calling `IsAlive`
  virtual bool IsClosing() = 0;

  // **optional**, if `true` - `send_handle` may be not `nullptr` in `DoWrite`
  virtual bool IsIPCPipe();

  // **optional**, return fd
  virtual int GetFD();

  // Start reading and emitting data
  virtual int ReadStart() = 0;

  // Stop reading and emitting data (immediately)
  virtual int ReadStop() = 0;

 protected:
  // One of these must be implemented
  virtual AsyncWrap* GetAsyncWrap();
  virtual v8::Local<v8::Object> GetObject();
};
```

## Public APIs

Public facing APIs have two sides: JS and C++.

### JS

```js
stream.fd; // stream's fd or -1
stream._externalStream;  // External pointer to the deepest C++ class
stream.readStart();
stream.readStop();
stream.shutdown(req);
stream.writev(req, [ ..., chunk, size, ... ]);
stream.writeBuffer(req, buffer);
stream.writeAsciiString(req, string);
stream.writeUtf8String(req, string);
stream.writeUcs2String(req, string);
stream.writeBinaryString(req, string);
```

### C++

All of the C++ interface methods may be called. Additionally following methods
are available:

```C++
class Stream {
 public:
  //
  // Low-level APIs
  //

  // Invoke `after_write_cb`
  inline void OnAfterWrite(WriteWrap* w);

  // Invoke `alloc_cb`
  inline void OnAlloc(size_t size, uv_buf_t* buf);

  // Invoke `read_cb`
  inline void OnRead(size_t nread,
                     const uv_buf_t* buf,
                     uv_handle_type pending = UV_UNKNOWN_HANDLE);

  // Override and get callbacks
  inline void set_after_write_cb(Callback<AfterWriteCb> c);
  inline void set_alloc_cb(Callback<AllocCb> c);
  inline void set_read_cb(Callback<ReadCb> c);
  inline Callback<AfterWriteCb> after_write_cb();
  inline Callback<AllocCb> alloc_cb();
  inline Callback<ReadCb> read_cb();

  //
  // High-level APIs
  //

  // Add JS API methods to the target `FunctionTemplate`
  // NOTE: `flags` control some optional methods like: `shutdown` and `writev`
  static inline void AddMethods(Environment* env,
                                v8::Local<v8::FunctionTemplate> target,
                                int flags = kFlagNone);

  // Emit data to the JavaScript listeners
  void EmitData(ssize_t nread,
                v8::Local<v8::Object> buf,
                v8::Local<v8::Object> handle);

  // Mark stream as consumed
  inline void Consume();

  // TODO(indutny): add this to node.js core
  inline void Unconsume();
};
```

## Putting things together

This APIs could be used for implementing high-performance protocols in a
following way:

Each TCP socket has a `_handle` property, which is actually a C++ `Stream`
instance. One may wrap this `_handle` into a custom C++ `Stream` instance, and
replace the `_handle` property with it. This stream could override `read`,
`alloc`, `after_write` callbacks to hook up the incoming data from the wrapped
stream.

Calling `DoWrite` and friends on the wrapped stream will result in performing
actual TCP writes.

It is easy to see that it could be wrapped this way several times, creating
levels of abstraction without leaving C++!
