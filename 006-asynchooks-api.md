| Title  | AsyncHook API |
|--------|---------------|
| Author | @trevnorris   |
| Status | ACCEPTED      |
| Date   | 2017-01-27    |

## Description

Since its initial introduction along side the `AsyncListener` API, the internal
class `AsyncWrap` has slowly evolved to ensure a generalized API that would
serve as a solid base for module authors who wished to add listeners to the
event loop's life cycle. Some of the use cases `AsyncWrap` has covered are long
stack traces, continuation local storage, profiling of asynchronous requests
and resource tracking. The public API is now exposed as `'async_hooks'`.

It must be clarified that the `AsyncHook` API is not meant to abstract away
Node's implementation details. Observing how operations are performed, not
simply how they appear to perform from the public API, is a key point in its
usability. For a real world example, a user was having difficulty figuring out
why their `http.get()` calls were taking so long. Because `AsyncHook` exposed
the `dns.lookup()` request to resolve the host being made by the `net` module
it was trivial to write a performance measurement that exposed the hidden
culprit. Which was that DNS resolution was taking much longer than expected.

A small amount of abstraction is necessary to accommodate the conceptual nature
of the API. For example, the `HTTPParser` is not destructed but instead placed
into an unused pool to be used later. Even so, at the time it is placed into
the pool the `destroy()` callback will run and then the id assigned to that
instance will be removed. If that resource is again requested then it will be
assigned a new id and will run `init()` again.


## Goals

The intent for the initial release is to provide the most minimal set of API
hooks that don't inhibit module authors from writing tools addressing anything
in this problem space. In order to remain minimal, all potential features for
initial release will first be judged on whether they can be achieved by the
existing public API. If so then such a feature won't be included.

If the feature cannot be done with the existing public API then discussion will
be had on whether requested functionality should be included in the initial
API. If so then the best course of action to include the capabilities of the
request will be discussed and performed. Meaning, the resulting API may not be
the same as initially requested, but the user will be able to achieve the same
end result.

Performance impact of `AsyncHook` should be zero if not being used, and near
zero while being used. The performance overhead of `AsyncHook` callbacks
supplied by the user should account for essentially all the performance
overhead.


## Terminology

Because of the potential ambiguity for those not familiar with the terms
"handle" and "request" they should be well defined.

Handles are a reference to a system resource. Some resources are a simple
identifier. For example, file system handles are represented by a file
descriptor. Other handles are represented by libuv as a platform abstracted
struct, e.g. `uv_tcp_t`. Each handle can be continually reused to access and
operate on the referenced resource.

Requests are short lived data structures created to accomplish one task. The
callback for a request should always and only ever fire one time. Which is when
the assigned task has either completed or encountered an error. Requests are
used by handles to perform these tasks. Such as accepting a new connection or
writing data to disk.

When both "handle" and "request" are being addressed it is simply called a
"resource".


## API

### Overview

Here is a quick overview of the entire API. All of this API is explained in
more detail further down.

```js
// Standard way of requiring a module. Snake case follows core module
// convention.
const async_hooks = require('async_hooks');

// Return the id of the current execution context. Useful for tracking state
// and retrieving the resource of the current trigger without needing to use an
// AsyncHook.
const cid = async_hooks.currentId();

// Return the id of the resource responsible for triggering the callback of the
// current execution scope to fire.
const tid = async_hooks.triggerId();

// Propagating the correct trigger id to newly created asynchronous resource is
// important. To make that easier triggerIdScope() will make sure all resources
// created during callback() have that trigger. This also tracks nested calls
// and will unwind properly.
async_hooks.triggerIdScope(triggerId, callback);

// Create a new instance of AsyncHook. All of these callbacks are optional.
const asyncHook = async_hooks.createHook({ init, before, after, destroy });

// Allow callbacks of this AsyncHook instance to fire. This is not an implicit
// action after running the constructor, and must be explicitly run to begin
// executing callbacks.
asyncHook.enable();

// Disable listening for all new asynchronous events.
asyncHook.disable();


// The following are the callbacks that can be passed to createHook().

// init() is called during object construction. The resource may not have
// completed construction when this callback runs. So all fields of the
// resource referenced by "id" may not have been populated.
function init(id, type, triggerId, resource) { }

// before() is called just before the resource's callback is called. It can be
// called 0-N times for handles (e.g. TCPWrap), and should be called exactly 1
// time for requests (e.g. FSReqWrap).
function before(id) { }

// after() is called just after the resource's callback has finished, and will
// always fire. In the case of an uncaught exception after() will fire after
// the 'uncaughtException' handler, or domain, has handled the exception.
function after(id) { }

// destroy() is called when an AsyncWrap instance is destroyed. In cases like
// HTTPParser where the resource is reused, or timers where the resource is
// only a JS object, destroy() will be triggered manually to fire
// asynchronously after the after() hook has completed.
function destroy(id) { }


// The following is the recommended embedder API.

// AsyncEvent() is meant to be extended. Instantiating a new AsyncEvent() also
// triggers init(). If triggerId is omitted then currentId() is used.
const asyncEvent = new AsyncEvent(type[, triggerId]);

// Call before() hooks.
asyncEvent.emitBefore();

// Call after() hooks. It is important that before/after calls are unwound
// in the same order they are called. Otherwise an unrecoverable exception
// will be made.
asyncEvent.emitAfter();

// Call destroy() hooks.
asyncEvent.emitDestroy();

// Return the unique id assigned to the AsyncEvent instance.
asyncEvent.asyncId();

// Return the trigger id for the AsyncEvent instance.
asyncEvent.triggerId();


// The following calls are specific to the embedder API. This API is considered
// more low-level than the AsyncEvent API, and the AsyncEvent API is
// recommended over using the following.

// Return new unique id for a constructing resource. This value must be
// manually tracked by the user for triggering async events.
const id = async_hooks.newId();

// Retrieve the triggerId for the constructed resource. This value must be
// manually tracked by the user for triggering async events.
async_hooks.initTriggerId(id);

// Set the trigger id for the next asynchronous resource that is created. The
// value is reset after it's been retrieved. This API is similar to
// triggerIdScope() except it's more brittle because if a constructor fails the
// then the internal field may not be reset.
async_hooks.setInitTriggerId(id);

// Call the init() callbacks. It is recommended that resource be passed. If it
// is not then "null" will be passed to the init() hook.
async_hooks.emitInit(id, type[, triggerId[, resource]]);

// Call the before() callbacks. The reason for requiring both arguments is
// explained in further detail below.
async_hooks.emitBefore(id, triggerId);

// Call the after() callbacks.
async_hooks.emitAfter(id);

// Call the destroy() callbacks.
async_hooks.emitDestroy(id);
```


### `async_hooks`

The object returned from `require('async_hooks')`.


#### `async_hooks.currentId()`

* Returns {Number}

Return the id of the current execution context. Useful to track when something
fires. For example:

```js
console.log(async_hooks.currentId());    // 1 - bootstrap
fs.open(path, (err, fd) => {
  console.log(async_hooks.currentId());  // 2 - open()
}):
```

It is important to note that the id returned has to do with execution timing.
Not causality (which is covered by `triggerId()`). For example:

```js
const server = net.createServer(function onconnection(conn) {
  // Returns the id of the server, not of the new connection. Because the
  // on connection callback runs in the execution scope of the server's
  // MakeCallback().
  async_hooks.currentId();

}).listen(port, function onlistening() {
  // Returns the id of a TickObject (i.e. process.nextTick()) because all
  // callbacks passed to .listen() are wrapped in a nextTick().
  async_hooks.currentId();
});
```


#### `async_hooks.triggerId()`

* Returns {Number}

Return the id of the resource responsible for calling the callback that is
currently being executed. For example:

```js
const server = net.createServer(conn => {
  // Though the resource that caused (or triggered) this callback to
  // be called was that of the new connection. Thus the return value
  // of triggerId() is the id of "conn".
  async_hooks.triggerId();

}).listen(port, () => {
  // Even though all callbacks passed to .listen() are wrapped in a nextTick()
  // the callback itself exists because the call to the server's .listen()
  // was made. So the return value would be the id of the server.
  async_hooks.triggerId();
});
```


#### `async_hooks.triggerIdScope(triggerId, callback)`

* `triggerId` {Number}
* `callback` {Function}
* Returns {Undefined}

All resources created during the execution of `callback` will be given
`triggerId`. Unless it was otherwise 1) passed in as an argument to
`AsyncEvent` 2) set via `setInitTriggerId()` or 3) a nested call to
`triggerIdScope()` is made.

Meant to be used in conjunction with the `AsyncEvent` API, and preferred over
`setInitTriggerId()` because it is more error proof.

Example using this to make sure the correct `triggerId` propagates to newly
created asynchronous resources:

```js
class MyThing extends AsyncEvent {
  constructor(foo, cb) {
    this.foo = foo;
    async_hooks.triggerIdScope(this.asyncId(), () => {
      process.nextTick(cb);
    });
  }
}
```


#### `async_hooks.createHook(callbacks)`

* `callbacks` {Object}
* Returns {AsyncHook}

`createHook()` returns an `AsyncHook` instance that contains information about
the callbacks that will fire during specific asynchronous events in the
lifetime of the event loop. The focal point of these calls centers around the
`AsyncWrap` C++ class. These callbacks will also be called to emulate the
lifetime of resources that do not fit this model. For example, `HTTPParser`
instances are recycled to improve performance. So the `destroy()` callback will
be called manually after a request is complete. Just before it's placed into
the unused resource pool.

All callbacks are optional. So, for example, if only resource cleanup needs to
be tracked then only the `destroy()` callback needs to be passed. The
specifics of all functions that can be passed to `callbacks` is in the section
`Hook Callbacks`.

**Error Handling**: If any `AsyncHook` callbacks throw the application will
print the stack trace and exit. The exit path does follow that of any uncaught
exception, except for the fact that it is not catchable by an uncaught
exception handler, so any `'exit'` callbacks will fire. Unless the application
is run with `--abort-on-uncaught-exception`. In which case a stack trace will
be printed and the application will exit, leaving a core file.

The reason for this error handling behavior is that these callbacks are running
at potentially volatile points in an object's lifetime. For example during
class construction and destruction. Because of this, it is deemed necessary to
bring down the process quickly as to prevent an unintentional abort in the
future. This is subject to change in the future if a comprehensive analysis is
performed to ensure an exception can follow the normal control flow without
unintentional side effects.


#### `asyncHook.enable()`

* Returns {AsyncHook} A reference to `asyncHook`.

Enable the callbacks for a given `AsyncHook` instance. When a hook is enabled
it is added to a global pool of hooks to execute. These hooks do not propagate
with a single asynchronous chain but will be completely disabled when
`disable()` is called.

The reason that `enable()`/`disable()` only work on a global scale, instead of
allowing every asynchronous branch to track their own as was done in the
previous implementation, is because it is prohibitively expensive to track
all hook instances for every asynchronous execution chain.

Callbacks are not implicitly enabled after an instance is created. One goal of
the API is to require explicit action from the user. Though to help simplify
using `AsyncHook`, `enable()` returns the `asyncHook` instance so the call can
be chained. For example:

```js
const async_hooks = require('async_hooks');

const hook = async_hooks.createHook(callbacks).enable();
```


#### `asyncHook.disable()`

* Returns {AsyncHook} A reference to `asyncHook`.

Disable the callbacks for a given `AsyncHook` instance from the global pool of
hooks to be executed. Once a hook has been disabled it will not fire again
until enabled.

For API consistency `disable()` also returns the `AsyncHook` instance.


### Hook Callbacks

Key events in the lifetime of asynchronous events have been categorized into
four areas. On instantiation, before/after the callback is called and when the
instance is destructed. For cases where resources are reused, instantiation and
destructor calls are emulated.


#### `init(id, type, triggerId, resource)`

* `id` {Number}
* `type` {String}
* `triggerId` {Number}
* `resource` {Object}

Called when a class is constructed that has the _possibility_ to trigger an
asynchronous event. This _does not_ mean the instance must trigger
`before()`/`after()` before `destroy()` is called. Only that the possibility
exists.

This behavior can be observed by doing something like opening a resource then
closing it before the resource can be used. The following snippet demonstrates
this.

```js
require('net').createServer().listen(function() { this.close() });
// OR
clearTimeout(setTimeout(() => {}, 10));
```

Every new resource is assigned a unique id. Since JS only supports IEEE 754
floating floating point numbers, the maximum assignable id is `2^53 - 1` (also
defined in `Number.MAX_SAFE_INTEGER`). At this size node can assign a new id
every 100 nanoseconds and not run out for over 28 years. So while another
method could be used to identify new resources that would truly never run out,
doing so is not deemed necessary at this time since. If an alternative is found
that has at least the same performance as using a `double` then the
consideration will be made.

The `type` is a String that represents the type of resource that caused
`init()` to fire. Generally it will be the name of the resource's constructor.
Some examples include `TCP`, `GetAddrInfo` and `HTTPParser`. Users will be able
to define their own `type` when using the public embedder API.

**Note:** It's possible to have `type` name collisions. So embedders are
recommended to use a prefix of sorts to prevent this.

`triggerId` is the `id` of the resource that caused (or "triggered") the
new resource to initialize and that caused `init()` to fire.

The following is a simple demonstration of this:

```js
const async_hooks = require('async_hooks');

asyns_hooks.createHook({
  init: (id, type, triggerId) => {
    const cId = async_hooks.currentId();
    process._rawDebug(`${type}(${id}): trigger: ${triggerId} scope: ${cId}`);
  }
}).enable();

require('net').createServer(c => {}).listen(8080);
```

Output hitting the server with `nc localhost 8080`:

```
TCPWRAP(2): trigger: 1 scope: 1
TCPWRAP(4): trigger: 2 scope: 0
```

The second `TCPWRAP` is the new connection from the client. When a new
connection is made the `TCPWrap` instance is immediately constructed. This
happens outside of any JavaScript stack (side note: a `currentId()` of `0`
means it's being executed in "the void", with no JavaScript stack above it).
With only that information it would be impossible to link resources together in
terms of what caused them to be created. So `triggerId` is given the task of
propagating what resource is responsible for the new resource's existence.

Below is another example with additional information about the calls to
`init()` between the `before()` and `after()` calls. Specifically what the
callback to `listen()` will look like. The output formatting is slightly more
elaborate to make calling context easier to see.

```js
'use strict';
const async_hooks = require('async_hooks');

let ws = 0;
async_hooks.createHook({
  init: (id, type, triggerId) => {
    const cId = async_hooks.currentId();
    process._rawDebug(' '.repeat(ws) +
                      `${type}(${id}): trigger: ${triggerId} scope: ${cId}`);
  },
  before: (id) => {
    process._rawDebug(' '.repeat(ws) + 'before: ', id);
    ws += 2;
  },
  after: (id) => {
    ws -= 2;
    process._rawDebug(' '.repeat(ws) + 'after:  ', id);
  },
  destroy: (id) => {
    process._rawDebug(' '.repeat(ws) + 'destroy:', id);
  },
}).enable();

require('net').createServer(() => {}).listen(8080, () => {
  // Let's wait 10ms before logging the server started.
  setTimeout(() => {
    console.log('>>>', async_hooks.currentId());
  }, 10);
});
```

Output from only starting the server:

```
TCPWRAP(2): trigger: 1 scope: 1
TickObject(3): trigger: 2 scope: 1
before:  3
  Timeout(4): trigger: 3 scope: 3
  TIMERWRAP(5): trigger: 3 scope: 3
after:   3
destroy: 3
before:  5
  before:  4
    TTYWRAP(6): trigger: 4 scope: 4
    SIGNALWRAP(7): trigger: 4 scope: 4
    TTYWRAP(8): trigger: 4 scope: 4
>>> 4
    TickObject(9): trigger: 4 scope: 4
  after:   4
  destroy: 4
after:   5
before:  9
after:   9
destroy: 9
destroy: 5
```

First notice that `scope` and the value returned by `currentId()` are always
the same. That's because `currentId()` simply returns the value of the
current execution context; which is defined by `before()` and `after()` calls.

Now if we only use `scope` to graph resource allocation we get the following:

```
TTYWRAP(6) -> Timeout(4) -> TIMERWRAP(5) -> TickObject(3) -> root(1)
```

No where here do we see the `TCPWRAP` created; which was the reason for
`console.log()` being called. This is because binding to a port without a
hostname is actually synchronous, but to maintain a completely asynchronous API
the user's callback is placed in a `process.nextTick()`.

The graph only shows **when** a resource was created. Not **why**. So to track
the **why** use `triggerId`.


#### `before(id)`

* `id` {Number}

When an asynchronous operation is triggered (such as a TCP server receiving a
new connection) or completes (such as writing data to disk) a callback is
called to notify node. The `before()` callback is called just before said
callback is executed. `id` is the unique identifier assigned to the
resource about to execute the callback.

The `before()` callback will be called 0-N times if the resource is a handle,
and exactly 1 time if the resource is a request.


#### `after(id)`

* `id` {Number}

Called immediately after the callback specified in `before()` is completed. If
an uncaught exception occurs during execution of the callback then `after()`
will run after `'uncaughtException'` or a `domain`'s handler runs.


#### `destroy(id)`

* `id` {Number}

Called either when the class destructor is run or if the resource is manually
marked as free. For core C++ classes that have a destructor the callback will
fire during deconstruction. It is also called synchronously from the embedder
API `emitDestroy()`.

Some resources, such as `HTTPParser`, are not actually destructed but instead
placed in an unused resource pool to be used later. For these `destroy()` will
be called just before the resource is placed on the unused resource pool.

**Note:** Some resources depend on GC for cleanup. So if a reference is made to
the `resource` object passed to `init()` it's possible that `destroy()` is
never called. Causing a memory leak in the application. Of course if you know
the resource doesn't depend on GC then this isn't an issue.


## Embedder API

Library developers that handle their own I/O will need to hook into the
`AsyncWrap` API so that all the appropriate callbacks are called. To
accommodate this both a C++ and JS API is provided.


### `class AsyncEvent()`


#### `AsyncEvent(type[, triggerId])`

* Returns {AsyncEvent}

The class `AsyncEvent` was designed to be extended from for embedder's async
resources. Using this users can easily trigger the lifetime events of their
own resources.

The `init()` hook will trigger when `AsyncEvent` is instantiated.

Example usage:

```js
class DBQuery extends AsyncEvent {
  construtor(db) {
    this.db = db;
  }

  getInfo(query, callback) {
    this.db.get(query, (err, data) => {
      this.emitBefore();
      callback(err, data)
      this.emitAfter();
    });
  }

  close() {
    this.db = null;
    this.emitDestroy();
  }
}
```


#### `asyncEvent.emitBefore()`

* Returns {Undefined}

Call all `before()` hooks and let them know a new asynchronous execution
context is being entered. If nested calls to `emitBefore()` are made the stack
of `id`s will be tracked and properly unwound.


#### `asyncEvent.emitAfter()`

* Returns {Undefined}

Call all `after()` hooks. If nested calls to `emitBefore()` were made then make
sure the stack is unwound properly. Otherwise an error will be thrown.

If the user's callback thrown an exception then `emitAfter()` will
automatically be called for all `id`'s on the stack if the error is handled by
a domain or `'uncaughtException'` handler. So there is no need to guard against
this.


#### `asyncEvent.emitDestroy()`

* Returns {Undefined}

Call all `destroy()` hooks. This should only ever be called once. An error will
be thrown if it is. This **must** be manually called. If the resource is left
to be collected by the GC then the `destroy()` hooks will never be called.


#### `asyncEvent.asyncId()`

* Returns {Number}

Return the unique identifier assigned to the resource. Useful when used with
`triggerIdScope()`.


#### `asyncEvent.triggerId()`

* Returns {Number}

Return the same `triggerId` that is passed to `init()` hooks.


### Standalone JS API

The following API can be used as an alternative to using `AsyncEvent()`, but it
is left to the embedder to manually track values needed for all emitted events
(e.g. `id` and `triggerId`). It is very recommended that embedders instead
use `AsyncEvent`.


#### `async_hooks.newId()`

* Returns {Number}

Return a new unique id meant for a newly created asynchronous resource. The
value returned will never be assigned to another resource.

Generally this should be used during object construction. e.g.:

```js
class MyClass {
  constructor() {
    this._id = async_hooks.newId();
    this._triggerId = async_hooks.initTriggerId();
  }
}
```


#### `async_hooks.initTriggerId()`

* Returns {Number}

There are several ways to set the `triggerId` for an instantiated resource.
This API is how that value is retrieved. It returns the `id` of the resource
responsible for the newly created resource being instantiated. For example:


#### `async_hooks.emitInit(id, type[, triggerId][, resource])`

* `id` {Number} Generated by calling `newId()`
* `type` {String}
* `triggerId` {Number} **Default:** `currentId()`
* `resource` {Object} **Default:** `null`
* Returns {Undefined}

Emit that a resource is being initialized. `id` should be a value returned by
`async_hooks.newId()`. Usage will probably be as follows:

```js
class Foo {
  constructor() {
    this._id = async_hooks.newId();
    this._triggerId = async_hooks.initTriggerId();
    async_hooks.emitInit(this._id, 'Foo', this._triggerId, this);
  }
}
```

In the circumstance that the embedder needs to define a different trigger id
than `currentId()`, they can pass in that id manually.

It is suggested to have `emitInit()` be the last call in the object's
constructor.


#### `async_hooks.emitBefore(id[, triggerId])`

* `id` {Number} Generated by `newId()`
* `triggerId` {Number}
* Returns {Undefined}

Notify `before()` hooks the resource is about to enter its execution call
stack. If the `triggerId` of the resource is different from `id` then pass
it in.

Example usage:

```js
MyThing.prototype.done = function done() {
  // First call the before() hooks. So currentId() shows the id of the
  // resource wrapping the id that's been passed.
  async_hooks.emitBefore(this._id);

  // Run the callback.
  this.callback();

  // Call after() callbacks now that the old id has been restored.
  async_hooks.emitAfter(this._id);
};
```


#### `async_hooks.emitAfter(id)`

* `id` {Number} Generated by `newId()`
* Returns {Undefined}

Notify `after()` hooks the resource is exiting its execution call stack.

Even though the state of `id` is tracked internally, passing it in is required
as a way to validate that the stack is unwinding properly.

For example, no two async stack should cross when `emitAfter()` is called.

```
init # Foo
init # Bar
...
before # Foo
before # Bar
after # Foo   <- Should be called after Bar
after # Bar
```

**Note:** It is not necessary to wrap the callback in a `try`/`finally` and
force `emitAfter()` if the callback throws. That is automatically handled by
the fatal exception handler.


#### `async_hooks.emitDestroy(id)`

* `id` {Number} Generated by `newId()`
* Returns {Undefined}

Notify hooks that a resource is being destroyed (or being moved to the free'd
resource pool).


### Native API

**Note:** The native API is not yet implemented in the initial PR associated
with this EP, but will come soon enough afterward.

```cpp
// Helper class users can inherit from, but is not necessary. If
// `AsyncHook::MakeCallback()` is used then all four callbacks will be
// called automatically.
class AsyncHook {
  public:
    AsyncHook(v8::Local<v8::Object> resource, const char* name);
    ~AsyncHook();
    v8::MaybeLocal<v8::Value> MakeCallback(
        const v8::Local<v8::Function> callback,
        int argc,
        v8::Local<v8::Value>* argv);
    v8::Local<v8::Object> get_resource();
    double get_uid();
  private:
    AsyncHook();
    v8::Persistent<v8::Object> resource_;
    const char* name_;
    double uid_;
}

// Returns the id of the current execution context. If the return value is
// zero then no execution has been set. This will happen if the user handles
// I/O from native code.
double node::GetCurrentId();

// Return same value as async_hooks.triggerId();
double node::GetTriggerId();

// If the native API doesn't inherit from the helper class then the callbacks
// must be triggered manually. This triggers the init() callback. The return
// value is the uid assigned to the resource.
// TODO(trevnorris): This always needs to be called so that the resource can be
// placed on the Map for future query.
double node::EmitAsyncInit(v8::Local<v8::Object> resource);

// Emit the destroy() callback.
void node::EmitAsyncDestroy(double id);

// An API specific to emit before/after callbacks is unnecessary because
// MakeCallback will automatically call them for you.
// TODO(trevnorris): There should be a final optional parameter of
// "double triggerId" that's needed so async_hooks.triggerId() returns the
// correct value.
v8::MaybeLocal<v8::Value> node::MakeCallback(v8::Isolate* isolate,
                                             v8::Local<v8::Object> recv,
                                             v8::Local<v8::Function> callback,
                                             int argc,
                                             v8::Local<v8::Value>* argv,
                                             double id);
```


## API Exceptions

### Reused Resources

Resources like `HTTPParser` are reused throughout the lifetime of the
application. This means node will have to synthesize the `init()` and
`destroy()` calls. Also the id on the class instance will need to be changed
every time the resource is acquired for use.

Though for a shared resource like `TimerWrap` we're not concerned with
reassigning the id because it isn't directly used by the user. Instead it is
used internally for JS created resources that are all assigned their own unique
id, and each of these JS resources are linked to the `TimerWrap` instance.


## Notes

### Native Modules

Existing native modules that call `node::MakeCallback` today will always have
their trigger id as `1`. Which is the root id. There will be two native APIs
that users can use. One will be a static API and another will be a class that
users can inherit from.


### Promises

Recently V8 has implemented an API so that async hooks can properly interface
with Promises. Unfortunately this API cannot be backported, so if this API is
backported then there will be an API gap in which native Promises will break
the async stack.


### Immediate Write Without Request

When data is written through `StreamWrap` node first attempts to write as much
to the kernel as possible. If all the data can be flushed to the kernel then
the function exits without creating a `WriteWrap` and calls the user's
callback in `nextTick()` (which will only be called if the user passes a
callback). Meaning detection of the write won't be as straightforward as
watching for a `WriteWrap`.
