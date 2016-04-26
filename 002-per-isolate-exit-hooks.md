| Title  | Per-isolate exit hooks      |
|--------|-----------------------------|
| Author | @dherman                    |
| Status | DRAFT                       |
| Date   | 2016-02-12                  |

## Description

The public C++ API has the [`AtExit`](https://nodejs.org/api/addons.html#addons_void_atexit_callback_args) API for specifying teardown logic for a Node process, but with the upcoming addition of workers, there will be a need for custom teardown logic for arbitrary isolates.

This is particularly useful for addons that want to lazily allocate per-isolate data on a by-need basis, and therefore do not control the actual creation of the workers. Addons like that need the ability to deallocate any associated data structures when the isolate goes away.

For example, an addon can create a tagged native class with [`v8::FunctionTemplate`](https://v8docs.nodesource.com/node-5.0/d8/d83/classv8_1_1_function_template.html) that's associated with some additional class metadata. (This is in fact my situation for [https://github.com/rustbridge/neon](Neon).) Any workers that use the class will cause it to lazily create the class, its function template, and its metadata. In order to avoid a memory leak, the code that lazily allocates the metadata will also want to register a callback to deallocate that metadata when the worker terminates.

## API

**enum IsolateKind { Main, Worker }**
  * **`Main`**: this isolate belongs to the main Node process.
  * **`Worker`**: this isolate belongs to a worker.

**void AtIsolateExit(isolate, callback, args)**
  * **`isolate`**: `v8::Isolate *` - A pointer to the isolate to hook into.
  * **`callback`**: `void (*)(v8::Isolate *, void *, IsolateKind)` - A pointer to a function to call when the isolate is exited.
  * **`args`**: `void *` - A pointer to pass to the callback as the second argument.

Registers exit hooks that run after the event loop of an isolate has ended.

AtIsolateExit takes three parameters: a pointer to the isolate to hook into, a pointer to the callback function to run at exit, and a pointer to untyped context data to be passed to that callback.

Callbacks are run in last-in first-out order.

Callbacks are passed three arguments: the isolate being exited, the context argument that was passed in, and a flag indicating whether the isolate is the Node process's main isolate or the isolate for a worker.

Isolate exit handlers for the main isolate are called before `AtExit` handlers are called. (Intuitively: the isolate is torn down before the process is torn down.)

## Rollout

Initially, `AtIsolateExit` would simply provide equivalent functionality to `AtExit`, with callbacks being called with the main isolate and an `IsolateKind::Main` flag.

In order for the [workers API](https://github.com/nodejs/node/pull/2133) lands, it should provide per-worker callback queues for `AtIsolateExit`, and each worker's teardown logic should invoke the callbacks in its queue.

## Alternatives

I'm not particularly attached to the name, but I tried to pick something consistent with the `AtExit` name.

The isolate doesn't _have_ to be passed to the callback, but it seems like a nice convenience so that it doesn't have to be added to the context data structure.

The isolate kind could just be a boolean `is_main` flag, but I tried to avoid [the Boolean trap](http://ariya.ofilabs.com/2011/08/hall-of-api-shame-boolean-trap.html) of API design.

We could try to raise the level of abstraction to `node::Environment`, in order to avoid the V8-specific API. I understand there are people interested in revamping the Node C++ architecture away from V8 specifics. I think that's a great idea, but given how pervasive isolates are in the existing API, it would be hard (or maybe impossible) to make the incremental change just here. I think the more straightforward path is to add this small API now and orthogonally allow people to investigate the overall strategy for migrating Node and native addons towards a more abstract API layer.

## License

This document is provided under the MIT license.
